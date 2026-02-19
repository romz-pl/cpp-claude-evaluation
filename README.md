# C++ Development with Claude

> [!NOTE]
> This project leverages [Claude.ai](https://claude.ai) for C++ development assistance.
> 
> Claude can help with:

## Code Development
+ Writing new C++ code with modern standards (C++11/14/17/20/23)
+ Implementing algorithms, data structures, and design patterns
+ Generating boilerplate code and class scaffolding
+ Creating unit tests and test fixtures

## Code Review & Debugging
+ Analyzing code for potential bugs, memory leaks, and undefined behavior
+ Identifying performance bottlenecks and optimization opportunities
+ Reviewing code for best practices and C++ idioms
+ Suggesting refactoring improvements

## Problem Solving
+ Explaining complex C++ concepts (templates, smart pointers, move semantics, etc.)
+ Debugging compiler errors and understanding error messages
+ Recommending appropriate STL containers and algorithms
+ Providing guidance on memory management and RAII principles

## Build & Tooling
+ Assistance with CMake configuration
+ Help with build system setup and dependency management
+ Guidance on compiler flags and optimization settings

Claude can work with your existing codebase through file uploads and provide context-aware suggestions. For complex tasks, it can create complete, compilable code examples and explain the reasoning behind implementation decisions.

# C++ Core Guidelines

> [!NOTE]
> The [C++ Core Guidelines](https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines) is a collaborative effort led by **Bjarne Stroustrup** and **Herb Sutter** to establish best practices for modern C++.
> These guidelines aim to help developers write safer, more efficient, and more maintainable code by providing concrete recommendations on:

## Resource management and ownership
+ Type safety and interfaces
+ Memory safety and error handling
+ Concurrency and performance
+ Code organization and style

The guidelines are designed to be supported by static analysis tools and focus on helping developers avoid common pitfalls while leveraging modern C++ features (C++11 and beyond). 
They emphasize writing code that is not just correct, but also clear, maintainable, and efficient.


## Key principles:
+ Express ideas directly in code
+ Write code that is type-safe and resource-safe by default
+ Don't leak resources
+ Don't waste time or space
+ Prefer compile-time checking to run-time checking


# Evaluated C++ programs
> [!NOTE]
> Claude can evaluate any C++ program.
>
> See the examples and generated reports

+ [A dynamic polymorphism](./eval-00.md)
+ [A global variable with a managed lifetime](./eval-01.md)
+ [A global variable with a managed lifetime in C++23](./eval-04.md)
+ [A random number generator wrapper](./eval-02.md)
+ [Two threads and I/O](./eval-03.md)
