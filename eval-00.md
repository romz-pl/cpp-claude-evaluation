# Evaluation of dynamic polymorphism


## Claude Prompt

In the context of C++23 and C++ Core Guidelines evaluate the following program 


=====================
```cpp
#include <print>
#include <memory>
#include <vector>
class Shape {
public:
    virtual ~Shape() = default;
    virtual void draw() const = 0;
};
class Rectangle : public Shape {
public:
    explicit Rectangle(int width) : width_(width) {}
    ~Rectangle() = default;
    void draw() const override {
        std::print("R{} ", width_);
    }
private:
    int width_;
};
class Circle : public Shape {
public:
    explicit Circle(int radius) : radius_(radius) {}
    ~Circle() = default;
    void draw() const override {
        std::print("C{} ", radius_);
    }
private:
    int radius_;
};
void draw_all(const std::vector<std::unique_ptr<Shape>>& shapes)
{
    for (const auto& p : shapes) {
        p->draw();
    }
}
int main()
{
    std::vector<std::unique_ptr<Shape>> v;
    v.emplace_back(std::make_unique<Rectangle>(1));
    v.emplace_back(std::make_unique<Circle>(2));
    draw_all(v);
    return 0;
}
```
## Claude is thinking

```
The user wants me to evaluate a C++ program in the context of C++23 and the C++ Core Guidelines. Let me analyze this program systematically.

First, let me understand what the program does:

    Defines a base class Shape with a pure virtual draw() method
    Two derived classes: Rectangle and Circle
    Uses std::unique_ptr to manage Shape objects polymorphically
    Stores them in a vector and calls draw on each

Now let me evaluate it against C++23 features and Core Guidelines:

C++23 Features Used:

    #include <print> - This is a C++23 feature (std::print)
    The rest is using older C++ features (smart pointers, vectors, virtual functions)

Core Guidelines Analysis:

Let me go through relevant guidelines:

    C.121: If a base class is used as an interface, make it a pure abstract class
        Shape has a pure virtual function and virtual destructor - GOOD
    C.127: A class with a virtual function should have a virtual or protected destructor
        Shape has a virtual destructor - GOOD
    C.128: Virtual functions should specify exactly one of virtual, override, or final
        Rectangle and Circle use override correctly - GOOD
    C.129: When designing a class hierarchy, distinguish between implementation inheritance and interface inheritance
        This follows interface inheritance pattern - GOOD
    C.35: A base class destructor should be either public and virtual, or protected and non-virtual
        Shape's destructor is public and virtual - GOOD
    R.20: Use unique_ptr or shared_ptr to represent ownership
        Uses unique_ptr for ownership - GOOD
    R.3: A raw pointer (a T) is non-owning*
        No raw owning pointers - GOOD
    ES.23: Prefer the {}-initializer syntax
        Member initializer lists are used appropriately
    I.11: Never transfer ownership by a raw pointer (T) or reference (T&)*
        Uses unique_ptr - GOOD
    C.45: Don't define a default constructor that only initializes data members; use in-class member initializers instead
        width_ and radius_ are initialized in constructors, could use in-class initializers but this is minor
    C.49: Prefer initialization to assignment in constructors
        Uses initializer lists - GOOD
    F.15: Prefer simple and conventional ways of passing information
        draw_all takes const reference to vector - GOOD
    C.131: Avoid trivial getters and setters
        No getters/setters at all - GOOD (none needed)
    ES.5: Keep scopes small
        Good scoping throughout

Potential Issues/Improvements:

    Explicit constructors: Both Rectangle and Circle use explicit on single-argument constructors - GOOD (C.46)
    Default destructors: Rectangle and Circle explicitly define default destructors, which is unnecessary since the compiler-generated ones would be identical. Could be removed to follow the rule of zero more strictly.
    In-class member initializers: width_ and radius_ could benefit from in-class initialization for cleaner default handling.
    Range-based for loop: The code correctly uses range-based iteration.
    Modern C++ features: The use of std::print aligns with C++23 standards, and the overall structure leverages smart pointers effectively.

The program demonstrates strong adherence to modern C++ guidelines. Minor refinements would include removing redundant default destructors and considering in-class member initializers for data members.
Done
```

# Evaluation of the C++ Program

This is a **well-written, modern C++ program** that demonstrates good practices. Here's a comprehensive analysis:
## ✅ Strengths
### C++23 Features
+ `std::print`: Correctly uses C++23's print facility instead of `std::cout`

## Core Guidelines Compliance
+ **C.121, C.35**: `Shape` is properly designed as an abstract interface with public virtual destructor ✓
+ **C.127**: Virtual destructor in base class ✓
+ **C.128**: Derived classes use `override` (not redundant `virtual`) ✓
+ **C.46**: Single-argument constructors are `explicit` (prevents implicit conversions) ✓
+ **R.20, R.3**: Uses `unique_ptr` for ownership, no raw owning pointers ✓
+ **F.15**: `draw_all` passes by const reference (efficient, no copying) ✓
+ **C.49**: Constructors use member initializer lists ✓
+ **ES.71**: Uses range-based for loop ✓

## 🟡 Minor Improvements (Not violations, just enhancements)

### 1. In-class Member Initializers (C.45, C.48)
```cpp
class Rectangle : public Shape {
private:
    int width_ = 0;  // Better: default initialization
};
```

### 2. Redundant Destructors (C.80)

The derived class destructors can be omitted entirely:
```cpp
class Rectangle : public Shape {
public:
    explicit Rectangle(int width) : width_(width) {}
    // ~Rectangle() = default;  // Can be removed
    void draw() const override {
        std::print("R{} ", width_);
    }
private:
    int width_;
};
```
**Reason**: Compiler-generated destructors are sufficient when there's nothing special to do.

### 3. Consider Validation

Add parameter validation if invalid values are possible:
```cpp
explicit Rectangle(int width) : width_(width) {
    if (width <= 0) throw std::invalid_argument("width must be positive");
}
```
### 4. Modern Alternatives

For C++23, consider:
```cpp
// Using ranges (if filtering/transforming)
void draw_all(const std::vector<std::unique_ptr<Shape>>& shapes) {
    for (const auto& p : shapes | std::views::filter([](auto& s) { return s != nullptr; })) {
        p->draw();
    }
}

// Or use std::println for automatic newline
std::println("R{}", width_);
```

## 📋 Summary
| Aspect          | Rating | Notes |
|-----------------|--------|-------|
| Core Guidelines	| 9.5/10 | Excellent adherence |
| C++23 Usage	    | 8/10   | Good use of std::print |
| Memory Safety   | 10/10  | Perfect - uses RAII via smart pointers |
| OOP Design      | 10/10  | Proper interface/inheritance pattern |
| Code Clarity    | 10/10  | Clean, readable, well-structured |

This code would pass most professional code reviews. The suggestions above are optimizations, not corrections of errors. 

Great job following modern C++ practices!
