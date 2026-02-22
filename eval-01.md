# A global variable with a managed lifetime

## Claude Prompt

laude Model: **Sonnet 4.6**

In the context of C++23 and C++ Core Guidelines evaluate the following program

```cpp
#include <cassert>
#include <iostream>
#include <stdexcept>
#include <type_traits>
template<typename B>
class global
{
public:
    template<typename D = B, typename... Args>
    void init(Args&&... args) {
        static_assert(std::is_same_v<B, D> || std::is_base_of_v<B, D>,
                      "Type D and B must be identical or type D must be derived from B.");
        if(m_ptr != nullptr)
        {
            throw std::runtime_error(__PRETTY_FUNCTION__);
        }
        else
        {
            m_ptr = new D(std::forward<Args>(args)...);
        }
    }
    void destroy() {
        if(m_ptr == nullptr)
        {
            throw std::runtime_error(__PRETTY_FUNCTION__);
        }
        else
        {
            delete m_ptr;
            m_ptr = nullptr;
        }
    }
    B& get(){
        if(m_ptr == nullptr)
        {
            throw std::runtime_error(__PRETTY_FUNCTION__);
        }
        else
        {
            return *m_ptr;
        }
    }
    bool exists() const {
        return m_ptr != nullptr;
    }
    global() = default;
    ~global() {
        assert(m_ptr == nullptr && "Function destroy() was not called");
    }
    global(const global&) = delete;
    global(global&&) = delete;
    global& operator=(const global&) = delete;
    global& operator=(global&&) = delete;
private:
    B* m_ptr{nullptr};
};
class Matrix
{
public:
    Matrix() {}
    Matrix transpose() const { return Matrix{}; }
};
global<Matrix> gMatrix;
int main()
{
    // First, the global variable must be initialized
    assert(!gMatrix.exists());
    gMatrix.init(/* arguments to class Matrix*/);
    assert(gMatrix.exists());
    // Use the global variable of type Matrix
    gMatrix.get().transpose();
    // Lasst the global variable must be destroyed
    gMatrix.destroy();
    assert(!gMatrix.exists());
    std::cout << "Job done\n";
    return 0;
}
```

## Claude is thinking

```
Let me analyze this C++ program in the context of C++23 and C++ Core Guidelines.

The program implements a global<B> template class that manages a pointer to dynamically allocated objects. Let me evaluate it systematically:
Issues identified:

    Manual memory management (C.149, R.11): The class uses raw pointers with new and delete, which is against modern C++ practices. Should use smart pointers (std::unique_ptr).
    Resource leak potential (C.31, R.1): The destructor only asserts but doesn't clean up. If someone forgets to call destroy(), there's a memory leak.
    Exception safety (E.2, E.6): If an exception is thrown during init(), the program might terminate or the object might not be properly initialized.
    Using exceptions for control flow (E.3): Throwing exceptions in normal scenarios (double init, double destroy) is not ideal.
    Non-const get() (C.164): Should probably have both const and non-const versions.
    Static_assert placement: It's good but the template default parameter pattern is somewhat unusual.
    PRETTY_FUNCTION (not standard): This is a GCC/Clang extension, not standard C++.
    Singleton pattern concerns (I.3): Global state is generally discouraged.
    Rule of Five violation (C.21): While copy/move are deleted, the destructor doesn't clean up properly.
    Assert in destructor (E.16): Assertions in destructors can lead to program termination during stack unwinding.
    No thread safety: Not thread-safe if used in multi-threaded context.
    Typo in comment: "Lasst" should be "Last"

Let me structure a comprehensive evaluation.
```

# Evaluation of the C++ Program

This program implements a manual lifetime management wrapper for global objects. 

Here's a detailed analysis against C++23 and C++ Core Guidelines:
## 🔴 Critical Issues

### 1. Raw Pointer Management (C.149, R.11, R.3)
+ Violates "Avoid raw pointers for ownership"
+ Should use `std::unique_ptr<B>` instead of `B* m_ptr`
+ Current design is error-prone and leak-prone


### 2.Destructor Doesn't Clean Up (C.31, R.1, C.21)
+ Destructor only `assert`s but doesn't delete the resource
+ Memory leak if `destroy()` not called
+ Violates "A destructor should release all resources"
+ In release builds, the assert disappears and leak is silent


### 3. Exception in Destructor Risk (E.16)
+ Assert in destructor can terminate during stack unwinding
+ Destructors should be `noexcept` and not throw



## ⚠️ Major Issues

### 4. Manual Lifetime Management Anti-Pattern (I.3, R.6)
+ Requiring manual `init()`/`destroy()` calls defeats RAII
+ Easy to forget `destroy()`, causing leaks
+ Order dependencies are fragile


### 5.Non-Standard Extension (Portability)
+  `__PRETTY_FUNCTION__` is GCC/Clang-specific
+ Use `std::source_location` (C++20) or standard alternatives


### 6.Global Mutable State (I.2)
+ Global variables are discouraged
+ Makes testing difficult, creates hidden dependencies
+ Not thread-safe


### 7. Exceptions for Control Flow (E.3)
+ Throwing exceptions for double-init/destroy is expensive
+ These are programming errors, not exceptional circumstances
+ Consider contracts (C++26) or just assertions



## ⚙️ Design Issues

### 8.Incomplete Rule of Five (C.21)
+ Copy/move are deleted (good), but destructor is incomplete
+ Should either manage resource or not have special destructor


### 9.Missing const correctness (C.164)
+ `get()` should have const and non-const overloads

```cpp
B& get();
const B& get() const;
```

### 10.Template Parameter Pattern
+ `template<typename D = B>` allows derived types (good)
+ But could be clearer with concepts (C++20)


### 11.No Thread Safety (CP.2)
+ Not thread-safe for initialization/access
+ Needs synchronization if used across threads

## 🔄 Improved Version

```c++
#include <iostream>
#include <memory>
#include <source_location>
#include <stdexcept>
#include <type_traits>

template<typename B>
class global {
public:
    template<typename D = B, typename... Args>
        requires std::is_same_v<B, D> || std::is_base_of_v<B, D>
    void init(Args&&... args) {
        if (m_ptr) {
            throw std::logic_error("Already initialized");
        }
        m_ptr = std::make_unique<D>(std::forward<Args>(args)...);
    }

    void destroy() noexcept {
        m_ptr.reset();
    }

    [[nodiscard]] B& get() {
        if (!m_ptr) {
            throw std::logic_error("Not initialized");
        }
        return *m_ptr;
    }

    [[nodiscard]] const B& get() const {
        if (!m_ptr) {
            throw std::logic_error("Not initialized");
        }
        return *m_ptr;
    }

    [[nodiscard]] bool exists() const noexcept {
        return m_ptr != nullptr;
    }

    global() = default;
    ~global() = default;  // unique_ptr handles cleanup

    global(const global&) = delete;
    global(global&&) = delete;
    global& operator=(const global&) = delete;
    global& operator=(global&&) = delete;

private:
    std::unique_ptr<B> m_ptr;
};

class Matrix {
public:
    Matrix() = default;
    Matrix transpose() const { return Matrix{}; }
};

global<Matrix> gMatrix;

int main() {
    assert(!gMatrix.exists());
    gMatrix.init();
    assert(gMatrix.exists());
    
    gMatrix.get().transpose();
    
    gMatrix.destroy();
    assert(!gMatrix.exists());
    
    std::cout << "Job done\n";
}
```

## 🎯 Better Alternative: RAII

Consider whether manual lifetime management is necessary at all:
```cpp
#include <iostream>
#include <memory>

template<typename B>
class global_holder {
    std::unique_ptr<B> m_ptr;
public:
    template<typename D = B, typename... Args>
        requires std::derived_from<D, B> || std::same_as<D, B>
    explicit global_holder(Args&&... args) 
        : m_ptr(std::make_unique<D>(std::forward<Args>(args)...)) {}
    
    B& operator*() { return *m_ptr; }
    const B& operator*() const { return *m_ptr; }
    B* operator->() { return m_ptr.get(); }
    const B* operator->() const { return m_ptr.get(); }
};

// Initialize at construction
global_holder<Matrix> gMatrix;  // RAII!

int main() {
    gMatrix->transpose();
    std::cout << "Job done\n";
}
```

## 📋 Summary
| Aspect           | Rating     | Key Issues |
|------------------|------------|------------|
| Memory Safety    | ❌ Poor    | Raw pointers, potential leaks |
| RAII Compliance  | ❌ Violate | dManual init/destroy required |
| Exception Safety | ⚠️ Weak    | Exceptions in control flow |
| Modern C++       | ⚠️ Dated   | Should use smart pointers, concepts |
| Portability      | ⚠️ Limited | Non-standard extensions |
| Thread Safety    | ❌ None    | No synchronization |



**Recommendation**: Redesign using RAII principles with `std::unique_ptr` or reconsider if manual lifetime management is truly necessary.
