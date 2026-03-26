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

## Extensive tests

### Claude Prompt

Claude Model: **Sonnet 4.6**

> [!NOTE]
> Generate extensive test cases to cover each use case. Use the Gtest framework to generate tests.


### Suite test description

**68 tests, 13 suites, all green**. 

Here's what each suite covers:

| Suite | Tests | What is verified |
|-------|-------|------------------|
| Construction | 5 | Default/custom capacity accepted, zero capacity throws, initial size and stats are zero |
| ReadWrite    | 7 | Fresh pages are zero-filled, write→read roundtrip, page ID preservation, independent pages, overwrite, page 0, high page ID (large file offset) | 
| Stats        | 6 | First access is a miss, repeated reads hit, write counts as miss then hit, accumulated counters, evicted page becomes a miss again | 
| LruEviction  | 7 | Capacity never exceeded, correct victim selection, `read` promotes to MRU, `write` promotes to MRU, capacity-1 always evicts, sequential scan, repeated access on same page | 
| WriteBack    | 8 | `flush()` writes to disk, `flush_all()` covers all files, dirty page is written on eviction, clean page is **not** written on eviction, destructor flushes, data survives cache recreation, last write wins, non-contiguous pages | | MultiFile    | 5 | Same page ID in different files are isolated, size counts across files, capacity is shared, `flush()` targets one file, many files coexist |
| CloseFile    | 5 | Evicts pages, flushes dirty first, targets only one file, reopen works, idempotent on unknown file |
| CacheSize    | 3 | Size tracks unique misses, never exceeds capacity, hits don't change size | 
| Flush        | 5 | No-op on empty cache, raw I/O confirms byte content, page stays in cache after flush |
| Persistence  | 2 | Cross-instance read, eviction-driven flush without explicit `flush()` call |
| PageContent  | 4 | Full 4096-byte buffer equality, first/last byte roundtrip, mixed byte pattern, new page beyond EOF is zero-filled |
| ThreadSafety | 4 | Concurrent reads, concurrent writes (disjoint pages), mixed read/write/flush, stats consistency under load | 
| EdgeCases    | 7 | Repeated overwrite of page 0, interleaved reads/writes, flush with no dirty pages, close→write→read, double close, 256 pages through a capacity-8 cache, multi-file capacity crossover |


### Generated test suite
```cpp
// page_cache_test.cpp
// Comprehensive GTest suite for PageCache / Page.
//
// Build (from the directory containing both files):
//
//   g++ -std=c++20 -Wall -Wextra -pthread \
//       page_cache_test.cpp \
//       -lgtest_main -lgtest -lpthread \
//       -o page_cache_test
//
//   ./page_cache_test          # run all tests
//   ./page_cache_test --gtest_filter='LruEviction*'   # run a suite

#include "cache.hpp"

#include <algorithm>
#include <atomic>
#include <chrono>
#include <cstring>
#include <filesystem>
#include <fstream>
#include <numeric>
#include <set>
#include <thread>
#include <vector>

#include <gtest/gtest.h>

namespace fs = std::filesystem;

// ────────────────────────────────────────────────────────────────────────────
// Test fixtures & helpers
// ────────────────────────────────────────────────────────────────────────────

// Returns a unique temp path that is removed automatically when the guard goes
// out of scope.
struct TempFile {
    fs::path path;
    explicit TempFile(const std::string& suffix = ".db") {
        path = fs::temp_directory_path() /
               ("pc_test_" + std::to_string(
                    std::chrono::steady_clock::now().time_since_epoch().count()) +
                suffix);
    }
    ~TempFile() { std::error_code ec; fs::remove(path, ec); }
    // Implicit conversion so it can be passed directly to PageCache calls.
    operator const fs::path&() const { return path; }
};

// Build a Page with every byte set to `fill` and the given id.
static Page make_page(uint32_t id, char fill) {
    Page p;
    p.id = id;
    p.buf.fill(fill);
    return p;
}

// Read the raw bytes of page `id` directly from disk (bypassing the cache).
static std::array<char, Page::SIZE> read_raw(const fs::path& file, uint32_t id) {
    std::array<char, Page::SIZE> buf{};
    int fd = ::open(file.c_str(), O_RDONLY);
    if (fd < 0) return buf;
    ::pread(fd, buf.data(), Page::SIZE, static_cast<off_t>(id) * Page::SIZE);
    ::close(fd);
    return buf;
}

// ────────────────────────────────────────────────────────────────────────────
// Suite 1 – Construction & capacity
// ────────────────────────────────────────────────────────────────────────────

TEST(Construction, DefaultCapacityIsAccepted) {
    EXPECT_NO_THROW({ PageCache c; });
}

TEST(Construction, CustomCapacityIsStored) {
    PageCache c(16);
    EXPECT_EQ(c.capacity(), 16u);
}

TEST(Construction, ZeroCapacityThrows) {
    EXPECT_THROW({ PageCache c(0); }, std::invalid_argument);
}

TEST(Construction, InitialSizeIsZero) {
    PageCache c(8);
    EXPECT_EQ(c.size(), 0u);
}

TEST(Construction, InitialStatsAreZero) {
    PageCache c(8);
    auto s = c.stats();
    EXPECT_EQ(s.hits,   0u);
    EXPECT_EQ(s.misses, 0u);
}

// ────────────────────────────────────────────────────────────────────────────
// Suite 2 – Basic read / write
// ────────────────────────────────────────────────────────────────────────────

TEST(ReadWrite, ReadFreshPageIsZeroFilled) {
    TempFile f;
    PageCache c(4);
    const Page& p = c.read_page(f, 0);
    EXPECT_EQ(p.id, 0u);
    for (char byte : p.buf)
        EXPECT_EQ(byte, '\0');
}

TEST(ReadWrite, WriteThenReadReturnsCorrectData) {
    TempFile f;
    PageCache c(4);
    c.write_page(f, 0, make_page(0, 'X'));
    const Page& p = c.read_page(f, 0);
    EXPECT_EQ(p.id, 0u);
    for (char byte : p.buf)
        EXPECT_EQ(byte, 'X');
}

TEST(ReadWrite, PageIdIsPreservedAfterWrite) {
    TempFile f;
    PageCache c(4);
    c.write_page(f, 7, make_page(7, 'A'));
    EXPECT_EQ(c.read_page(f, 7).id, 7u);
}

TEST(ReadWrite, MultipleDistinctPagesStoredIndependently) {
    TempFile f;
    PageCache c(8);
    for (uint32_t id = 0; id < 5; ++id)
        c.write_page(f, id, make_page(id, static_cast<char>('a' + id)));

    for (uint32_t id = 0; id < 5; ++id) {
        const Page& p = c.read_page(f, id);
        EXPECT_EQ(p.id, id);
        for (char byte : p.buf)
            EXPECT_EQ(byte, static_cast<char>('a' + id));
    }
}

TEST(ReadWrite, OverwritePageUpdatesData) {
    TempFile f;
    PageCache c(4);
    c.write_page(f, 0, make_page(0, 'A'));
    c.write_page(f, 0, make_page(0, 'B'));
    for (char byte : c.read_page(f, 0).buf)
        EXPECT_EQ(byte, 'B');
}

TEST(ReadWrite, PageZeroIdIsHandledCorrectly) {
    TempFile f;
    PageCache c(4);
    c.write_page(f, 0, make_page(0, 'Z'));
    EXPECT_EQ(c.read_page(f, 0).buf[0], 'Z');
}

TEST(ReadWrite, HighPageIdIsHandledCorrectly) {
    TempFile f;
    PageCache c(4);
    constexpr uint32_t HIGH_ID = 1000;
    c.write_page(f, HIGH_ID, make_page(HIGH_ID, 'H'));
    c.flush(f);
    // Evict then re-read from disk.
    c.close_file(f);
    PageCache c2(4);
    const Page& p = c2.read_page(f, HIGH_ID);
    EXPECT_EQ(p.id, HIGH_ID);
    EXPECT_EQ(p.buf[0], 'H');
}

// ────────────────────────────────────────────────────────────────────────────
// Suite 3 – Hit / miss statistics
// ────────────────────────────────────────────────────────────────────────────

TEST(Stats, FirstReadIsMiss) {
    TempFile f;
    PageCache c(4);
    c.read_page(f, 0);
    EXPECT_EQ(c.stats().misses, 1u);
    EXPECT_EQ(c.stats().hits,   0u);
}

TEST(Stats, SecondReadSamePageIsHit) {
    TempFile f;
    PageCache c(4);
    c.read_page(f, 0);
    c.read_page(f, 0);
    EXPECT_EQ(c.stats().misses, 1u);
    EXPECT_EQ(c.stats().hits,   1u);
}

TEST(Stats, WriteCountsAsMiss) {
    TempFile f;
    PageCache c(4);
    c.write_page(f, 0, make_page(0, 'W'));
    EXPECT_EQ(c.stats().misses, 1u);
}

TEST(Stats, WriteAfterReadIsHit) {
    TempFile f;
    PageCache c(4);
    c.read_page(f, 0);
    c.write_page(f, 0, make_page(0, 'W')); // page already in cache → hit
    EXPECT_EQ(c.stats().hits,   1u);
    EXPECT_EQ(c.stats().misses, 1u);
}

TEST(Stats, RepeatedReadsAccumulateHits) {
    TempFile f;
    PageCache c(4);
    c.read_page(f, 0);          // miss
    for (int i = 0; i < 10; ++i)
        c.read_page(f, 0);      // 10 hits
    EXPECT_EQ(c.stats().hits,   10u);
    EXPECT_EQ(c.stats().misses,  1u);
}

TEST(Stats, EvictedPageBecomesMissOnReread) {
    TempFile f;
    PageCache c(2);             // capacity 2
    c.read_page(f, 0);          // miss, cache=[0]
    c.read_page(f, 1);          // miss, cache=[1,0]
    c.read_page(f, 2);          // miss, evicts 0, cache=[2,1]
    c.read_page(f, 0);          // miss again (0 was evicted)
    EXPECT_EQ(c.stats().misses, 4u);
    EXPECT_EQ(c.stats().hits,   0u);
}

// ────────────────────────────────────────────────────────────────────────────
// Suite 4 – LRU eviction ordering
// ────────────────────────────────────────────────────────────────────────────

// Helper: fill a capacity-N cache with pages 0..N-1 (page 0 is LRU, N-1 MRU).
static void fill_cache(PageCache& c, const fs::path& f, std::size_t n) {
    for (uint32_t id = 0; id < static_cast<uint32_t>(n); ++id)
        c.read_page(f, id);
}

TEST(LruEviction, CacheNeverExceedsCapacity) {
    TempFile f;
    constexpr std::size_t CAP = 4;
    PageCache c(CAP);
    for (uint32_t id = 0; id < 20; ++id)
        c.read_page(f, id);
    EXPECT_EQ(c.size(), CAP);
}

TEST(LruEviction, LruPageIsEvictedFirst) {
    TempFile f;
    PageCache c(3);             // capacity 3
    fill_cache(c, f, 3);        // MRU→[2,1,0]←LRU
    // Page 0 is LRU — inserting page 3 must evict it, not 1 or 2.
    auto before = c.stats();
    c.read_page(f, 3);          // evicts 0
    // Re-read page 0 → must be a miss.
    c.read_page(f, 0);
    EXPECT_EQ(c.stats().misses, before.misses + 2u); // page 3 + page 0
}

TEST(LruEviction, AccessingPagePromotesItToMru) {
    TempFile f;
    PageCache c(3);
    fill_cache(c, f, 3);        // MRU→[2,1,0]←LRU
    c.read_page(f, 0);          // promote 0 → MRU→[0,2,1]←LRU
    c.read_page(f, 3);          // evicts 1 (new LRU)
    // Page 1 should now be a miss; page 0 and 2 should be hits.
    auto hits_before = c.stats().hits;
    c.read_page(f, 0);
    c.read_page(f, 2);
    EXPECT_EQ(c.stats().hits, hits_before + 2u);
    auto misses_before = c.stats().misses;
    c.read_page(f, 1);
    EXPECT_EQ(c.stats().misses, misses_before + 1u);
}

TEST(LruEviction, WriteAlsoPromotesPage) {
    TempFile f;
    PageCache c(3);
    fill_cache(c, f, 3);                         // MRU→[2,1,0]←LRU
    c.write_page(f, 0, make_page(0, 'W'));        // promote 0 → MRU→[0,2,1]←LRU
    c.read_page(f, 3);                            // evicts 1
    auto misses_before = c.stats().misses;
    c.read_page(f, 1);                            // miss  – was evicted
    EXPECT_EQ(c.stats().misses, misses_before + 1u);
    auto hits_before = c.stats().hits;
    c.read_page(f, 0);                            // hit  – was promoted
    EXPECT_EQ(c.stats().hits, hits_before + 1u);
}

TEST(LruEviction, CapacityOfOneAlwaysEvictsOnInsert) {
    TempFile f;
    PageCache c(1);
    for (uint32_t id = 0; id < 5; ++id) {
        c.read_page(f, id);
        EXPECT_EQ(c.size(), 1u);
    }
    EXPECT_EQ(c.stats().misses, 5u);
}

TEST(LruEviction, SequentialAccessPatternEvictsInOrder) {
    TempFile f;
    PageCache c(3);
    // Access order: 0,1,2,3,4,5 — classic sequential scan, always evicts LRU.
    for (uint32_t id = 0; id < 6; ++id)
        c.read_page(f, id);
    EXPECT_EQ(c.stats().misses, 6u);
    EXPECT_EQ(c.stats().hits,   0u);
}

TEST(LruEviction, RepeatedAccessOnSamePageNoExtraEvictions) {
    TempFile f;
    PageCache c(2);
    c.read_page(f, 0);
    for (int i = 0; i < 50; ++i) c.read_page(f, 0); // all hits
    EXPECT_EQ(c.size(), 1u);
    EXPECT_EQ(c.stats().hits,   50u);
    EXPECT_EQ(c.stats().misses,  1u);
}

// ────────────────────────────────────────────────────────────────────────────
// Suite 5 – Dirty / write-back persistence
// ────────────────────────────────────────────────────────────────────────────

TEST(WriteBack, FlushWritesDirtyPageToDisk) {
    TempFile f;
    {
        PageCache c(4);
        c.write_page(f, 0, make_page(0, 'P'));
        c.flush(f);
    }
    auto raw = read_raw(f, 0);
    EXPECT_EQ(raw[0], 'P');
    EXPECT_EQ(raw[Page::SIZE - 1], 'P');
}

TEST(WriteBack, FlushAllWritesAllDirtyPagesToDisk) {
    TempFile f1, f2;
    {
        PageCache c(8);
        c.write_page(f1, 0, make_page(0, 'A'));
        c.write_page(f2, 0, make_page(0, 'B'));
        c.flush_all();
    }
    EXPECT_EQ(read_raw(f1, 0)[0], 'A');
    EXPECT_EQ(read_raw(f2, 0)[0], 'B');
}

TEST(WriteBack, EvictionWritesDirtyPageToDisk) {
    TempFile f;
    PageCache c(1);             // capacity 1 → every new page evicts
    c.write_page(f, 0, make_page(0, 'Q'));
    c.read_page(f, 1);          // evicts page 0 (dirty → written to disk)
    auto raw = read_raw(f, 0);
    EXPECT_EQ(raw[0], 'Q');
}

TEST(WriteBack, CleanPageEvictionDoesNotWriteToDisk) {
    TempFile f;
    {
        PageCache c(1);
        c.read_page(f, 0);      // loads page 0 (clean, zero-filled)
        c.read_page(f, 1);      // evicts page 0 (clean → no disk write)
    }
    // File should still not exist (or be empty) because nothing was ever written.
    if (fs::exists(f.path)) {
        EXPECT_EQ(fs::file_size(f.path), 0u);
    }
}

TEST(WriteBack, DestructorFlushesDirtyPages) {
    TempFile f;
    {
        PageCache c(4);
        c.write_page(f, 2, make_page(2, 'D'));
        // Destructor is called here — must flush page 2.
    }
    auto raw = read_raw(f, 2);
    EXPECT_EQ(raw[0], 'D');
}

TEST(WriteBack, DataSurvivesCacheRecreation) {
    TempFile f;
    {
        PageCache c(4);
        c.write_page(f, 3, make_page(3, 'S'));
        c.flush(f);
    }
    PageCache c2(4);
    const Page& p = c2.read_page(f, 3);
    EXPECT_EQ(p.id, 3u);
    EXPECT_EQ(p.buf[0], 'S');
}

TEST(WriteBack, MultipleWritesToSamePageOnlyLastSurvives) {
    TempFile f;
    PageCache c(4);
    c.write_page(f, 0, make_page(0, 'X'));
    c.write_page(f, 0, make_page(0, 'Y'));
    c.write_page(f, 0, make_page(0, 'Z'));
    c.flush(f);
    EXPECT_EQ(read_raw(f, 0)[0], 'Z');
}

TEST(WriteBack, NonContiguousPagesPersistIndependently) {
    TempFile f;
    {
        PageCache c(8);
        c.write_page(f, 0,   make_page(0,   'A'));
        c.write_page(f, 5,   make_page(5,   'B'));
        c.write_page(f, 100, make_page(100, 'C'));
        c.flush_all();
    }
    EXPECT_EQ(read_raw(f, 0  )[0], 'A');
    EXPECT_EQ(read_raw(f, 5  )[0], 'B');
    EXPECT_EQ(read_raw(f, 100)[0], 'C');
}

// ────────────────────────────────────────────────────────────────────────────
// Suite 6 – Multi-file isolation
// ────────────────────────────────────────────────────────────────────────────

TEST(MultiFile, SamePageIdInDifferentFilesAreIndependent) {
    TempFile f1, f2;
    PageCache c(8);
    c.write_page(f1, 0, make_page(0, 'F'));
    c.write_page(f2, 0, make_page(0, 'G'));
    EXPECT_EQ(c.read_page(f1, 0).buf[0], 'F');
    EXPECT_EQ(c.read_page(f2, 0).buf[0], 'G');
}

TEST(MultiFile, CacheSizeCountsAcrossAllFiles) {
    TempFile f1, f2;
    PageCache c(4);
    c.read_page(f1, 0);
    c.read_page(f1, 1);
    c.read_page(f2, 0);
    c.read_page(f2, 1);
    EXPECT_EQ(c.size(), 4u);
}

TEST(MultiFile, CapacityIsSharedAcrossFiles) {
    TempFile f1, f2;
    PageCache c(3);
    c.read_page(f1, 0);
    c.read_page(f1, 1);
    c.read_page(f2, 0);    // fills to capacity
    c.read_page(f2, 1);    // must evict one page
    EXPECT_EQ(c.size(), 3u);
}

TEST(MultiFile, FlushTargetsOnlyOneFile) {
    TempFile f1, f2;
    PageCache c(8);
    c.write_page(f1, 0, make_page(0, 'A'));
    c.write_page(f2, 0, make_page(0, 'B'));
    c.flush(f1);
    // f1 page 0 is on disk; f2 page 0 may not be yet.
    EXPECT_EQ(read_raw(f1, 0)[0], 'A');
    // We cannot assert f2 is NOT on disk (eviction might have written it),
    // but the data must be correct once flushed.
    c.flush(f2);
    EXPECT_EQ(read_raw(f2, 0)[0], 'B');
}

TEST(MultiFile, ManyFilesCoexist) {
    constexpr int N = 8;
    std::vector<TempFile> files(N);
    PageCache c(N * 2);
    for (int i = 0; i < N; ++i)
        c.write_page(files[i], 0, make_page(0, static_cast<char>('A' + i)));
    for (int i = 0; i < N; ++i)
        EXPECT_EQ(c.read_page(files[i], 0).buf[0], static_cast<char>('A' + i));
}

// ────────────────────────────────────────────────────────────────────────────
// Suite 7 – close_file()
// ────────────────────────────────────────────────────────────────────────────

TEST(CloseFile, RemovesFilesPagesFromCache) {
    TempFile f;
    PageCache c(4);
    c.read_page(f, 0);
    c.read_page(f, 1);
    EXPECT_EQ(c.size(), 2u);
    c.close_file(f);
    EXPECT_EQ(c.size(), 0u);
}

TEST(CloseFile, FlushesBeforeEviction) {
    TempFile f;
    PageCache c(4);
    c.write_page(f, 0, make_page(0, 'C'));
    c.close_file(f);
    EXPECT_EQ(read_raw(f, 0)[0], 'C');
}

TEST(CloseFile, OnlyRemovesTargetFile) {
    TempFile f1, f2;
    PageCache c(8);
    c.read_page(f1, 0);
    c.read_page(f1, 1);
    c.read_page(f2, 0);
    c.close_file(f1);
    EXPECT_EQ(c.size(), 1u);     // only f2's page remains
    // f2 page must still be readable (cache hit).
    auto hits_before = c.stats().hits;
    c.read_page(f2, 0);
    EXPECT_EQ(c.stats().hits, hits_before + 1u);
}

TEST(CloseFile, ReopenAfterCloseWorksFine) {
    TempFile f;
    PageCache c(4);
    c.write_page(f, 0, make_page(0, 'R'));
    c.close_file(f);
    // Now reopen (new miss) and verify data persisted.
    const Page& p = c.read_page(f, 0);
    EXPECT_EQ(p.buf[0], 'R');
}

TEST(CloseFile, CloseNonExistentFileIsNoOp) {
    TempFile f;
    PageCache c(4);
    // Never opened f — closing it must not throw.
    EXPECT_NO_THROW(c.close_file(f));
}

// ────────────────────────────────────────────────────────────────────────────
// Suite 8 – Cache size bookkeeping
// ────────────────────────────────────────────────────────────────────────────

TEST(CacheSize, SizeIncreasesWithUniqueMisses) {
    TempFile f;
    PageCache c(8);
    for (uint32_t id = 0; id < 5; ++id) {
        c.read_page(f, id);
        EXPECT_EQ(c.size(), id + 1u);
    }
}

TEST(CacheSize, SizeDoesNotExceedCapacity) {
    TempFile f;
    constexpr std::size_t CAP = 3;
    PageCache c(CAP);
    for (uint32_t id = 0; id < 10; ++id)
        c.read_page(f, id);
    EXPECT_LE(c.size(), CAP);
}

TEST(CacheSize, HitsDoNotChangeSize) {
    TempFile f;
    PageCache c(4);
    c.read_page(f, 0);
    for (int i = 0; i < 5; ++i) c.read_page(f, 0);
    EXPECT_EQ(c.size(), 1u);
}

// ────────────────────────────────────────────────────────────────────────────
// Suite 9 – Flush semantics
// ────────────────────────────────────────────────────────────────────────────

TEST(Flush, FlushOnEmptyCacheIsNoOp) {
    TempFile f;
    PageCache c(4);
    EXPECT_NO_THROW(c.flush(f));
}

TEST(Flush, FlushAllOnEmptyCacheIsNoOp) {
    PageCache c(4);
    EXPECT_NO_THROW(c.flush_all());
}

TEST(Flush, FlushMakesPageReadableByRawIo) {
    TempFile f;
    PageCache c(4);
    c.write_page(f, 1, make_page(1, 'M'));
    c.flush(f);
    auto raw = read_raw(f, 1);
    for (char byte : raw) EXPECT_EQ(byte, 'M');
}

TEST(Flush, FlushAllMakesAllPagesReadableByRawIo) {
    TempFile f;
    PageCache c(4);
    for (uint32_t id = 0; id < 4; ++id)
        c.write_page(f, id, make_page(id, static_cast<char>('0' + id)));
    c.flush_all();
    for (uint32_t id = 0; id < 4; ++id)
        EXPECT_EQ(read_raw(f, id)[0], static_cast<char>('0' + id));
}

TEST(Flush, PageRemainsInCacheAfterFlush) {
    TempFile f;
    PageCache c(4);
    c.write_page(f, 0, make_page(0, 'K'));
    auto sz_before = c.size();
    c.flush(f);
    EXPECT_EQ(c.size(), sz_before);   // flush does not evict
    auto hits_before = c.stats().hits;
    c.read_page(f, 0);
    EXPECT_EQ(c.stats().hits, hits_before + 1u);
}

// ────────────────────────────────────────────────────────────────────────────
// Suite 10 – Disk persistence across cache instances
// ────────────────────────────────────────────────────────────────────────────

TEST(Persistence, PagesWrittenByOneInstanceReadByAnother) {
    TempFile f;
    constexpr uint32_t NPAGES = 6;
    {
        PageCache w(NPAGES + 1);
        for (uint32_t id = 0; id < NPAGES; ++id)
            w.write_page(f, id, make_page(id, static_cast<char>('A' + id)));
        // Destructor flushes.
    }
    PageCache r(NPAGES + 1);
    for (uint32_t id = 0; id < NPAGES; ++id) {
        const Page& p = r.read_page(f, id);
        EXPECT_EQ(p.id, id);
        EXPECT_EQ(p.buf[0], static_cast<char>('A' + id));
    }
}

TEST(Persistence, EvictedDirtyPagesPersistWithoutExplicitFlush) {
    TempFile f;
    {
        PageCache c(2);                         // small cache to force evictions
        for (uint32_t id = 0; id < 5; ++id)
            c.write_page(f, id, make_page(id, static_cast<char>('a' + id)));
        // No explicit flush; destructor must persist everything.
    }
    PageCache r(10);
    for (uint32_t id = 0; id < 5; ++id)
        EXPECT_EQ(r.read_page(f, id).buf[0], static_cast<char>('a' + id));
}

// ────────────────────────────────────────────────────────────────────────────
// Suite 11 – Page content correctness (byte-level)
// ────────────────────────────────────────────────────────────────────────────

TEST(PageContent, FullBufferIsCorrectAfterWrite) {
    TempFile f;
    PageCache c(4);
    Page written = make_page(0, '\xAB');
    c.write_page(f, 0, written);
    const Page& read = c.read_page(f, 0);
    EXPECT_EQ(read.buf, written.buf);
}

TEST(PageContent, FirstAndLastByteAfterRoundTrip) {
    TempFile f;
    PageCache c(4);
    Page p = make_page(0, '\xFF');
    p.buf.front() = '\x01';
    p.buf.back()  = '\x02';
    c.write_page(f, 0, p);
    c.flush(f);
    c.close_file(f);

    PageCache c2(4);
    const Page& r = c2.read_page(f, 0);
    EXPECT_EQ(static_cast<unsigned char>(r.buf.front()), 0x01u);
    EXPECT_EQ(static_cast<unsigned char>(r.buf.back()),  0x02u);
}

TEST(PageContent, MixedBytesPreservedOnDisk) {
    TempFile f;
    Page p;
    p.id = 0;
    std::iota(p.buf.begin(), p.buf.end(), '\x00');  // 0,1,2,...,255,0,1,...
    {
        PageCache c(4);
        c.write_page(f, 0, p);
        c.flush(f);
    }
    PageCache c2(4);
    EXPECT_EQ(c2.read_page(f, 0).buf, p.buf);
}

TEST(PageContent, NewPageBeyondEofIsZeroFilled) {
    TempFile f;
    PageCache c(4);
    // Write page 0, then read page 5 (should be zeroed, file smaller).
    c.write_page(f, 0, make_page(0, 'Z'));
    c.flush(f);
    c.close_file(f);

    PageCache c2(4);
    const Page& p = c2.read_page(f, 5);
    for (char byte : p.buf) EXPECT_EQ(byte, '\0');
}

// ────────────────────────────────────────────────────────────────────────────
// Suite 12 – Thread safety
// ────────────────────────────────────────────────────────────────────────────

TEST(ThreadSafety, ConcurrentReadsFromMultipleThreads) {
    TempFile f;
    PageCache c(16);
    // Pre-populate some pages.
    for (uint32_t id = 0; id < 8; ++id)
        c.write_page(f, id, make_page(id, static_cast<char>('A' + id)));
    c.flush(f);

    constexpr int THREADS = 8;
    std::vector<std::thread> threads;
    std::atomic<int> errors{0};

    for (int t = 0; t < THREADS; ++t) {
        threads.emplace_back([&, t]() {
            for (int iter = 0; iter < 100; ++iter) {
                uint32_t id = static_cast<uint32_t>(t % 8);
                try {
                    const Page& p = c.read_page(f, id);
                    if (p.buf[0] != static_cast<char>('A' + id))
                        ++errors;
                } catch (...) {
                    ++errors;
                }
            }
        });
    }
    for (auto& th : threads) th.join();
    EXPECT_EQ(errors.load(), 0);
}

TEST(ThreadSafety, ConcurrentWritesFromMultipleThreads) {
    TempFile f;
    PageCache c(32);
    constexpr int THREADS = 8;
    // Each thread owns its own page range to avoid data races on content.
    constexpr int PAGES_PER_THREAD = 4;

    std::vector<std::thread> threads;
    std::atomic<int> errors{0};

    for (int t = 0; t < THREADS; ++t) {
        threads.emplace_back([&, t]() {
            for (int id = 0; id < PAGES_PER_THREAD; ++id) {
                uint32_t page_id = static_cast<uint32_t>(t * PAGES_PER_THREAD + id);
                try {
                    c.write_page(f, page_id,
                                 make_page(page_id, static_cast<char>('A' + t)));
                } catch (...) {
                    ++errors;
                }
            }
        });
    }
    for (auto& th : threads) th.join();
    EXPECT_EQ(errors.load(), 0);
}

TEST(ThreadSafety, ConcurrentMixedReadWriteFlush) {
    TempFile f;
    PageCache c(8);
    std::atomic<int> errors{0};

    std::vector<std::thread> threads;

    // Writer threads.
    for (int t = 0; t < 4; ++t) {
        threads.emplace_back([&, t]() {
            for (int i = 0; i < 50; ++i) {
                try {
                    c.write_page(f,
                                 static_cast<uint32_t>((t * 50 + i) % 16),
                                 make_page(static_cast<uint32_t>(i), static_cast<char>(t)));
                } catch (...) { ++errors; }
            }
        });
    }

    // Reader threads.
    for (int t = 0; t < 4; ++t) {
        threads.emplace_back([&, t]() {
            for (int i = 0; i < 50; ++i) {
                try {
                    (void)c.read_page(f, static_cast<uint32_t>((t * 13 + i) % 16));
                } catch (...) { ++errors; }
            }
        });
    }

    // Flush thread.
    threads.emplace_back([&]() {
        for (int i = 0; i < 20; ++i) {
            try { c.flush(f); } catch (...) { ++errors; }
            std::this_thread::yield();
        }
    });

    for (auto& th : threads) th.join();
    EXPECT_EQ(errors.load(), 0);
}

TEST(ThreadSafety, SizeAndStatsAreConsistentUnderConcurrency) {
    TempFile f;
    PageCache c(4);
    std::atomic<bool> go{false};
    std::vector<std::thread> threads;

    for (int t = 0; t < 4; ++t) {
        threads.emplace_back([&, t]() {
            while (!go.load()) std::this_thread::yield();
            for (int i = 0; i < 100; ++i)
                (void)c.read_page(f, static_cast<uint32_t>(t * 4 + (i % 4)));
        });
    }

    go = true;
    for (auto& th : threads) th.join();

    // Just check these don't crash and are within sensible bounds.
    EXPECT_LE(c.size(), c.capacity());
    auto s = c.stats();
    EXPECT_GE(s.hits + s.misses, 400u); // 4 threads × 100 ops
}

// ────────────────────────────────────────────────────────────────────────────
// Suite 13 – Edge cases
// ────────────────────────────────────────────────────────────────────────────

TEST(EdgeCases, ReadAndWritePageZeroRepeatedly) {
    TempFile f;
    PageCache c(4);
    for (char fill = 'A'; fill <= 'Z'; ++fill) {
        c.write_page(f, 0, make_page(0, fill));
        EXPECT_EQ(c.read_page(f, 0).buf[0], fill);
    }
}

TEST(EdgeCases, InterleaveReadsAndWrites) {
    TempFile f;
    PageCache c(8);
    for (uint32_t id = 0; id < 5; ++id) {
        c.write_page(f, id, make_page(id, 'W'));
        const Page& p = c.read_page(f, id);
        EXPECT_EQ(p.buf[0], 'W');
    }
}

TEST(EdgeCases, FlushAfterNoWrites) {
    TempFile f;
    PageCache c(4);
    for (uint32_t id = 0; id < 4; ++id) c.read_page(f, id);
    EXPECT_NO_THROW(c.flush(f));
    EXPECT_NO_THROW(c.flush_all());
}

TEST(EdgeCases, CloseFileThenWriteThenRead) {
    TempFile f;
    PageCache c(4);
    c.write_page(f, 0, make_page(0, 'B'));
    c.close_file(f);
    c.write_page(f, 0, make_page(0, 'E'));
    EXPECT_EQ(c.read_page(f, 0).buf[0], 'E');
}

TEST(EdgeCases, MultipleCloseFileCallsAreIdempotent) {
    TempFile f;
    PageCache c(4);
    c.write_page(f, 0, make_page(0, 'I'));
    EXPECT_NO_THROW(c.close_file(f));
    EXPECT_NO_THROW(c.close_file(f));   // second close must not throw/crash
}

TEST(EdgeCases, LargeNumberOfPagesExceedingCapacity) {
    TempFile f;
    constexpr std::size_t CAP = 8;
    constexpr uint32_t    N   = 256;
    {
        PageCache c(CAP);
        for (uint32_t id = 0; id < N; ++id)
            c.write_page(f, id, make_page(id, static_cast<char>(id & 0x7F)));
        EXPECT_LE(c.size(), CAP);
    }
    // Verify all pages were persisted (destructor flushed the rest).
    PageCache r(N);
    for (uint32_t id = 0; id < N; ++id)
        EXPECT_EQ(static_cast<unsigned char>(r.read_page(f, id).buf[0]),
                  static_cast<unsigned char>(id & 0x7F));
}

TEST(EdgeCases, FlushAllWithMultipleFilesAndCapacityCrossover) {
    TempFile f1, f2, f3;
    PageCache c(5);
    c.write_page(f1, 0, make_page(0, '1'));
    c.write_page(f2, 0, make_page(0, '2'));
    c.write_page(f3, 0, make_page(0, '3'));
    c.write_page(f1, 1, make_page(1, 'X'));
    c.write_page(f2, 1, make_page(1, 'Y'));   // this evicts one page
    c.flush_all();
    EXPECT_EQ(read_raw(f2, 1)[0], 'Y');
}

```
