---
title: "C++ guidelines"
tags:
-   molecule
---
Topics: [[Software Engineering]]  
Reference:  
-   [[Effective C++]]

---

## General rules
-   Prefer the compiler over the preprocessor
    -   `constexpr`, `const`, `enums` and inlines over defines
-   `constexpr` and `const` whenever possible.
-   Ensure objects initialised before use e.g. `int x = 0;`
-   Prefer _pass-by-reference-to-const_ to pass-by-value
-   Postpone varialbe definitions as late as possible.
-   Minimise casting, especially dynamic casting.

## Type design
-   Assignment operators should:
    -   return a reference to `this`
    -   handle assignment to self
-   Polymorphic base classes should declare virtual destructors.
-   Ensure all parts of an object are copied, **including** the base class.
-   RAII classes should provide access to underlying resource. e.g. `unique_ptr::get`
-   Prefer non-member non-friend functions to member functions.
    -   Focus on _tighter_ encapsulation which means utility functions that **can** be free
        functions **should** be free functions.
-   Data members should be private.
-   Try to support non-throwing swap for your types.
-   Avoid returning 'handles' to object internals.
-   Strive for exception safe code:
    -   Exception safety requirements:
        -   Leak no resources
        -   Don't allow data corruption
    -   Exception safe functions offer one of three guarantees:
        -   **Basic guarantee**
            -   Program remains in a valid state, even if objects values are changed.
        -   **Strong guarantee**
            -   State of program remains unchanged. Meaning the functions are `atomic`
            -   _Copy-and-swap_ pattern can be used to achieve strong guarantee.
        -   **nothrow guarantee**
            -   Always do what they promise to.
