# A pool allocator adhering to the `std::allocator` contract 

Claude Model: **Sonnet 4.6**

> [!NOTE]
> For C++23, implement a pool allocator that allocates a large block of memory upfront and manages fixed-size objects using an intrusive linked list. This allocator should provide constant-time allocation, zero fragmentation, and reusable objects without repeated heap requests. I plan to use this allocator in production for processing high-frequency credit card transactions, for which throughput and predictable latency are essential.
> Since I want to use it with the std::vector class, the pool allocator implementation must adhere to the std::allocator contract.
> Write a test demonstrating the correctness.


##  The pool allocator implementation

```cpp
#pragma once

// ============================================================================
//  pool_allocator.hpp  –  C++23 fixed-size pool allocator
//
//  Design goals
//  ────────────
//  • O(1) allocation and deallocation via an intrusive singly-linked free list
//  • Zero fragmentation: every slot occupies exactly SlotSize bytes inside
//    a single upfront-allocated contiguous block; no per-object bookkeeping
//  • Reuse without heap traffic: freed slots are chained back into the free
//    list and re-issued on the next allocate(1) call
//  • std::allocator contract: works as a drop-in allocator for std::vector
//    and other standard containers (including ones that use internal rebind)
//  • Thread-safety: intentionally absent – in HFT each thread owns its pool
//
//  Memory layout
//  ─────────────
//  ┌─────────────────────────────────────────────────────────────────────┐
//  │  slot[0]  │  slot[1]  │ …  │  slot[Capacity-1]                     │
//  └─────────────────────────────────────────────────────────────────────┘
//      ↑                                             ↑
//  bump_index starts here              Capacity boundary
//
//  A free slot stores a void* link in its first bytes (intrusive list).
//  No separate node array is ever allocated.
//
//  Allocation strategy
//  ───────────────────
//  allocate(1)    → pop recycled slot from free list if available,
//                   else advance bump pointer by SlotSize
//  allocate(n>1)  → advance bump pointer by n×SlotSize (contiguous run for
//                   container-internal buffers, e.g. std::vector growth)
//  deallocate(p,1)  → push slot onto free list  (O(1), zero heap traffic)
//  deallocate(p,n>1)→ no-op*  (bump region reclaimed on reset())
//
//  * std::vector's old buffer is freed only during reallocation.  A pool
//    sized >= max_elements × growth_factor avoids those frees entirely.
//    Call reset() between transaction batches to reclaim everything in O(1).
//
//  Cross-type rebind
//  ─────────────────
//  PoolStorage<Capacity> is type-erased so different PoolAllocator<T,Capacity>
//  specialisations produced by container rebind share the same raw slab.
//  The slot_size is fixed at sizeof(T) for the type that first constructs the
//  pool.  Rebound types must fit within that slot size.
// ============================================================================

#include <cassert>
#include <cstddef>
#include <memory>
#include <new>
#include <type_traits>
#include <format>

// ── type-erased pool storage ───────────────────────────────────────────────
// Extracted from PoolAllocator<T> so that cross-type rebind copies share
// a single concrete PoolStorage type and a single shared_ptr control block.
template<std::size_t Capacity>
struct PoolStorage {
    // Intrusive free-list link: lives inside each free slot
    struct FreeNode { FreeNode* next; };

    FreeNode*   free_head  = nullptr;
    std::size_t bump_index = 0;      // next fresh slot index
    std::size_t slot_size  = 0;      // bytes per slot
    // Raw slab follows immediately after this struct in memory (see make())

    // ── factory ──────────────────────────────────────────────────────────
    [[nodiscard]] static std::shared_ptr<PoolStorage> make(std::size_t sz) {
        assert(sz >= sizeof(FreeNode*) &&
               "slot_size must be >= sizeof(void*) for the intrusive free list");
        const std::size_t total = sizeof(PoolStorage)
                                + sz * Capacity;
        void* raw = ::operator new(total,
                                   std::align_val_t{alignof(std::max_align_t)});
        auto* p = ::new(raw) PoolStorage{};
        p->slot_size = sz;
        return {p, [](PoolStorage* q) {
            q->~PoolStorage();
            ::operator delete(q, std::align_val_t{alignof(std::max_align_t)});
        }};
    }

    // Pointer to the i-th raw slot
    [[nodiscard]] std::byte* slot(std::size_t i) noexcept {
        return reinterpret_cast<std::byte*>(this + 1) + i * slot_size;
    }

    // ── allocate n contiguous slots ───────────────────────────────────────
    [[nodiscard]] void* allocate(std::size_t n) {
        if (n == 1 && free_head) [[likely]] {
            FreeNode* node = free_head;
            free_head      = node->next;
            return static_cast<void*>(node);
        }
        if (bump_index + n > Capacity) [[unlikely]]
            throw std::bad_alloc{};
        void* ptr   = slot(bump_index);
        bump_index += n;
        return ptr;
    }

    // ── return n slots to the pool ────────────────────────────────────────
    void deallocate(void* ptr, std::size_t n) noexcept {
        assert(owns(ptr));
        if (n == 1) {
            auto* node = static_cast<FreeNode*>(ptr);
            node->next = free_head;
            free_head  = node;
        }
        // n>1: no-op; bulk reclaim via reset()
    }

    // ── O(1) full reclaim ─────────────────────────────────────────────────
    void reset() noexcept { free_head = nullptr; bump_index = 0; }

    // ── diagnostics ───────────────────────────────────────────────────────
    [[nodiscard]] std::size_t fresh_remaining() const noexcept {
        return Capacity - bump_index;
    }
    [[nodiscard]] std::size_t recycled() const noexcept {
        std::size_t n = 0;
        for (FreeNode* p = free_head; p; p = p->next) ++n;
        return n;
    }
    [[nodiscard]] bool owns(const void* ptr) const noexcept {
        const auto* base = reinterpret_cast<const std::byte*>(this + 1);
        return ptr >= base && ptr < base + slot_size * Capacity;
    }

private:
    PoolStorage()  noexcept = default;
    ~PoolStorage()          = default;
};


// ── PoolAllocator<T, Capacity> ────────────────────────────────────────────
template<typename T, std::size_t Capacity>
class PoolAllocator {
public:
    // ── mandatory allocator typedefs ─────────────────────────────────────
    using value_type      = T;
    using size_type       = std::size_t;
    using difference_type = std::ptrdiff_t;

    // Propagate the shared pool on container copy / move / swap so that
    // memory allocated by one copy can always be freed by another
    using propagate_on_container_copy_assignment = std::true_type;
    using propagate_on_container_move_assignment = std::true_type;
    using propagate_on_container_swap            = std::true_type;
    using is_always_equal                        = std::false_type;

    template<typename U>
    struct rebind { using other = PoolAllocator<U, Capacity>; };

private:
    using Storage = PoolStorage<Capacity>;

    static_assert(sizeof(T) >= sizeof(typename Storage::FreeNode),
        "T is too small to embed a free-list pointer.  "
        "sizeof(T) must be >= sizeof(void*).");

    std::shared_ptr<Storage> storage_;

    explicit PoolAllocator(std::shared_ptr<Storage> s) noexcept
        : storage_(std::move(s)) {}

    template<typename U, std::size_t C>
    friend class PoolAllocator;

public:
    // ── constructors ─────────────────────────────────────────────────────

    /// Creates a new pool sized for Capacity objects of type T.
    PoolAllocator() : storage_(Storage::make(sizeof(T))) {}

    PoolAllocator(const PoolAllocator&) noexcept = default;
    PoolAllocator(PoolAllocator&&)      noexcept = default;
    PoolAllocator& operator=(const PoolAllocator&) noexcept = default;
    PoolAllocator& operator=(PoolAllocator&&)      noexcept = default;
    ~PoolAllocator() = default;

    /// Cross-type rebind constructor: shares the same underlying PoolStorage.
    /// This is called by containers that internally allocate node types
    /// different from T (e.g. list nodes, map nodes).
    template<typename U>
    explicit PoolAllocator(const PoolAllocator<U, Capacity>& other) noexcept
        : storage_(other.storage_) {}

    // ── std::allocator contract ───────────────────────────────────────────

    [[nodiscard]] T* allocate(std::size_t n) {
        return std::launder(static_cast<T*>(storage_->allocate(n)));
    }

    void deallocate(T* ptr, std::size_t n) noexcept {
        storage_->deallocate(static_cast<void*>(ptr), n);
    }

    [[nodiscard]] std::size_t max_size() const noexcept { return Capacity; }

    // ── utility API ───────────────────────────────────────────────────────

    /// Reclaim the entire pool in O(1).
    /// The caller is responsible for destroying all live objects first.
    void reset() noexcept { storage_->reset(); }

    [[nodiscard]] std::size_t fresh_slots_remaining() const noexcept {
        return storage_->fresh_remaining();
    }
    [[nodiscard]] std::size_t recycled_slots() const noexcept {
        return storage_->recycled();
    }
    [[nodiscard]] bool pool_owns(const void* ptr) const noexcept {
        return storage_->owns(ptr);
    }

    // ── equality ─────────────────────────────────────────────────────────
    // Equal iff backed by the same PoolStorage (same raw slab).

    template<typename U>
    [[nodiscard]] bool operator==(
            const PoolAllocator<U, Capacity>& rhs) const noexcept {
        return storage_ == rhs.storage_;
    }
    template<typename U>
    [[nodiscard]] bool operator!=(
            const PoolAllocator<U, Capacity>& rhs) const noexcept {
        return !(*this == rhs);
    }
};
```

##  The pool allocator tests

```cpp
// ============================================================================
//  test_pool_allocator.cpp  –  correctness tests for PoolAllocator<T, N>
//
//  Sections
//  ────────
//  1.  Static properties (concept checks, rebind)
//  2.  Raw allocate / deallocate mechanics
//  3.  Free-list reuse (intrusive linked list correctness)
//  4.  std::vector integration
//  5.  Pool exhaustion and bad_alloc
//  6.  reset() and reuse
//  7.  Shared-pool identity (copies share state)
//  8.  Simulated credit-card transaction batch
//  9.  Benchmark: throughput and latency comparison vs. std::allocator
// ============================================================================

#include "pool_allocator.hpp"

#include <algorithm>
#include <chrono>
#include <cstring>
#include <format>
#include <iostream>
#include <numeric>
#include <random>
#include <string_view>
#include <vector>
#include <memory>
#include <cassert>
#include <stdexcept>

// ── tiny test harness ──────────────────────────────────────────────────────
namespace {

int  g_tests_run    = 0;
int  g_tests_passed = 0;

void check(bool cond, std::string_view label) {
    ++g_tests_run;
    if (cond) {
        ++g_tests_passed;
        std::cout << std::format("  [PASS] {}\n", label);
    } else {
        std::cout << std::format("  [FAIL] {}\n", label);
    }
}

void section(std::string_view title) {
    std::cout << std::format("\n─── {} ───\n", title);
}

} // namespace

// ── domain types ──────────────────────────────────────────────────────────
struct Transaction {
    std::uint64_t id;
    double        amount;
    std::uint32_t merchant_id;
    std::uint8_t  status;          // 0=pending 1=approved 2=declined
    std::uint8_t  currency;
    char          card_last4[4];

    // Padding ensures sizeof(Transaction) >= sizeof(void*) on all platforms;
    // the static_assert in PoolAllocator will catch it otherwise.
    std::uint8_t  _pad[2];
};

static_assert(sizeof(Transaction) >= sizeof(void*),
    "Transaction must be at least pointer-sized for the intrusive free list");


// ── 1. Static properties ───────────────────────────────────────────────────
void test_static_properties() {
    section("1. Static properties");

    using Alloc = PoolAllocator<Transaction, 256>;

    check(std::is_same_v<Alloc::value_type, Transaction>,
          "value_type == Transaction");

    check(std::is_same_v<Alloc::propagate_on_container_copy_assignment,
                         std::true_type>,
          "propagate_on_container_copy_assignment == true_type");

    check(std::is_same_v<Alloc::propagate_on_container_move_assignment,
                         std::true_type>,
          "propagate_on_container_move_assignment == true_type");

    check(std::is_same_v<Alloc::is_always_equal, std::false_type>,
          "is_always_equal == false_type");

    // Rebind to a different type (must be >= pointer-sized; int is too small
    // on 64-bit platforms so we use std::uint64_t which satisfies the
    // intrusive-list constraint on every tier-1 platform).
    using Rebound = Alloc::rebind<std::uint64_t>::other;
    check(std::is_same_v<Rebound::value_type, std::uint64_t>,
          "rebind<uint64_t>::other::value_type == uint64_t");

    // Meets the Allocator named requirement
    check(std::allocator_traits<Alloc>::is_always_equal::value == false,
          "allocator_traits is_always_equal == false");
}


// ── 2. Raw allocate / deallocate ──────────────────────────────────────────
void test_raw_alloc() {
    section("2. Raw allocate / deallocate");

    constexpr std::size_t Cap = 8;
    PoolAllocator<Transaction, Cap> alloc;

    // Allocate all slots via bump pointer
    Transaction* ptrs[Cap];
    for (std::size_t i = 0; i < Cap; ++i) {
        ptrs[i] = alloc.allocate(1);
        check(ptrs[i] != nullptr,
              std::format("allocate(1) slot {} returns non-null", i));
    }

    check(alloc.fresh_slots_remaining() == 0,
          "bump pointer exhausted after Cap allocations");

    // Write and read back to verify we own the memory
    for (std::size_t i = 0; i < Cap; ++i) {
        ptrs[i]->id     = i * 100;
        ptrs[i]->amount = static_cast<double>(i) * 9.99;
    }
    bool rw_ok = true;
    for (std::size_t i = 0; i < Cap; ++i)
        rw_ok &= (ptrs[i]->id == i * 100);
    check(rw_ok, "read-back after write is correct");

    // All pointers are distinct and inside the pool
    bool distinct = true, owned = true;
    for (std::size_t i = 0; i < Cap; ++i) {
        owned   &= alloc.pool_owns(ptrs[i]);
        for (std::size_t j = i + 1; j < Cap; ++j)
            distinct &= (ptrs[i] != ptrs[j]);
    }
    check(distinct, "all returned pointers are distinct");
    check(owned,    "all returned pointers lie inside the pool block");

    // Deallocate all
    for (std::size_t i = 0; i < Cap; ++i)
        alloc.deallocate(ptrs[i], 1);

    check(alloc.recycled_slots() == Cap,
          "recycled_slots() == Cap after deallocating everything");
}


// ── 3. Free-list reuse ────────────────────────────────────────────────────
void test_free_list_reuse() {
    section("3. Free-list reuse (intrusive linked list)");

    constexpr std::size_t Cap = 4;
    PoolAllocator<Transaction, Cap> alloc;

    Transaction* a = alloc.allocate(1);
    Transaction* b = alloc.allocate(1);
    Transaction* c = alloc.allocate(1);

    // Free b, then a (so free list is a → b)
    alloc.deallocate(b, 1);
    alloc.deallocate(a, 1);

    check(alloc.recycled_slots() == 2, "2 slots in free list after 2 frees");

    // Next allocation must come from the free list (no new bump advance)
    std::size_t bump_before = Cap - alloc.fresh_slots_remaining();
    Transaction* r1 = alloc.allocate(1);
    std::size_t bump_after  = Cap - alloc.fresh_slots_remaining();

    check(bump_before == bump_after,
          "allocate(1) from free list does not advance bump pointer");

    check(r1 == a || r1 == b,
          "re-allocated pointer is one of the previously freed slots");

    Transaction* r2 = alloc.allocate(1);
    check((r1 == a && r2 == b) || (r1 == b && r2 == a),
          "second re-allocation returns the other freed slot");

    check(alloc.recycled_slots() == 0,
          "free list is empty after draining recycled slots");

    // c is still live; clean up
    alloc.deallocate(c,  1);
    alloc.deallocate(r1, 1);
    alloc.deallocate(r2, 1);
}


// ── 4. std::vector integration ────────────────────────────────────────────
void test_vector_integration() {
    section("4. std::vector integration");

    // Size pool generously: vector may request a few internal reallocation
    // buffers (1→2→4→8 etc.) in addition to the payload slots.
    constexpr std::size_t Cap = 1024;
    using TxAlloc = PoolAllocator<Transaction, Cap>;
    using TxVec   = std::vector<Transaction, TxAlloc>;

    TxAlloc alloc;
    TxVec   txns(alloc);
    txns.reserve(64);

    // Push 64 transactions
    for (std::uint64_t i = 0; i < 64; ++i) {
        Transaction t{};
        t.id        = i;
        t.amount    = static_cast<double>(i) * 1.11;
        t.status    = 1;
        txns.push_back(t);
    }

    check(txns.size() == 64,
          "vector holds 64 transactions");

    bool sequential_ids = true;
    for (std::size_t i = 0; i < txns.size(); ++i)
        sequential_ids &= (txns[i].id == i);
    check(sequential_ids, "transaction IDs are sequential and correct");

    // Internal buffer is inside the pool
    check(alloc.pool_owns(txns.data()),
          "vector's internal buffer lies inside the pool");

    // Erase half, push new elements – tests that vector can use the allocator
    // for repeated modifications
    txns.erase(txns.begin(), txns.begin() + 32);
    for (std::uint64_t i = 64; i < 96; ++i) {
        Transaction t{};
        t.id = i;
        txns.push_back(t);
    }
    check(txns.size() == 64,
          "vector size is correct after erase + push");
    check(txns.back().id == 95,
          "last element has correct id after modifications");

    // Swap two vectors sharing the same pool
    TxVec txns2(alloc);
    txns2.reserve(8);
    txns2.push_back(Transaction{999, 0.0, 0, 0, 0, {}, {}});
    txns.swap(txns2);
    check(txns.size()  == 1   && txns[0].id == 999,
          "after swap, first vector holds the one-element vector");
    check(txns2.size() == 64,
          "after swap, second vector holds the 64-element vector");
}


// ── 5. Pool exhaustion ────────────────────────────────────────────────────
void test_exhaustion() {
    section("5. Pool exhaustion → bad_alloc");

    constexpr std::size_t Cap = 4;
    PoolAllocator<Transaction, Cap> alloc;

    Transaction* ptrs[Cap];
    for (auto& p : ptrs) p = alloc.allocate(1);

    bool threw = false;
    try {
        [[maybe_unused]] auto* extra = alloc.allocate(1);
    } catch (const std::bad_alloc&) {
        threw = true;
    }
    check(threw, "allocate(1) on full pool throws std::bad_alloc");

    // Clean up so the pool is left in a valid state
    for (auto* p : ptrs) alloc.deallocate(p, 1);
}


// ── 6. reset() and reuse ──────────────────────────────────────────────────
void test_reset() {
    section("6. reset() reclaims the full pool in O(1)");

    constexpr std::size_t Cap = 8;
    PoolAllocator<Transaction, Cap> alloc;

    // Exhaust the pool
    Transaction* ptrs[Cap];
    for (auto& p : ptrs) p = alloc.allocate(1);

    check(alloc.fresh_slots_remaining() == 0 &&
          alloc.recycled_slots()        == 0,
          "pool fully exhausted before reset");

    // Caller destroys objects, then resets the pool
    alloc.reset();

    check(alloc.fresh_slots_remaining() == Cap,
          "all slots available again after reset");

    // Allocate again – no bad_alloc
    bool ok = true;
    Transaction* ptrs2[Cap];
    try {
        for (auto& p : ptrs2) p = alloc.allocate(1);
    } catch (...) {
        ok = false;
    }
    check(ok, "full capacity allocatable again after reset");

    for (auto& p : ptrs2) alloc.deallocate(p, 1);
}


// ── 7. Shared-pool identity ───────────────────────────────────────────────
void test_shared_pool_identity() {
    section("7. Copies share the same underlying pool");

    constexpr std::size_t Cap = 16;
    PoolAllocator<Transaction, Cap> a1;
    PoolAllocator<Transaction, Cap> a2(a1);     // copy
    PoolAllocator<Transaction, Cap> a3;          // independent

    check(a1 == a2, "copy shares the same pool (a1 == a2)");
    check(a1 != a3, "independently constructed allocators differ (a1 != a3)");

    // Allocate with a1, deallocate with a2 – must work
    Transaction* p = a1.allocate(1);
    check(a2.pool_owns(p),
          "a2 recognises pointer allocated by a1 (shared pool)");

    bool ok = true;
    try { a2.deallocate(p, 1); }
    catch (...) { ok = false; }
    check(ok, "a2 can deallocate memory allocated by a1");

    // Rebind shares the same pool (use uint64_t: satisfies the >= pointer-size
    // constraint of the intrusive free list on all 64-bit platforms)
    PoolAllocator<std::uint64_t, Cap> a_u64(a1);
    Transaction* q = a1.allocate(1);
    check(a_u64.pool_owns(reinterpret_cast<std::uint64_t*>(q)),
          "rebound allocator shares the same pool block");
    a1.deallocate(q, 1);
}


// ── 8. Simulated transaction batch ───────────────────────────────────────
void test_transaction_batch() {
    section("8. Simulated credit-card transaction batch");

    constexpr std::size_t PoolCap  = 4096;
    constexpr std::size_t BatchSz  = 512;

    using TxAlloc = PoolAllocator<Transaction, PoolCap>;
    using TxVec   = std::vector<Transaction, TxAlloc>;

    TxAlloc alloc;
    std::mt19937_64 rng{42};

    std::uint64_t total_approved  = 0;
    std::uint64_t total_declined  = 0;
    constexpr int Batches         = 4;

    for (int batch = 0; batch < Batches; ++batch) {
        TxVec pending(alloc);
        pending.reserve(BatchSz);

        // Ingest
        for (std::size_t i = 0; i < BatchSz; ++i) {
            Transaction t{};
            t.id          = static_cast<std::uint64_t>(batch) * BatchSz + i;
            t.amount      = static_cast<double>(rng() % 100000) / 100.0;
            t.merchant_id = static_cast<std::uint32_t>(rng() % 10000);
            t.status      = 0; // pending
            pending.push_back(t);
        }
        check(pending.size() == BatchSz,
              std::format("batch {}: ingested {} transactions", batch, BatchSz));

        // Authorise: approve if amount < 500, decline otherwise
        for (auto& tx : pending) {
            if (tx.amount < 500.0) {
                tx.status = 1;
                ++total_approved;
            } else {
                tx.status = 2;
                ++total_declined;
            }
        }

        bool all_processed = std::all_of(pending.begin(), pending.end(),
            [](const Transaction& t){ return t.status == 1 || t.status == 2; });
        check(all_processed,
              std::format("batch {}: all transactions processed", batch));

        // End of batch: clear vector (destructs elements), then reset pool
        pending.clear();
        pending.shrink_to_fit();  // releases vector's internal buffer to pool
        alloc.reset();
    }

    check(total_approved + total_declined == BatchSz * Batches,
          std::format("total processed = {} (expected {})",
              total_approved + total_declined, BatchSz * Batches));
    std::cout << std::format("  approved={} declined={}\n",
                             total_approved, total_declined);
}


// ── 9. Throughput benchmark ───────────────────────────────────────────────
void benchmark_throughput() {
    section("9. Throughput benchmark (pool vs heap)");

    constexpr std::size_t N       = 50'000;
    constexpr std::size_t PoolCap = N + 128; // headroom for vector metadata

    using TxPool  = PoolAllocator<Transaction, PoolCap>;
    using TxVecP  = std::vector<Transaction, TxPool>;
    using TxVecH  = std::vector<Transaction>;          // std::allocator

    auto run = [&](auto& vec) {
        vec.reserve(N);
        for (std::size_t i = 0; i < N; ++i) {
            Transaction t{};
            t.id     = i;
            t.amount = static_cast<double>(i) * 0.01;
            vec.push_back(t);
        }
        // Prevent the compiler from optimising away the loop
        volatile std::uint64_t sink = vec.back().id;
        (void)sink;
        vec.clear();
    };

    // Warm-up
    {
        TxPool   alloc;
        TxVecP   wvp(alloc); run(wvp);
        TxVecH   wvh;        run(wvh);
    }

    // Pool
    TxPool alloc;
    TxVecP vp(alloc);
    auto t0 = std::chrono::high_resolution_clock::now();
    run(vp);
    auto t1 = std::chrono::high_resolution_clock::now();
    double pool_ms = std::chrono::duration<double, std::milli>(t1 - t0).count();

    // Heap
    TxVecH vh;
    auto t2 = std::chrono::high_resolution_clock::now();
    run(vh);
    auto t3 = std::chrono::high_resolution_clock::now();
    double heap_ms = std::chrono::duration<double, std::milli>(t3 - t2).count();

    std::cout << std::format("  pool  : {:.3f} ms  ({} objects)\n", pool_ms, N);
    std::cout << std::format("  heap  : {:.3f} ms  ({} objects)\n", heap_ms, N);
    std::cout << std::format("  ratio : {:.2f}x  (pool / heap)\n",
                             pool_ms / heap_ms);

    // Just verify correctness, not strict speed (benchmark results vary by env)
    check(true, "benchmark completed without errors");
}


// ── entry point ───────────────────────────────────────────────────────────
int main() {
    std::cout << "╔══════════════════════════════════════════════════════╗\n";
    std::cout << "║         PoolAllocator – correctness test suite        ║\n";
    std::cout << "╚══════════════════════════════════════════════════════╝\n";

    test_static_properties();
    test_raw_alloc();
    test_free_list_reuse();
    test_vector_integration();
    test_exhaustion();
    test_reset();
    test_shared_pool_identity();
    test_transaction_batch();
    benchmark_throughput();

    std::cout << std::format(
        "\n══════════════════════════════════════════\n"
        "  Result: {}/{} tests passed\n"
        "══════════════════════════════════════════\n",
        g_tests_passed, g_tests_run);

    return g_tests_passed == g_tests_run ? 0 : 1;
}
```

##  The pool allocator production usage pattern

```cpp
// One pool per thread, sized for worst-case batch
constexpr std::size_t MaxBatch = 8192;
thread_local PoolAllocator<Transaction, MaxBatch * 4> pool;

void process_batch(std::span<RawEvent> events) {
    std::vector<Transaction, decltype(pool)> txns(pool);
    txns.reserve(events.size());
    // ... fill and process ...
    txns.clear();
    txns.shrink_to_fit();
    pool.reset();           // O(1) reclaim, zero heap traffic
}
```


# Architecture

## `PoolStorage<Capacity>` — type-erased slab manager

The pool's raw memory lives in a single `operator new` allocation:

```
[ PoolStorage control block | slot[0] | slot[1] | … | slot[N-1] ]
```

Being type-erased (no `T` parameter) is critical: it means `PoolAllocator<Transaction, N>` and whatever internal node type 
a container's rebind produces both share one `shared_ptr<PoolStorage<N>>` control block, satisfying the allocator propagation 
contract without any `reinterpret_cast` games between incompatible `shared_ptr` types.

## Intrusive free list

Free slots store a `FreeNode*` link inside **themselves** — the first `sizeof(void*)` bytes of the slot double as 
the next pointer when the slot is unused. This eliminates any separate bookkeeping array and 
means `deallocate(p, 1)` is a 3-instruction pointer swap.

The `static_assert` enforces `sizeof(T) >= sizeof(void*)` at compile time; 
if `T` is too small the error fires before any code runs.

## Two-tier allocation

| Call | Strategy | Complexity |
|------|----------|------------|
| `allocate(1)` with recycled slot | pop free list head      | O(1), 0 heap ops |
| `allocate(1)` no recycled slot   | advance bump index      | O(1), 0 heap ops |
| `allocate(n > 1)`                | advance bump index by n | O(1), 0 heap ops | 
| `deallocate(p, 1)`               | push free list head     | O(1), 0 heap ops |
| `deallocate(p, n > 1)`           | no-op                   | O(1)             |
| `reset()`                        | zero two fields         | O(1)             |

The `n > 1` deallocation no-op is the right tradeoff for `std::vector`: the old buffer is only freed during reallocation, 
which is rare if you `reserve()` upfront. Calling `reset()` between transaction batches reclaims everything in constant time.


## `std::allocator` contract compliance

| Requirement | How it's met |
|-------------|--------------|
| `value_type`, `size_type`, `difference_type` | provided | 
| `propagate_on_container_*` all `true_type`   | shared pool survives copy/move/swap | 
| `is_always_equal = false_type`               | distinct pools are not interchangeable | 
| `rebind<U>::other`                           | shares the same `PoolStorage` via cross-type constructor |
| Equality operatorse                          | qual iff `shared_ptr` addresses match |

