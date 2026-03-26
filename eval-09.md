# Write C++20 code to implement page cache

## Claude Prompt

Claude Model: **Sonnet 4.6**

> [!NOTE]
> Implement a cache for reading and writing pages from a file in C++20.
> Use the file path as the file identifier.
> Use the least recently used (LRU) replacement policy.
> A page is represented by the following C++ class:
> class Page {
>     std::array<char, 4096> buf;
>     uint32_t id;
> };



## Here's a walkthrough of every design decision:


### Data structures

| Structure | Role   |
|-----------|--------|
| `std::list<CacheEntry>` (LRU list) | Doubly-linked list with front = MRU, back = LRU. Splicing an iterator to the front is O(1) and never invalidates other iterators.|
| `std::unordered_map<PageKey, LruIterator>` (index) | O(1) lookup of any (file, page_id) pair directly to its list node, avoiding a linear scan. |
| `std::unordered_map<std::string, int>` (fd map) | One file descriptor per open file, opened lazily on first access. |


### LRU mechanics
Every access (read or write) calls `promote()`, which does a single `list::splice` to move the entry to the front — O(1). 
When the cache is full, `evict_lru()` removes the back element — also O(1).

### Dirty / write-back policy
`write_page()` marks entries **dirty**. A dirty page is only flushed to disk when it is evicted, when `flush() / flush_all()` is called explicitly, 
or when the PageCache destructor runs. This is the standard write-back (copy-back) policy: it minimises disk I/O by coalescing multiple writes to the same page.

### Thread safety
A single `std::mutex` guards all public methods via `std::scoped_lock`. 
This is straightforward and correct; 
if you need higher concurrency you can partition the cache into shards (one mutex per shard) using `page_id % N` as the shard key.

### Page layout on disk
Page id is stored at byte offset `id × 4096` in the file. 
Reads beyond EOF are silently zero-filled (new pages), and writes extend the file automatically via `pwrite`.

## Implementation

```cpp
#pragma once

#include <array>
#include <cstdint>
#include <cstring>
#include <fcntl.h>
#include <format>
#include <filesystem>
#include <list>
#include <mutex>
#include <stdexcept>
#include <string>
#include <unistd.h>
#include <unordered_map>

// ────────────────────────────────────────────────────────────────────────────
// Page
// ────────────────────────────────────────────────────────────────────────────

class Page {
public:
    static constexpr std::size_t SIZE = 4096;

    std::array<char, SIZE> buf{};
    uint32_t               id{0};
};

// ────────────────────────────────────────────────────────────────────────────
// PageCache
//
//  • Capacity  : maximum number of pages held across ALL open files.
//  • Eviction  : least-recently-used page is evicted when the cache is full.
//  • Thread safety : a single mutex guards every public operation so the cache
//                    is safe to share among threads.
//  • Dirty tracking: pages modified via write_page() are marked dirty and are
//                    written back to disk before eviction (write-back policy).
// ────────────────────────────────────────────────────────────────────────────

class PageCache {
public:
    // ── Construction / destruction ──────────────────────────────────────────

    explicit PageCache(std::size_t capacity = 64)
        : capacity_(capacity)
    {
        if (capacity == 0)
            throw std::invalid_argument("PageCache capacity must be > 0");
    }

    ~PageCache() {
        // Best-effort flush of all dirty pages on destruction.
        try { flush_all(); } catch (...) {}
        // Close every file descriptor we still hold open.
        for (auto& [path, fd] : fd_map_)
            ::close(fd);
    }

    // Non-copyable, movable.
    PageCache(const PageCache&)            = delete;
    PageCache& operator=(const PageCache&) = delete;
    PageCache(PageCache&&)                 = default;
    PageCache& operator=(PageCache&&)      = default;

    // ── Public API ──────────────────────────────────────────────────────────

    /**
     * Read the page with the given id from the file at `path`.
     *
     * If the page is already cached it is returned immediately (cache hit).
     * Otherwise it is read from disk, inserted into the cache (potentially
     * evicting the LRU page), and then returned (cache miss).
     *
     * Returns a const reference that remains valid until the next mutating
     * cache operation on the same slot.  Copy the Page if you need to keep it
     * across calls.
     */
    const Page& read_page(const std::filesystem::path& path, uint32_t page_id)
    {
        std::scoped_lock lock(mutex_);
        return *fetch(path, page_id, /*mark_dirty=*/false);
    }

    /**
     * Write `data` into the page with the given id in the file at `path`.
     *
     * The page is brought into the cache (cache miss → load from disk first,
     * or a brand-new page if it does not exist yet), updated in-place, and
     * marked dirty.  The dirty page is written back to disk either when it is
     * evicted or when flush() / flush_all() is called explicitly.
     */
    void write_page(const std::filesystem::path& path,
                    uint32_t                      page_id,
                    const Page&                   data)
    {
        std::scoped_lock lock(mutex_);
        Page* p = fetch(path, page_id, /*mark_dirty=*/true);
        *p = data;
        p->id = page_id;          // ensure id is consistent
    }

    /**
     * Flush all dirty pages that belong to the file at `path` to disk.
     */
    void flush(const std::filesystem::path& path)
    {
        std::scoped_lock lock(mutex_);
        flush_file(path.string());
    }

    /**
     * Flush every dirty page in the cache to disk.
     */
    void flush_all()
    {
        std::scoped_lock lock(mutex_);
        for (auto& entry : lru_list_)
            if (entry.dirty)
                write_back(entry);
    }

    /**
     * Evict all pages belonging to `path` and close its file descriptor.
     * Dirty pages are written back before eviction.
     */
    void close_file(const std::filesystem::path& path)
    {
        std::scoped_lock lock(mutex_);
        const std::string key = path.string();
        flush_file(key);
        evict_file(key);
        if (auto it = fd_map_.find(key); it != fd_map_.end()) {
            ::close(it->second);
            fd_map_.erase(it);
        }
    }

    // ── Diagnostics ─────────────────────────────────────────────────────────

    std::size_t size()     const noexcept { std::scoped_lock l(mutex_); return lru_list_.size(); }
    std::size_t capacity() const noexcept { return capacity_; }

    struct Stats { std::size_t hits{0}; std::size_t misses{0}; };
    Stats stats() const noexcept { std::scoped_lock l(mutex_); return stats_; }

private:
    // ── Internal types ───────────────────────────────────────────────────────

    // Each entry in the LRU list carries the page data plus bookkeeping.
    struct CacheEntry {
        Page        page{};
        std::string file_key{};   // canonical file path
        bool        dirty{false};
    };

    // The LRU list: front = most recently used, back = least recently used.
    using LruList     = std::list<CacheEntry>;
    using LruIterator = LruList::iterator;

    // Index: (file_key, page_id) → iterator into the LRU list.
    struct PageKey {
        std::string file_key;
        uint32_t    page_id;
        bool operator==(const PageKey&) const noexcept = default;
    };
    struct PageKeyHash {
        std::size_t operator()(const PageKey& k) const noexcept {
            std::size_t h1 = std::hash<std::string>{}(k.file_key);
            std::size_t h2 = std::hash<uint32_t>{}(k.page_id);
            // Combine with a golden-ratio mix.
            return h1 ^ (h2 * 0x9e3779b9u + 0x6c62272eu + (h1 << 6) + (h1 >> 2));
        }
    };

    // ── Data members ─────────────────────────────────────────────────────────

    const std::size_t                                       capacity_;
    LruList                                                 lru_list_;
    std::unordered_map<PageKey, LruIterator, PageKeyHash>   index_;
    std::unordered_map<std::string, int>                    fd_map_;   // path → fd
    mutable std::mutex                                      mutex_;
    Stats                                                   stats_{};

    // ── Internal helpers (all called with mutex_ already held) ───────────────

    /**
     * Core fetch routine.  Returns a pointer to the (possibly just-loaded)
     * page.  If mark_dirty is true the entry is flagged for write-back.
     */
    Page* fetch(const std::filesystem::path& path, uint32_t page_id, bool mark_dirty)
    {
        const std::string key = path.string();
        const PageKey     pk{key, page_id};

        if (auto it = index_.find(pk); it != index_.end()) {
            // ── Cache hit ───────────────────────────────────────────────────
            ++stats_.hits;
            promote(it->second);           // move to front (most recently used)
            if (mark_dirty)
                it->second->dirty = true;
            return &it->second->page;
        }

        // ── Cache miss ──────────────────────────────────────────────────────
        ++stats_.misses;
        if (lru_list_.size() >= capacity_)
            evict_lru();

        // Construct the new entry at the front of the list.
        lru_list_.push_front(CacheEntry{.file_key = key, .dirty = mark_dirty});
        LruIterator entry_it = lru_list_.begin();
        index_.emplace(pk, entry_it);

        Page& p   = entry_it->page;
        p.id      = page_id;

        // Try to read existing page data from disk; it's fine if the page
        // doesn't exist yet (new file or new page beyond EOF).
        int fd = open_fd(key);
        load_page(fd, p);

        return &p;
    }

    /**
     * Move an iterator to the front of the LRU list without invalidating it.
     */
    void promote(LruIterator it) {
        lru_list_.splice(lru_list_.begin(), lru_list_, it);
    }

    /**
     * Evict the least-recently-used entry (back of the list).
     */
    void evict_lru() {
        CacheEntry& victim = lru_list_.back();
        if (victim.dirty)
            write_back(victim);
        index_.erase(PageKey{victim.file_key, victim.page.id});
        lru_list_.pop_back();
    }

    /**
     * Write a dirty entry back to disk.
     */
    void write_back(CacheEntry& entry) {
        int fd = open_fd(entry.file_key);
        store_page(fd, entry.page);
        entry.dirty = false;
    }

    /**
     * Write back all dirty pages for a specific file.
     */
    void flush_file(const std::string& key) {
        for (auto& entry : lru_list_)
            if (entry.file_key == key && entry.dirty)
                write_back(entry);
    }

    /**
     * Remove every cache entry that belongs to `key` (without writing back).
     * Call flush_file() first if you need dirty pages persisted.
     */
    void evict_file(const std::string& key) {
        for (auto list_it = lru_list_.begin(); list_it != lru_list_.end(); ) {
            if (list_it->file_key == key) {
                index_.erase(PageKey{key, list_it->page.id});
                list_it = lru_list_.erase(list_it);
            } else {
                ++list_it;
            }
        }
    }

    // ── File I/O helpers ─────────────────────────────────────────────────────

    /**
     * Return (and lazily open) the file descriptor for `key`.
     */
    int open_fd(const std::string& key) {
        if (auto it = fd_map_.find(key); it != fd_map_.end())
            return it->second;

        // O_CREAT | O_RDWR so we can both read existing pages and write new ones.
        int fd = ::open(key.c_str(), O_RDWR | O_CREAT, 0644);
        if (fd < 0)
            throw std::runtime_error(std::format("PageCache: cannot open '{}': {}",
                                                 key, std::strerror(errno)));
        fd_map_.emplace(key, fd);
        return fd;
    }

    /**
     * Read one page from disk into `page`.  If the file is too short (new
     * page) the buffer is zero-filled instead of raising an error.
     */
    static void load_page(int fd, Page& page) {
        const off_t offset = static_cast<off_t>(page.id) * Page::SIZE;
        const ssize_t n = ::pread(fd, page.buf.data(), Page::SIZE, offset);
        if (n < 0)
            throw std::runtime_error(std::format("PageCache: pread failed: {}",
                                                 std::strerror(errno)));
        // Partial read (new page beyond EOF) → zero-fill the remainder.
        if (static_cast<std::size_t>(n) < Page::SIZE)
            std::memset(page.buf.data() + n, 0, Page::SIZE - static_cast<std::size_t>(n));
    }

    /**
     * Write one page from `page` to disk.
     */
    static void store_page(int fd, const Page& page) {
        const off_t   offset = static_cast<off_t>(page.id) * Page::SIZE;
        const ssize_t n      = ::pwrite(fd, page.buf.data(), Page::SIZE, offset);
        if (n < 0 || static_cast<std::size_t>(n) != Page::SIZE)
            throw std::runtime_error(std::format("PageCache: pwrite failed: {}",
                                                 std::strerror(errno)));
    }
};

```

## Elementary test

```cpp
#include <cassert>
#include <cstring>
#include <iostream>
#include "cache.hpp"

int main() {
    const std::filesystem::path file = "/tmp/test.db";
    std::filesystem::remove(file); // start fresh

    PageCache cache(4); // timy capacity to force evictions

    // Write pages
    for(uint32_t i = 0; i < 6; ++i) {
        Page p;
        p.id = i;
        std::memset(p.buf.data(), static_cast<int>('A' + i), Page::SIZE);
        cache.write_page(file, i, p);
    }
    // Flush to disk
    cache.flush(file);

    // Read back
    for(uint32_t i = 0; i < 6; ++i) {
        const Page& p = cache.read_page(file, i);
        assert(p.id == i);
        assert(static_cast<unsigned char>(p.buf[0]) == static_cast<unsigned char>('A' + i));
    }

    auto s = cache.stats();
    std::cout << "hits=" << s.hits
              << " misses=" << s.misses
              << " size=" << cache.size()
              << "/" << cache.capacity()
              << "\n";

    cache.close_file(file);
    std::filesystem::remove(file);

    return 0;
}

// g++ -std=c++20 -Wall -Wextra -o test_cache test_cache.cpp && test_cache
```
