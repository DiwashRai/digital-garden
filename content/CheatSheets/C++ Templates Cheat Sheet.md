---
title: "C++ Templates"
tags:
-   molecule
---
Topics: [[Software Engineering]]  
Reference:  
- [[C++ Templates - The Complete Guide(2nd edition)]]
- CppCon 2021 - Back to Basics: Templates

---

## Fundamentals

### ODR
-   [[CppCon 2021 - Back to Basics Templates#One-definition rule|One Definition Rule]]
    -   Just define the template in a header file and include that header file wherever you need
        the template.

### Template parameter types
-   What can template parameters be?
    -   Types
    -   Non-type parameter:
        -   Int (e.g. `std::array<char, 26>`)
        -   enumeration type
        -   pointer or reference to a class object(`T*`, `T&`)
        -   pointer or reference to a function (`template<void (*Func)(int)>`)
        -   pointer or reference to a class member function (`template<void (SomeClass::*Func)(int)>`)
        -   std::nullptr_t
        -   floating point type (c++ 20)

### Instantiation and specialization
-   Template instantiation is the process or act(verb) of creating a specialization of a template.
-   A specialization if the outcome of template instantiation. It is like the concrete instantiation
    of a 'recipe', the template.
-   Specialization is also referred to as 'instantiations' or 'instantiated class'.
-   Two ways to instantiate a template specialization:
    -   Implicit: The more commonly used method. Also known as _on-demand_ or _automatic_. It's
        when you use the name of a temlate alongside concrete parameters. `std::vector<int> vec;`
    -   Explicit: Looks like this: `template class std::vector<foo>;`
        -   Unlike implicit, this instantiates **all** members.

### Template argument deduction
