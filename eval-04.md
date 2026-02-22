# A global variable with a managed lifetime in C++23

## Claude Prompt

Claude Model: **Sonnet 4.6**

In the context of C++23 ISO Standard and C++ Core Guidelines evaluate the following program 

```cpp
#include <memory>
#include <mutex>
#include <stdexcept>
template<typename B>
requires std::is_destructible_v<B>
class global {
public:
    global() = default;
    ~global() noexcept = default;
    global(const global&) = delete;
    global(global&&) = delete;
    global& operator=(const global&) = delete;
    global& operator=(global&&) = delete;
    template<std::derived_from<B> D = B, typename... Args>
    requires std::constructible_from<D, Args...>
    void init(Args&&... args) {
        std::lock_guard<std::mutex> lock(mtx);
        if (ptr)
            throw std::runtime_error("global already initialized");
        ptr = std::make_unique<D>(std::forward<Args>(args)...);
    }
    void reset() noexcept {
        std::lock_guard<std::mutex> lock(mtx);
        ptr.reset();
    }
    template<typename F>
    auto with(F&& func) -> decltype(func(std::declval<B&>())) {
        std::lock_guard<std::mutex> lock(mtx);
        if (!ptr)
            throw std::runtime_error("global not initialized");
        return func(*ptr);
    }
    bool exists() const noexcept {
        std::lock_guard<std::mutex> lock(mtx);
        return ptr != nullptr;
    }
private:
    std::unique_ptr<B> ptr;
    mutable std::mutex mtx;
};
```

## Claude is thinking

```
Scrutinized thread-safe singleton pattern against C++23 standards and guidelines

The user is asking me to evaluate a C++ program in the context of C++23 ISO Standard and C++ Core Guidelines. Let me analyze this code carefully.

The code defines a template class global that appears to be a thread-safe wrapper for a singleton-like object. Let me break down the key aspects:

    Design: It's a non-copyable, non-movable class that holds a unique_ptr to a polymorphic base class B.
    Thread safety: Uses mutex for synchronization
    Initialization: Template method init that can construct derived types
    Access: with method for accessing the object safely
    Lifecycle: reset to destroy the object, exists to check if initialized

Now let me evaluate this against C++23 standards and Core Guidelines:

Issues to consider:

    C.129: When designing a class, define copy, move, or destructors together - This is mostly followed, but there are some considerations.
    Mutex in const member function: The exists() function is marked const and locks a mutable mutex. This is technically correct but can be confusing.
    Return type deduction in with: Uses decltype(func(std::declval<B&>())) which is a bit verbose. In C++23, we could potentially simplify this.
    Exception safety: The code throws exceptions while holding locks, which is generally safe since lock_guard will unlock in its destructor, but worth noting.
    CP.111: Use a conventional pattern if you really need double-checked locking - Not applicable here as there's no double-checked locking.
    CP.20: Use RAII, never plain lock()/unlock() - Followed correctly with lock_guard.
    Thread safety of destruction: The destructor doesn't lock the mutex. This could be problematic if other threads are still using the object.
    Perfect forwarding and parameter passing - The template uses appropriate forwarding patterns. The noexcept specification on reset is questionable since mutex operations could theoretically fail, and the class intentionally prevents move operations despite unique_ptr being movable, which aligns with the singleton-like design pattern.
    Const correctness and thread safety - The exists() method is properly const, though the mutable mutex approach, while technically correct, adds some complexity. There's a potential race condition during destruction if another thread accesses the object while it's being torn down, and the exists() check creates a TOCTOU vulnerability where the pointer could become invalid between validation and use.
    Modern C++ features - The code appropriately uses noexcept specifications and leverages C++20/23 concepts like std::is_destructible_v, std::derived_from, and std::constructible_from with well-designed requires clauses for template constraints.
    Return value semantics - The with function returns by value and should benefit from RVO/NRVO optimization in typical scenarios.

Done

I'll evaluate this program against C++23 standards and the C++ Core Guidelines, highlighting both strengths and issues.
```

# Evaluation of the C++ Program

## ✅ Strengths

### 1.Good use of C++20/23 features
+ Proper use of concepts (`std::derived_from`, `std::constructible_from`)
+ Constraints with `requires` clauses
+ Perfect forwarding with variadic templates


### 2.RAII compliance (CP.20)
+ Uses `std::lock_guard` instead of manual lock/unlock
+ Uses `std::unique_ptr` for automatic memory management


### 3.Rule of Five/Zero (C.21)
+ Explicitly deletes copy/move operations (appropriate for singleton-like pattern)
+ Default constructor and destructor


### 4.Thread-safe operations
+ Most methods properly lock the mutex



## ⚠️ Issues & Concerns

### Critical: Destructor Race Condition (CP.20, CP.43)

```cpp
~global() noexcept = default;  // ❌ No mutex lock!
```

**Problem**: If another thread calls `with()`, `reset()`, or `exists()` while the object is being destroyed, undefined behavior occurs.

**Fix**:
```cpp
~global() noexcept {
    std::lock_guard<std::mutex> lock(mtx);
    ptr.reset();
}
```

### **Questionable**: `noexcept` on `reset()`

```cpp
void reset() noexcept {
    std::lock_guard<std::mutex> lock(mtx);  // Can throw!
```

**Issue**: Mutex operations can throw exceptions in rare cases (resource exhaustion). Marking `noexcept` will call `std::terminate` if this happens.

**Recommendation**: Remove `noexcept` or document the assumption.

### TOCTOU Vulnerability (CP.110)

```cpp
bool exists() const noexcept;
```

**Problem**: Classic time-of-check-time-of-use race:
```cpp
if (g.exists()) {
    g.with([](B& b) { /* use b */ });  // ❌ Another thread might reset() here
}
```

**Better pattern**: Let `with()` return optional or use exception handling, eliminating the need for `exists()`.

### Const Correctness with Mutable (C.12)
```cpp
bool exists() const noexcept {
    std::lock_guard<std::mutex> lock(mtx);  // mutable mutex
```
**Issue**: While technically correct, `mutable` mutex in const methods can be confusing. The logical state doesn't change, but physical state does.

**Opinion**: This is acceptable for synchronization primitives but should be documented.


### Return Type Deduction (F.15)
```cpp
auto with(F&& func) -> decltype(func(std::declval<B&>()))
```
Modern alternative (C++20+):
```cpp
auto with(F&& func) -> std::invoke_result_t<F, B&>
```
More readable and aligns with standard library conventions.

### Exception Safety (E.15, E.16)
The code throws while holding locks (safe due to RAII), but consider:
+ `init()` can throw after partial initialization
+ No way to distinguish "not initialized" from "initialization failed"

### Missing: Thread-safe Singleton Pattern
If this is meant to be a singleton, consider C++11's thread-safe static initialization:
```cpp
static global<B>& instance() {
    static global<B> inst;
    return inst;
}
```

## 📋 Improved Version

```cpp
#include <memory>
#include <mutex>
#include <stdexcept>
#include <optional>
#include <type_traits>

template<typename B>
requires std::is_destructible_v<B>
class global {
public:
    global() = default;
    
    ~global() noexcept {
        // Fix: Lock mutex during destruction
        std::lock_guard<std::mutex> lock(mtx);
        ptr.reset();
    }
    
    global(const global&) = delete;
    global(global&&) = delete;
    global& operator=(const global&) = delete;
    global& operator=(global&&) = delete;
    
    template<std::derived_from<B> D = B, typename... Args>
    requires std::constructible_from<D, Args...>
    void init(Args&&... args) {
        std::lock_guard<std::mutex> lock(mtx);
        if (ptr)
            throw std::runtime_error("global already initialized");
        ptr = std::make_unique<D>(std::forward<Args>(args)...);
    }
    
    void reset() {  // Removed noexcept
        std::lock_guard<std::mutex> lock(mtx);
        ptr.reset();
    }
    
    // Improved: Returns optional instead of throwing
    template<typename F>
    auto with(F&& func) -> std::optional<std::invoke_result_t<F, B&>> {
        std::lock_guard<std::mutex> lock(mtx);
        if (!ptr)
            return std::nullopt;
        return func(*ptr);
    }
    
    // Alternative: Keep throwing version if preferred
    template<typename F>
    auto with_unchecked(F&& func) -> std::invoke_result_t<F, B&> {
        std::lock_guard<std::mutex> lock(mtx);
        if (!ptr)
            throw std::runtime_error("global not initialized");
        return func(*ptr);
    }
    
    // Consider: Is this really needed given TOCTOU issues?
    [[nodiscard]] bool exists() const {  // Removed noexcept, added nodiscard
        std::lock_guard<std::mutex> lock(mtx);
        return ptr != nullptr;
    }

private:
    std::unique_ptr<B> ptr;
    mutable std::mutex mtx;
};
```

## 📊 Core Guidelines Summary
| Guideline | Status | Notes |
|-----------|--------|-------|
| C.21 (Rule of Five)                | ✅ | Properly defined |
| CP.20 (Use RAII)                   | ⚠️ | Destructor issue |
| CP.43 (Minimize critical sections) | ✅ | Good |
| E.16 (Destructors noexcept)        | ⚠️ | Needs mutex lock |
| F.15 (Pass info conventionally)    | ✅ | Good use of forwarding |
| C.12 (const correctness)           | ⚠️ | Mutable mutex acceptable |

# 🎯 Overall Assessment
The code demonstrates good modern C++ practices but has a **critical bug** in the destructor that could cause undefined behavior in multithreaded scenarios. 
After fixing this and the `noexcept` issues, the code would be production-ready for its intended use case.
