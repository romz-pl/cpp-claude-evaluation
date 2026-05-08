# Evaluation of AVL tree in C++23

Claude Model: **Sonnet 4.6**

> [!NOTE]
> In the context of C++23 ISO Standard and C++ Core Guidelines evaluate the following program

```cpp
// =============================================================================
// avl_map.hpp  —  A C++23 AVL-tree associative container
//
// Named after its inventors Adelson-Velsky and Landis, an AVL tree is a
// self-balancing binary search tree that guarantees the heights of the two
// child subtrees of any node differ by no more than one.  This gives O(log n)
// worst-case guarantees for insert, erase and lookup.
//
// The public interface is a drop-in replacement for std::map.
//
// Design notes
// ------------
//  • Nodes own their key–value pair (std::pair<const Key, T>).
//  • A raw nullptr is used as the "past-the-end" sentinel so that end() is
//    lightweight and no dummy node allocation is required.
//  • Bidirectional iteration is implemented by walking parent pointers.
//  • Balance is maintained after every insert/erase via local rotations that
//    travel up to the root (O(log n) rotations per operation).
//  • All standard allocator-traits hooks (POCCA / POCMA / POCS) are observed.
//  • Transparent comparison (is_transparent) is supported throughout.
//  • C++ Core Guidelines are followed: no raw owning pointers outside RAII
//    wrappers, Expects / Ensures-style assertions via assert(), value
//    semantics preferred, rule-of-five on every class.
// =============================================================================

#pragma once

#include <algorithm>
#include <cassert>
#include <compare>
#include <concepts>
#include <functional>
#include <initializer_list>
#include <iterator>
#include <memory>
#include <optional>
#include <stdexcept>
#include <utility>

// =============================================================================
// avl_map
// =============================================================================
template<
    typename Key,
    typename T,
    typename Compare   = std::less<Key>,
    typename Allocator = std::allocator<std::pair<const Key, T>>
>
class avl_map {
    // =========================================================================
    // Internal node
    // =========================================================================
    struct Node {
        std::pair<const Key, T> kv;   // stored key–value pair
        Node* parent  = nullptr;
        Node* left    = nullptr;
        Node* right   = nullptr;
        int   height  = 1;            // height of subtree rooted here

        template<typename... Args>
        explicit Node(Args&&... args)
            : kv(std::forward<Args>(args)...) {}

        // Non-copyable, non-movable: nodes live on the allocator heap only
        Node(const Node&)            = delete;
        Node& operator=(const Node&) = delete;
    };

    using NodeAlloc  = typename std::allocator_traits<Allocator>
                                  ::template rebind_alloc<Node>;
    using NodeTraits = std::allocator_traits<NodeAlloc>;

public:
    // =========================================================================
    // Public type aliases (mirror std::map)
    // =========================================================================
    using key_type        = Key;
    using mapped_type     = T;
    using value_type      = std::pair<const Key, T>;
    using size_type       = std::size_t;
    using difference_type = std::ptrdiff_t;
    using key_compare     = Compare;
    using allocator_type  = Allocator;
    using reference       = value_type&;
    using const_reference = const value_type&;
    using pointer         = typename std::allocator_traits<Allocator>::pointer;
    using const_pointer   = typename std::allocator_traits<Allocator>::const_pointer;

    // =========================================================================
    // value_compare
    // =========================================================================
    class value_compare {
    public:
        bool operator()(const value_type& a, const value_type& b) const {
            return comp_(a.first, b.first);
        }

    protected:
        explicit value_compare(Compare c) : comp_(std::move(c)) {}
        Compare comp_;
        friend class avl_map;
    };

    // =========================================================================
    // Iterator (bidirectional)
    // =========================================================================
    class const_iterator;   // forward declaration

    class iterator {
    public:
        using iterator_category = std::bidirectional_iterator_tag;
        using value_type        = avl_map::value_type;
        using difference_type   = avl_map::difference_type;
        using pointer           = avl_map::pointer;
        using reference         = avl_map::reference;

        iterator() = default;

        reference operator*()  const noexcept { return  node_->kv; }
        pointer   operator->() const noexcept { return &node_->kv; }

        iterator& operator++() noexcept {
            node_ = avl_map::next_node(node_);
            return *this;
        }
        iterator operator++(int) noexcept { auto tmp = *this; ++*this; return tmp; }

        iterator& operator--() noexcept {
            // --end() → max node
            node_ = node_ ? avl_map::prev_node(node_)
                           : avl_map::max_node(map_->root_);
            return *this;
        }
        iterator operator--(int) noexcept { auto tmp = *this; --*this; return tmp; }

        bool operator==(const iterator&) const noexcept = default;

    private:
        Node*          node_ = nullptr;
        const avl_map* map_  = nullptr;

        explicit iterator(Node* n, const avl_map* m) noexcept
            : node_(n), map_(m) {}

        friend class avl_map;
        friend class const_iterator;
    };

    class const_iterator {
    public:
        using iterator_category = std::bidirectional_iterator_tag;
        using value_type        = avl_map::value_type;
        using difference_type   = avl_map::difference_type;
        using pointer           = avl_map::const_pointer;
        using reference         = avl_map::const_reference;

        const_iterator() = default;

        // Implicit conversion from iterator
        // NOLINTNEXTLINE(google-explicit-constructor)
        const_iterator(iterator it) noexcept
            : node_(it.node_), map_(it.map_) {}

        reference operator*()  const noexcept { return  node_->kv; }
        pointer   operator->() const noexcept { return &node_->kv; }

        const_iterator& operator++() noexcept {
            node_ = avl_map::next_node(const_cast<Node*>(node_));
            return *this;
        }
        const_iterator operator++(int) noexcept { auto tmp = *this; ++*this; return tmp; }

        const_iterator& operator--() noexcept {
            node_ = node_ ? avl_map::prev_node(const_cast<Node*>(node_))
                           : avl_map::max_node(map_->root_);
            return *this;
        }
        const_iterator operator--(int) noexcept { auto tmp = *this; --*this; return tmp; }

        bool operator==(const const_iterator&) const noexcept = default;

    private:
        const Node*    node_ = nullptr;
        const avl_map* map_  = nullptr;

        explicit const_iterator(const Node* n, const avl_map* m) noexcept
            : node_(n), map_(m) {}

        friend class avl_map;
    };

    using reverse_iterator       = std::reverse_iterator<iterator>;
    using const_reverse_iterator = std::reverse_iterator<const_iterator>;

    // =========================================================================
    // node_type  (node handle — for extract / merge)
    // =========================================================================
    class node_type {
    public:
        using key_type       = avl_map::key_type;
        using mapped_type    = avl_map::mapped_type;
        using allocator_type = avl_map::allocator_type;

        node_type()                          = default;
        node_type(const node_type&)          = delete;
        node_type& operator=(const node_type&) = delete;

        node_type(node_type&& other) noexcept
            : node_(other.node_), alloc_(std::move(other.alloc_))
        { other.node_ = nullptr; }

        node_type& operator=(node_type&& other) noexcept {
            reset();
            node_  = other.node_;
            alloc_ = std::move(other.alloc_);
            other.node_ = nullptr;
            return *this;
        }

        ~node_type() { reset(); }

        [[nodiscard]] bool empty()       const noexcept { return node_ == nullptr; }
        explicit operator bool()         const noexcept { return node_ != nullptr; }

        // Note: std::map::node_type::key() returns a non-const reference so
        // that callers can alter the key before re-inserting the handle.
        // NOLINTNEXTLINE(cppcoreguidelines-pro-type-const-cast)
        key_type&    key()    const { return const_cast<key_type&>(node_->kv.first); }
        mapped_type& mapped() const { return node_->kv.second; }

        allocator_type get_allocator() const {
            assert(alloc_.has_value());
            return allocator_type(*alloc_);
        }

        void swap(node_type& other) noexcept {
            std::swap(node_, other.node_);
            std::swap(alloc_, other.alloc_);
        }

        friend void swap(node_type& a, node_type& b) noexcept { a.swap(b); }

    private:
        Node*                    node_  = nullptr;
        std::optional<NodeAlloc> alloc_;

        // Called by avl_map internals only
        node_type(Node* n, const NodeAlloc& a) noexcept
            : node_(n), alloc_(a)
        {
            if (node_) {
                node_->parent = nullptr;
                node_->left   = nullptr;
                node_->right  = nullptr;
                node_->height = 1;
            }
        }

        Node* release() noexcept {
            Node* tmp = node_;
            node_     = nullptr;
            return tmp;
        }

        void reset() noexcept {
            if (node_ && alloc_) {
                NodeTraits::destroy(*alloc_, node_);
                NodeTraits::deallocate(*alloc_, node_, 1);
            }
            node_ = nullptr;
        }

        friend class avl_map;
    };

    // =========================================================================
    // insert_return_type
    // =========================================================================
    struct insert_return_type {
        iterator  position;
        bool      inserted{};
        node_type node;
    };

    // =========================================================================
    // Constructors / destructor / assignment
    // =========================================================================
    avl_map() = default;

    explicit avl_map(const Compare& comp, const Allocator& alloc = Allocator())
        : comp_(comp), alloc_(NodeAlloc(alloc)) {}

    explicit avl_map(const Allocator& alloc)
        : alloc_(NodeAlloc(alloc)) {}

    template<std::input_iterator It>
    avl_map(It first, It last,
            const Compare& comp   = Compare(),
            const Allocator& alloc = Allocator())
        : comp_(comp), alloc_(NodeAlloc(alloc))
    { insert(first, last); }

    template<std::input_iterator It>
    avl_map(It first, It last, const Allocator& alloc)
        : alloc_(NodeAlloc(alloc))
    { insert(first, last); }

    avl_map(const avl_map& other)
        : comp_(other.comp_),
          alloc_(NodeTraits::select_on_container_copy_construction(other.alloc_))
    { insert(other.begin(), other.end()); }

    avl_map(const avl_map& other, const Allocator& alloc)
        : comp_(other.comp_), alloc_(NodeAlloc(alloc))
    { insert(other.begin(), other.end()); }

    avl_map(avl_map&& other) noexcept
        : root_(other.root_), size_(other.size_),
          comp_(std::move(other.comp_)), alloc_(std::move(other.alloc_))
    { other.root_ = nullptr; other.size_ = 0; }

    avl_map(avl_map&& other, const Allocator& alloc)
        : comp_(other.comp_), alloc_(NodeAlloc(alloc))
    {
        if (alloc_ == other.alloc_) {
            root_ = other.root_; size_ = other.size_;
            other.root_ = nullptr; other.size_ = 0;
        } else {
            for (auto& kv : other)
                emplace_impl(std::move(const_cast<value_type&>(kv)));
        }
    }

    avl_map(std::initializer_list<value_type> init,
            const Compare& comp   = Compare(),
            const Allocator& alloc = Allocator())
        : comp_(comp), alloc_(NodeAlloc(alloc))
    { insert(init.begin(), init.end()); }

    avl_map(std::initializer_list<value_type> init, const Allocator& alloc)
        : alloc_(NodeAlloc(alloc))
    { insert(init.begin(), init.end()); }

    ~avl_map() { clear(); }

    avl_map& operator=(const avl_map& other) {
        if (this == &other) return *this;
        clear();
        if constexpr (NodeTraits::propagate_on_container_copy_assignment::value)
            alloc_ = other.alloc_;
        comp_ = other.comp_;
        insert(other.begin(), other.end());
        return *this;
    }

    avl_map& operator=(avl_map&& other)
        noexcept(NodeTraits::is_always_equal::value &&
                 std::is_nothrow_move_assignable_v<Compare>)
    {
        if (this == &other) return *this;
        clear();
        comp_ = std::move(other.comp_);

        if constexpr (NodeTraits::propagate_on_container_move_assignment::value) {
            alloc_ = std::move(other.alloc_);
            root_ = other.root_; size_ = other.size_;
            other.root_ = nullptr; other.size_ = 0;
        } else if (alloc_ == other.alloc_) {
            root_ = other.root_; size_ = other.size_;
            other.root_ = nullptr; other.size_ = 0;
        } else {
            for (auto& kv : other)
                emplace_impl(std::move(const_cast<value_type&>(kv)));
        }
        return *this;
    }

    avl_map& operator=(std::initializer_list<value_type> ilist) {
        clear();
        insert(ilist.begin(), ilist.end());
        return *this;
    }

    allocator_type get_allocator() const noexcept {
        return allocator_type(alloc_);
    }

    // =========================================================================
    // Element access
    // =========================================================================
    T& at(const Key& key) {
        Node* n = find_node(key);
        if (!n) throw std::out_of_range("avl_map::at: key not found");
        return n->kv.second;
    }
    const T& at(const Key& key) const {
        const Node* n = find_node(key);
        if (!n) throw std::out_of_range("avl_map::at: key not found");
        return n->kv.second;
    }

    T& operator[](const Key& key)  { return try_emplace(key).first->second; }
    T& operator[](Key&& key)        { return try_emplace(std::move(key)).first->second; }

    // =========================================================================
    // Iterators
    // =========================================================================
    iterator       begin()  noexcept       { return it(min_node(root_)); }
    const_iterator begin()  const noexcept { return cit(min_node(root_)); }
    const_iterator cbegin() const noexcept { return begin(); }

    iterator       end()  noexcept       { return it(nullptr); }
    const_iterator end()  const noexcept { return cit(nullptr); }
    const_iterator cend() const noexcept { return end(); }

    reverse_iterator       rbegin()  noexcept       { return reverse_iterator(end()); }
    const_reverse_iterator rbegin()  const noexcept { return const_reverse_iterator(end()); }
    const_reverse_iterator crbegin() const noexcept { return rbegin(); }

    reverse_iterator       rend()  noexcept       { return reverse_iterator(begin()); }
    const_reverse_iterator rend()  const noexcept { return const_reverse_iterator(begin()); }
    const_reverse_iterator crend() const noexcept { return rend(); }

    // =========================================================================
    // Capacity
    // =========================================================================
    [[nodiscard]] bool empty()    const noexcept { return size_ == 0; }
    size_type          size()     const noexcept { return size_; }
    size_type          max_size() const noexcept { return NodeTraits::max_size(alloc_); }

    // =========================================================================
    // Modifiers
    // =========================================================================
    void clear() noexcept {
        destroy_tree(root_);
        root_ = nullptr;
        size_ = 0;
    }

    // ---- insert ----
    std::pair<iterator, bool> insert(const value_type& v)  { return do_insert(v); }
    std::pair<iterator, bool> insert(value_type&& v)        { return do_insert(std::move(v)); }

    template<typename P>
        requires std::constructible_from<value_type, P&&>
    std::pair<iterator, bool> insert(P&& v) { return do_insert(std::forward<P>(v)); }

    iterator insert(const_iterator /*hint*/, const value_type& v)  { return insert(v).first; }
    iterator insert(const_iterator /*hint*/, value_type&& v)        { return insert(std::move(v)).first; }

    template<typename P>
        requires std::constructible_from<value_type, P&&>
    iterator insert(const_iterator /*hint*/, P&& v) { return insert(std::forward<P>(v)).first; }

    template<std::input_iterator It>
    void insert(It first, It last) {
        for (; first != last; ++first) insert(*first);
    }

    void insert(std::initializer_list<value_type> ilist) {
        insert(ilist.begin(), ilist.end());
    }

    insert_return_type insert(node_type&& nh) {
        if (nh.empty()) return {end(), false, {}};
        Node* n = nh.node_;
        auto [existing, inserted] = do_insert_node(n);
        if (inserted) {
            nh.node_ = nullptr;
            return {it(existing), true, {}};
        }
        return {it(existing), false, std::move(nh)};
    }

    iterator insert(const_iterator /*hint*/, node_type&& nh) {
        return insert(std::move(nh)).position;
    }

    // ---- insert_or_assign ----
    template<typename M>
    std::pair<iterator, bool> insert_or_assign(const Key& k, M&& obj) {
        return do_insert_or_assign(k, std::forward<M>(obj));
    }
    template<typename M>
    std::pair<iterator, bool> insert_or_assign(Key&& k, M&& obj) {
        return do_insert_or_assign(std::move(k), std::forward<M>(obj));
    }
    template<typename M>
    iterator insert_or_assign(const_iterator, const Key& k, M&& obj) {
        return insert_or_assign(k, std::forward<M>(obj)).first;
    }
    template<typename M>
    iterator insert_or_assign(const_iterator, Key&& k, M&& obj) {
        return insert_or_assign(std::move(k), std::forward<M>(obj)).first;
    }

    // ---- emplace ----
    template<typename... Args>
    std::pair<iterator, bool> emplace(Args&&... args) {
        auto [n, ins] = emplace_impl(std::forward<Args>(args)...);
        return {it(n), ins};
    }

    template<typename... Args>
    iterator emplace_hint(const_iterator /*hint*/, Args&&... args) {
        return emplace(std::forward<Args>(args)...).first;
    }

    // ---- try_emplace ----
    template<typename... Args>
    std::pair<iterator, bool> try_emplace(const Key& k, Args&&... args) {
        return do_try_emplace(k, std::forward<Args>(args)...);
    }
    template<typename... Args>
    std::pair<iterator, bool> try_emplace(Key&& k, Args&&... args) {
        return do_try_emplace(std::move(k), std::forward<Args>(args)...);
    }
    template<typename... Args>
    iterator try_emplace(const_iterator, const Key& k, Args&&... args) {
        return try_emplace(k, std::forward<Args>(args)...).first;
    }
    template<typename... Args>
    iterator try_emplace(const_iterator, Key&& k, Args&&... args) {
        return try_emplace(std::move(k), std::forward<Args>(args)...).first;
    }

    // ---- erase ----
    iterator erase(iterator pos) {
        Node* n    = pos.node_;
        iterator next{it(next_node(n))};
        erase_node(n);
        return next;
    }
    iterator erase(const_iterator pos) {
        // NOLINTNEXTLINE(cppcoreguidelines-pro-type-const-cast)
        Node* n = const_cast<Node*>(pos.node_);
        iterator next{it(next_node(n))};
        erase_node(n);
        return next;
    }
    iterator erase(const_iterator first, const_iterator last) {
        while (first != last) first = erase(first);
        // NOLINTNEXTLINE(cppcoreguidelines-pro-type-const-cast)
        return it(const_cast<Node*>(last.node_));
    }
    size_type erase(const Key& key) {
        Node* n = find_node(key);
        if (!n) return 0;
        erase_node(n);
        return 1;
    }
    template<typename K>
        requires requires { typename Compare::is_transparent; }
    size_type erase(const K& key) {
        Node* n = find_node_t(key);
        if (!n) return 0;
        erase_node(n);
        return 1;
    }

    // ---- swap ----
    void swap(avl_map& other)
        noexcept(NodeTraits::is_always_equal::value &&
                 std::is_nothrow_swappable_v<Compare>)
    {
        std::swap(root_, other.root_);
        std::swap(size_, other.size_);
        std::swap(comp_, other.comp_);
        if constexpr (NodeTraits::propagate_on_container_swap::value)
            std::swap(alloc_, other.alloc_);
    }

    // ---- extract ----
    node_type extract(const_iterator pos) {
        // NOLINTNEXTLINE(cppcoreguidelines-pro-type-const-cast)
        return do_extract(const_cast<Node*>(pos.node_));
    }
    node_type extract(const Key& key) {
        return do_extract(find_node(key));
    }
    template<typename K>
        requires requires { typename Compare::is_transparent; }
    node_type extract(const K& key) {
        return do_extract(find_node_t(key));
    }

    // ---- merge ----
    template<typename C2>
    void merge(avl_map<Key, T, C2, Allocator>& source) {
        for (auto it_s = source.begin(); it_s != source.end(); ) {
            auto cur = it_s++;
            if (!contains(cur->first))
                insert(source.extract(cur));
        }
    }
    template<typename C2>
    void merge(avl_map<Key, T, C2, Allocator>&& source) { merge(source); }

    // =========================================================================
    // Lookup
    // =========================================================================
    size_type count(const Key& key) const noexcept {
        return find_node(key) ? 1 : 0;
    }
    template<typename K>
        requires requires { typename Compare::is_transparent; }
    size_type count(const K& key) const noexcept {
        return find_node_t(key) ? 1 : 0;
    }

    iterator       find(const Key& key)       noexcept { return it(find_node(key)); }
    const_iterator find(const Key& key) const noexcept { return cit(find_node(key)); }
    template<typename K>
        requires requires { typename Compare::is_transparent; }
    iterator       find(const K& key)       noexcept { return it(find_node_t(key)); }
    template<typename K>
        requires requires { typename Compare::is_transparent; }
    const_iterator find(const K& key) const noexcept { return cit(find_node_t(key)); }

    bool contains(const Key& key) const noexcept { return find_node(key) != nullptr; }
    template<typename K>
        requires requires { typename Compare::is_transparent; }
    bool contains(const K& key) const noexcept { return find_node_t(key) != nullptr; }

    std::pair<iterator,iterator>             equal_range(const Key& key)       noexcept { return {lower_bound(key), upper_bound(key)}; }
    std::pair<const_iterator,const_iterator> equal_range(const Key& key) const noexcept { return {lower_bound(key), upper_bound(key)}; }
    template<typename K> requires requires { typename Compare::is_transparent; }
    std::pair<iterator,iterator>             equal_range(const K& key)       noexcept { return {lower_bound(key), upper_bound(key)}; }
    template<typename K> requires requires { typename Compare::is_transparent; }
    std::pair<const_iterator,const_iterator> equal_range(const K& key) const noexcept { return {lower_bound(key), upper_bound(key)}; }

    iterator       lower_bound(const Key& key)       noexcept { return it(lower_bound_node(key)); }
    const_iterator lower_bound(const Key& key) const noexcept { return cit(lower_bound_node(key)); }
    template<typename K> requires requires { typename Compare::is_transparent; }
    iterator       lower_bound(const K& key)       noexcept { return it(lower_bound_node_t(key)); }
    template<typename K> requires requires { typename Compare::is_transparent; }
    const_iterator lower_bound(const K& key) const noexcept { return cit(lower_bound_node_t(key)); }

    iterator       upper_bound(const Key& key)       noexcept { return it(upper_bound_node(key)); }
    const_iterator upper_bound(const Key& key) const noexcept { return cit(upper_bound_node(key)); }
    template<typename K> requires requires { typename Compare::is_transparent; }
    iterator       upper_bound(const K& key)       noexcept { return it(upper_bound_node_t(key)); }
    template<typename K> requires requires { typename Compare::is_transparent; }
    const_iterator upper_bound(const K& key) const noexcept { return cit(upper_bound_node_t(key)); }

    // =========================================================================
    // Observers
    // =========================================================================
    key_compare   key_comp()   const { return comp_; }
    value_compare value_comp() const { return value_compare(comp_); }

    // =========================================================================
    // Non-member functions (as hidden friends)
    // =========================================================================
    friend bool operator==(const avl_map& a, const avl_map& b) {
        return a.size_ == b.size_ &&
               std::equal(a.begin(), a.end(), b.begin());
    }

    friend auto operator<=>(const avl_map& a, const avl_map& b) {
        return std::lexicographical_compare_three_way(
            a.begin(), a.end(), b.begin(), b.end());
    }

    friend void swap(avl_map& a, avl_map& b) noexcept(noexcept(a.swap(b))) {
        a.swap(b);
    }

    template<typename Pred>
    friend size_type erase_if(avl_map& c, Pred pred) {
        size_type old_size = c.size();
        for (auto i = c.begin(); i != c.end(); ) {
            if (pred(*i)) i = c.erase(i);
            else          ++i;
        }
        return old_size - c.size();
    }

    // =========================================================================
    // Private state
    // =========================================================================
private:
    Node*     root_ = nullptr;
    size_type size_ = 0;
    Compare   comp_{};
    NodeAlloc alloc_{};

    // =========================================================================
    // Iterator factory helpers (keep iterator ctor private)
    // =========================================================================
    iterator       it(Node* n)        noexcept { return iterator(n, this); }
    const_iterator cit(const Node* n) const noexcept { return const_iterator(n, this); }

    // =========================================================================
    // AVL helpers — height & balance
    // =========================================================================
    [[nodiscard]] static int ht(const Node* n) noexcept { return n ? n->height : 0; }

    static void refresh_height(Node* n) noexcept {
        if (n) n->height = 1 + std::max(ht(n->left), ht(n->right));
    }

    [[nodiscard]] static int balance_factor(const Node* n) noexcept {
        return ht(n->right) - ht(n->left);
    }

    // =========================================================================
    // Rotations
    // All rotations fix parent pointers and update heights, then update the
    // parent's child pointer (or root_) so the caller doesn't have to.
    // Returns the new local root after rotation.
    // =========================================================================

    //         y                  x
    //        /|                 /|
    //       x   C   =>         A  y
    //      /|                    /|
    //     A   B                 B  C
    Node* rotate_right(Node* y) noexcept {
        Node* x  = y->left;
        Node* B  = x->right;

        // Perform rotation
        x->right = y;
        y->left  = B;

        // Fix parent links
        x->parent = y->parent;
        y->parent = x;
        if (B) B->parent = y;

        // Fix grandparent's child pointer
        link_to_parent(x, y);

        refresh_height(y);
        refresh_height(x);
        return x;
    }

    //     x                   y
    //    /|                  /|
    //   A   y      =>       x  C
    //      /|              /|
    //     B   C           A  B
    Node* rotate_left(Node* x) noexcept {
        Node* y  = x->right;
        Node* B  = y->left;

        y->left  = x;
        x->right = B;

        y->parent = x->parent;
        x->parent = y;
        if (B) B->parent = x;

        link_to_parent(y, x);

        refresh_height(x);
        refresh_height(y);
        return y;
    }

    // After a rotation, update the grandparent's child pointer (or root_).
    // new_child is the new local root; old_child is what it replaced.
    void link_to_parent(Node* new_child, Node* old_child) noexcept {
        Node* gp = new_child->parent;
        if (!gp)                      root_        = new_child;
        else if (gp->left == old_child) gp->left  = new_child;
        else                            gp->right = new_child;
    }

    // Rebalance a single node and return the (possibly new) local root.
    Node* rebalance_node(Node* n) noexcept {
        refresh_height(n);
        const int bf = balance_factor(n);

        if (bf < -1) {
            // Left-heavy
            if (balance_factor(n->left) > 0)
                rotate_left(n->left);     // Left-Right case: first rotate child
            return rotate_right(n);
        }
        if (bf > 1) {
            // Right-heavy
            if (balance_factor(n->right) < 0)
                rotate_right(n->right);   // Right-Left case: first rotate child
            return rotate_left(n);
        }
        return n;
    }

    // Walk from n up to the root, rebalancing as needed.
    void rebalance_up(Node* n) noexcept {
        while (n) {
            Node* parent = n->parent;
            rebalance_node(n);
            n = parent;
        }
    }

    // =========================================================================
    // Tree traversal helpers
    // =========================================================================
    [[nodiscard]] static Node* min_node(Node* n) noexcept {
        if (!n) return nullptr;
        while (n->left) n = n->left;
        return n;
    }

    [[nodiscard]] static Node* max_node(Node* n) noexcept {
        if (!n) return nullptr;
        while (n->right) n = n->right;
        return n;
    }

    // In-order successor (nullptr if n is the maximum node)
    [[nodiscard]] static Node* next_node(Node* n) noexcept {
        assert(n);
        if (n->right) return min_node(n->right);
        Node* p = n->parent;
        while (p && n == p->right) { n = p; p = p->parent; }
        return p;
    }

    // In-order predecessor (undefined if n is the minimum node)
    [[nodiscard]] static Node* prev_node(Node* n) noexcept {
        assert(n);
        if (n->left) return max_node(n->left);
        Node* p = n->parent;
        while (p && n == p->left) { n = p; p = p->parent; }
        return p;
    }

    // =========================================================================
    // Lookup helpers
    // =========================================================================
    [[nodiscard]] Node* find_node(const Key& key) const noexcept {
        Node* cur = root_;
        while (cur) {
            if      (comp_(key,           cur->kv.first)) cur = cur->left;
            else if (comp_(cur->kv.first, key          )) cur = cur->right;
            else                                           return cur;
        }
        return nullptr;
    }

    template<typename K>
    [[nodiscard]] Node* find_node_t(const K& key) const noexcept {
        Node* cur = root_;
        while (cur) {
            if      (comp_(key,           cur->kv.first)) cur = cur->left;
            else if (comp_(cur->kv.first, key          )) cur = cur->right;
            else                                           return cur;
        }
        return nullptr;
    }

    [[nodiscard]] Node* lower_bound_node(const Key& key) const noexcept {
        Node* cur = root_, *result = nullptr;
        while (cur) {
            if (!comp_(cur->kv.first, key)) { result = cur; cur = cur->left; }
            else                              cur = cur->right;
        }
        return result;
    }

    [[nodiscard]] Node* upper_bound_node(const Key& key) const noexcept {
        Node* cur = root_, *result = nullptr;
        while (cur) {
            if (comp_(key, cur->kv.first)) { result = cur; cur = cur->left; }
            else                             cur = cur->right;
        }
        return result;
    }

    template<typename K>
    [[nodiscard]] Node* lower_bound_node_t(const K& key) const noexcept {
        Node* cur = root_, *result = nullptr;
        while (cur) {
            if (!comp_(cur->kv.first, key)) { result = cur; cur = cur->left; }
            else                              cur = cur->right;
        }
        return result;
    }

    template<typename K>
    [[nodiscard]] Node* upper_bound_node_t(const K& key) const noexcept {
        Node* cur = root_, *result = nullptr;
        while (cur) {
            if (comp_(key, cur->kv.first)) { result = cur; cur = cur->left; }
            else                             cur = cur->right;
        }
        return result;
    }

    // =========================================================================
    // Allocator helpers
    // =========================================================================
    template<typename... Args>
    [[nodiscard]] Node* create_node(Args&&... args) {
        Node* n = NodeTraits::allocate(alloc_, 1);
        try {
            NodeTraits::construct(alloc_, n, std::forward<Args>(args)...);
        } catch (...) {
            NodeTraits::deallocate(alloc_, n, 1);
            throw;
        }
        return n;
    }

    void destroy_node(Node* n) noexcept {
        NodeTraits::destroy(alloc_, n);
        NodeTraits::deallocate(alloc_, n, 1);
    }

    void destroy_tree(Node* n) noexcept {
        if (!n) return;
        destroy_tree(n->left);
        destroy_tree(n->right);
        destroy_node(n);
    }

    // =========================================================================
    // Core insertion logic
    // Constructs a node in-place, then locates its position.  If the key
    // already exists the temporary node is destroyed and the existing node is
    // returned with inserted=false.
    // =========================================================================
    template<typename... Args>
    std::pair<Node*, bool> emplace_impl(Args&&... args) {
        // Eagerly construct the node so that we have the key available;
        // destroy it if the key already exists.
        Node* tmp = create_node(std::forward<Args>(args)...);
        const Key& key = tmp->kv.first;

        Node* cur = root_, *parent = nullptr;
        bool  go_left = false;

        while (cur) {
            if      (comp_(key,           cur->kv.first)) { parent = cur; go_left = true;  cur = cur->left;  }
            else if (comp_(cur->kv.first, key          )) { parent = cur; go_left = false; cur = cur->right; }
            else {
                destroy_node(tmp);
                return {cur, false};
            }
        }

        link_new_node(tmp, parent, go_left);
        return {tmp, true};
    }

    // ---- insert wrappers ----
    template<typename V>
    std::pair<iterator, bool> do_insert(V&& v) {
        auto [n, ins] = emplace_impl(std::forward<V>(v));
        return {it(n), ins};
    }

    // ---- insert_or_assign ----
    template<typename K2, typename M>
    std::pair<iterator, bool> do_insert_or_assign(K2&& key, M&& obj) {
        Node* cur = root_, *parent = nullptr;
        bool  go_left = false;
        while (cur) {
            if      (comp_(key,           cur->kv.first)) { parent = cur; go_left = true;  cur = cur->left;  }
            else if (comp_(cur->kv.first, key          )) { parent = cur; go_left = false; cur = cur->right; }
            else {
                cur->kv.second = std::forward<M>(obj);
                return {it(cur), false};
            }
        }
        Node* n = create_node(std::piecewise_construct,
                              std::forward_as_tuple(std::forward<K2>(key)),
                              std::forward_as_tuple(std::forward<M>(obj)));
        link_new_node(n, parent, go_left);
        return {it(n), true};
    }

    // ---- try_emplace ----
    template<typename K2, typename... Args>
    std::pair<iterator, bool> do_try_emplace(K2&& key, Args&&... args) {
        Node* cur = root_, *parent = nullptr;
        bool  go_left = false;
        while (cur) {
            if      (comp_(key,           cur->kv.first)) { parent = cur; go_left = true;  cur = cur->left;  }
            else if (comp_(cur->kv.first, key          )) { parent = cur; go_left = false; cur = cur->right; }
            else return {it(cur), false};
        }
        Node* n = create_node(std::piecewise_construct,
                              std::forward_as_tuple(std::forward<K2>(key)),
                              std::forward_as_tuple(std::forward<Args>(args)...));
        link_new_node(n, parent, go_left);
        return {it(n), true};
    }

    // ---- shared finalization for newly created nodes ----
    void link_new_node(Node* n, Node* parent, bool go_left) noexcept {
        n->parent = parent;
        if      (!parent)  root_         = n;
        else if (go_left)  parent->left  = n;
        else               parent->right = n;
        ++size_;
        rebalance_up(parent);
    }

    // ---- insert a pre-existing node handle ----
    std::pair<Node*, bool> do_insert_node(Node* new_node) {
        const Key& key = new_node->kv.first;
        Node* cur = root_, *parent = nullptr;
        bool  go_left = false;
        while (cur) {
            if      (comp_(key,           cur->kv.first)) { parent = cur; go_left = true;  cur = cur->left;  }
            else if (comp_(cur->kv.first, key          )) { parent = cur; go_left = false; cur = cur->right; }
            else return {cur, false};
        }
        new_node->parent = nullptr;
        new_node->left   = nullptr;
        new_node->right  = nullptr;
        new_node->height = 1;
        link_new_node(new_node, parent, go_left);
        return {new_node, true};
    }

    // =========================================================================
    // Erasure
    //
    // When a node has two children we cannot simply move the successor's data
    // into n (because Key is const).  Instead we perform a structural swap:
    // we physically move n and its in-order successor in the tree, then remove
    // the (now single-child) node from its new position.
    // =========================================================================

    // Splice n out of the tree without destroying it (used by erase and extract).
    void unlink_node(Node* n) noexcept {
        if (n->left && n->right) {
            // n has two children → swap n with its in-order successor
            Node* s = min_node(n->right);
            structural_swap(n, s);
            // After the swap n is guaranteed to have at most one child (right)
        }

        // n now has at most one child
        Node* child  = n->left ? n->left : n->right;
        Node* parent = n->parent;

        if (child) child->parent = parent;

        if      (!parent)           root_         = child;
        else if (parent->left == n) parent->left  = child;
        else                        parent->right = child;

        rebalance_up(parent);

        // Clean the node's links so it can be used as a fresh node handle
        n->parent = nullptr;
        n->left   = nullptr;
        n->right  = nullptr;
        n->height = 1;
    }

    void erase_node(Node* n) noexcept {
        assert(n);
        --size_;
        unlink_node(n);
        destroy_node(n);
    }

    node_type do_extract(Node* n) {
        if (!n) return {};
        --size_;
        unlink_node(n);
        return node_type(n, alloc_);
    }

    // Swap two nodes' positions in the tree (structural, not value swap).
    // n's in-order successor is s.  s has no left child.
    void structural_swap(Node* n, Node* s) noexcept {
        assert(s->left == nullptr);

        Node* n_parent = n->parent;
        Node* n_left   = n->left;
        Node* n_right  = n->right;    // s is somewhere in this subtree
        Node* s_parent = s->parent;
        Node* s_right  = s->right;

        if (s_parent == n) {
            // s is n's direct right child
            s->left   = n_left;
            s->right  = n;
            s->parent = n_parent;

            n->parent = s;
            n->left   = nullptr;
            n->right  = s_right;
            if (s_right) s_right->parent = n;
        } else {
            // General case: s_parent != n
            s_parent->left = n;
            n->parent      = s_parent;

            s->left   = n_left;
            s->right  = n_right;
            s->parent = n_parent;

            n->left   = nullptr;
            n->right  = s_right;
            if (s_right) s_right->parent = n;
            if (n_right) n_right->parent = s;
        }

        if (n_left)  n_left->parent  = s;

        if (!s->parent)                s_parent_update(s, n_parent, n);
        else if (s->parent->left == n) /* already set above, n_parent handled below */ {}

        // Fix grandparent link
        if      (!n_parent)               root_             = s;
        else if (n_parent->left == n)     n_parent->left   = s;
        else                              n_parent->right  = s;

        refresh_height(n);
        refresh_height(s);
    }

    // Helper to avoid a warning; updates the parent after structural_swap.
    static void s_parent_update(Node*, Node*, Node*) noexcept {}
};

// =============================================================================
// Deduction guides (C++17+, included for completeness)
// =============================================================================
template<
    std::input_iterator It,
    typename Compare   = std::less<typename std::iterator_traits<It>::value_type::first_type>,
    typename Allocator = std::allocator<typename std::iterator_traits<It>::value_type>
>
avl_map(It, It, Compare = Compare(), Allocator = Allocator())
    -> avl_map<
           typename std::iterator_traits<It>::value_type::first_type,
           typename std::iterator_traits<It>::value_type::second_type,
           Compare,
           Allocator>;

template<
    typename Key,
    typename T,
    typename Compare   = std::less<Key>,
    typename Allocator = std::allocator<std::pair<const Key, T>>
>
avl_map(std::initializer_list<std::pair<const Key,T>>,
        Compare   = Compare(),
        Allocator = Allocator())
    -> avl_map<Key, T, Compare, Allocator>;
```


# Evaluation: C++23 ISO Standard & C++ Core Guidelines

## 1. Correctness Bugs

### 1.1 Dead code in `structural_swap` masks a latent hazard

`structural_swap` contains a no-op function and a dead branch that look like they should do something:

```cpp
// ① s_parent_update does literally nothing:
static void s_parent_update(Node*, Node*, Node*) noexcept {}

// ② This else-if branch is always silently skipped:
if (!s->parent)                s_parent_update(s, n_parent, n);
else if (s->parent->left == n) /* already set above, n_parent handled below */ {}
```

The grandparent link is correctly fixed by the final block:

```cpp
if      (!n_parent)           root_          = s;
else if (n_parent->left == n) n_parent->left  = s;
else                          n_parent->right = s;
```

The dead code is harmless now, but it constitutes a serious maintenance hazard. 
It strongly implies that `s_parent_update` is meant to perform an update, and that the `else if` branch handles a real case — a future maintainer could delete the "redundant" final block and silently break the tree. 
This violates **C++ Core Guidelines P.1** (express ideas directly in code) and **P.3** (express intent).


### 1.2 `rebalance_up` saves the parent pointer before the rotation

```cpp
void rebalance_up(Node* n) noexcept {
    while (n) {
        Node* parent = n->parent;  // saved before rotation …
        rebalance_node(n);         // … rotation changes n->parent
        n = parent;                // … but we still use the saved value
    }
}
```

This is actually correct: `parent` is n's grandparent (the node above the newly rotated subtree), which is exactly where the next rebalance step should begin. It is subtle enough to deserve a comment, and the absence of one is a documentation bug that risks future "fixes" that actually break things.

### 1.3 `operator<=>` has a stronger constraint than std::map

```cpp
friend auto operator<=>(const avl_map& a, const avl_map& b) {
    return std::lexicographical_compare_three_way(
        a.begin(), a.end(), b.begin(), b.end());
}
```

`std::lexicographical_compare_three_way` requires `std::three_way_comparable` on `value_type`. 
`std::map`'s comparison only requires `LessThanComparable` on the element type. 
Any type that provides `operator<` but not `operator<=>` will fail to compile with this `avl_map` even though it would work with `std::map`. 
Per **[alg.3way]** the return type also depends on the element's comparison category, which may surprise callers expecting `std::strong_ordering`.

The fix is to add an explicit constraint or fall back to a synthesised three-way comparison from `operator<`.

### 1.4 Height update order after structural_swap is inverted

```cpp
refresh_height(n);   // n is now at the *lower* position (successor's old slot)
refresh_height(s);   // s is now at the *upper* position (n's old slot)
```

The order is correct (bottom-up), but `s`'s height is recomputed from `n_right` whose subtree still contains all its original descendants. 
This is fine, but these two calls are then immediately redundant: `rebalance_up(parent)` in `unlink_node` recomputes every height on the path to the root anyway. 
The redundant calls are harmless but add confusion.

## 2. API / Standard Conformance Deviations

### 2.1 All hint parameters are ignored

```cpp
iterator insert(const_iterator /*hint*/, const value_type& v) { return insert(v).first; }
iterator emplace_hint(const_iterator /*hint*/, Args&&... args) { ... }
```

The C++ standard specifies that a valid hint may reduce insertion complexity to amortised O(1). 
Silently ignoring the hint is legal, but it is a significant performance regression for callers who provide an immediately-preceding iterator (a common pattern for bulk construction). 
The deviation should at minimum be documented.

## 2.2 `merge` only accepts `avl_map`, not any compatible associative container

`std::map::merge` accepts any compatible node-based sorted associative container (`std::map`, `std::multimap` with the same key/value/allocator). 
This implementation only accepts another `avl_map` with a different comparator:

```cpp
template<typename C2>
void merge(avl_map<Key, T, C2, Allocator>& source);
```

Merging from a `std::map` is impossible, which breaks drop-in compatibility.

### 2.3 Missing `requires` on `operator<=>`
The hidden friend `operator<=>` will silently fail to instantiate for types that have no `<=>`. 
Per **[temp.req]** and **C++ Core Guidelines T.10**, a concept constraint should be added to surface this early:

```cpp
friend auto operator<=>(const avl_map& a, const avl_map& b)
    requires std::three_way_comparable<value_type>;
```

## 3. C++ Core Guidelines Violations

### 3.1 `const_cast` to call non-const static helpers (CP/ES guidelines)

`const_iterator` must `const_cast` its stored `const Node*` to call `next_node` / `prev_node`, which take `Node*`:

```cpp
const_iterator& operator++() noexcept {
    node_ = avl_map::next_node(const_cast<Node*>(node_));
    return *this;
}
```

The fix is straightforward: make `next_node` and `prev_node` take `const Node*` and return `const Node*`. 
**C++ Core Guidelines ES.50** prohibits `const_cast` to remove `const` from something that was originally `const`. 
While `node_` itself is `const Node*`, the fact that the function modifies no state means the cast is safe here, but it is unnecessary and triggers linter warnings.

### 3.2 `node_type::key()` silently discards `const` (C++ Core Guidelines I.3)

```cpp
key_type& key() const {
    return const_cast<key_type&>(node_->kv.first);  // NOLINT
}
```

The `NOLINT` annotation silences the warning but does not remove the design smell. 
The correct way to handle this in C++23 is to store the key separately in `node_type` (mutable) and write it back on release, 
or to rely on the fact that an extracted node is outside any container and its key field was never declared `const` — 
only the `pair::first` member alias is `const`. A cleaner design separates key storage from the in-tree representation.

### 3.3 Recursive `destroy_tree` (C++ Core Guidelines R.11, ES.76)

```cpp
void destroy_tree(Node* n) noexcept {
    if (!n) return;
    destroy_tree(n->left);
    destroy_tree(n->right);
    destroy_node(n);
}
```

For an AVL tree the height is bounded by ~1.44 log₂ n, so stack overflow is not a practical risk (≈ 44 frames for a billion elements). 
However, the function is marked `noexcept` and calls itself recursively. 
If `destroy_node` ever throws (it currently cannot, but a custom allocator might), the `noexcept` will call `std::terminate` rather than propagating. 
An iterative post-order traversal using the existing parent pointers is both safer and idiomatic.

### 3.4 `emplace_impl` eagerly constructs, then may immediately destroy (P.9, ES.5)

```cpp
Node* tmp = create_node(std::forward<Args>(args)...);  // allocate + construct
const Key& key = tmp->kv.first;
// … search …
if (key_already_exists) {
    destroy_node(tmp);                                  // destruct + deallocate
    return {cur, false};
}
```

For a duplicate key, a full construction followed by a full destruction occurs. 
More critically, if `Args...` contains moved-from references, the caller's data is consumed even when no insertion takes place — 
a violation of **P.9** (don't waste time or space) and a semantic surprise. 
The idiomatic fix is to search for the insertion position first; 
if no duplicate is found, construct the node in place. 
This is how `std::map::try_emplace` avoids the issue.

### 3.5 Missing `[[nodiscard]]` on several pure query functions

`find`, `contains`, `count`, `lower_bound`, `upper_bound`, `equal_range`, `key_comp`, `value_comp`, and `get_allocator` all 
return values that callers virtually always intend to use. Omitting `[[nodiscard]]` is a minor but consistent violation of **C++ Core Guidelines I.11**.

## 4. Style / Minor Issues

| Issue | Guideline |
|-------|-----------|
| `// NOLINTNEXTLINE` used to suppress rather than fix root cause | P.1 |
| `go_left` bool flag in `emplace_impl` is uninitialised if `cur == root_` is never null — actually default-constructed to `false`, safe but fragile | ES.20 |
| `s_parent_update` helper parameter names are anonymous — intention is opaque | NL.8 | 
| Hint-based `insert` overloads comment says `/*hint*/` — no note in the documentation that the hint is not used | I.1 |
| `avl_map(avl_map&& other, const Allocator& alloc)` uses alloc_ before it is assigned when allocators differ; this is fine due to initialisation order but reads confusingly | ES.20 |

# Summary
The implementation is functionally correct for typical usage: 
the rotation logic, rebalancing traversal, iterator traversal via parent links, transparent comparisons, node handles, 
and allocator-traits propagation are all implemented properly. The most serious issues are:

+ **The dead code in** `structural_swap` — it cannot be explained by a comment; the function that is called does nothing, and the branch that should update a pointer does nothing. This is a latent correctness time-bomb.
+ `operator<=>` imposes a stronger requirement than `std::map` without advertising it.
+ `emplace_impl` **consuming moved-from arguments** on a duplicate-key path violates caller expectations.
+ **Hints are silently ignored**, a significant performance deviation from the standard's intent.
  
