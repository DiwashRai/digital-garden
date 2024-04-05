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
        when you use the name of a template alongside concrete parameters. `std::vector<int> vec;`
    -   Explicit: Looks like this: `template class std::vector<foo>;`
        -   Unlike implicit, this instantiates **all** members.

### Value categories
-   **lvalue**
    -   Have storage. e.g. class data members, variables, function that return lvalue ref.
-   **prvalue**. Think of it as a 'pure' value.
    -   Literals such as enumerations, integer constants.
-   **xvalue**. 'Expiring' value.
    -   Usually a _glvalue_ with a value that no longer matters. e.g. by using std::move


### Template argument deduction
-   Template parameter types:
    -   `T`, `T*`, `T const*`, `T&`, `T const&`, `T&&`, `T const&&`
-   Expression types:
    -   `int`, `int const`, `int*`, `int const*`, `int&`, `int const&`, `int&&`, `int const&&`
-   There must be some rules to resolve expressions when encountered.
-   Behold... reference collapsing
    -   & + & -> &
    -   & + && -> &
    -   && + & -> &
    -   && + && -> &
-   Special rules for specifically `T&&` when it encounters `int const&&` or `int&&` allows for
    perfect forwarding.
-   **Perfect forwarding** allows for templates to perfectly pass forward parameters whilst
    maintaining the exact 'prvalue', 'xvalue' or 'lvalue'.

### Return type deduction
-   You can use auto as a return type to let the compiler choose a type that is capable of
    representing both types if you have a return statement such as `return (t1 < t2) ? t1 : t2;`
-   `std::common_type_t<T1, T2>` does the same thing.

### Full specialization
-   Can specialize a class template for a specific type. I guess something similar happens with
    `vector<bool>`. The syntax would look something like this:
-   Used to customise behaviour for specific types. Partial specializations are for parameter
    types i.e. pointers vs refs etc.

```cpp
template<>
class Stack<int>
{
    vector<int> m_data;
    ...
};
```


### Partial specialization
-   Used to customise behaviour for specific parameter types e.g. pointers.

```cpp
template <class T> // Still need template parameters
class Stack<T*>    // But then need to specify this stack is for pointers
{
    vector<T*> m_data;
    ...
    T* pop(); // returns the pointer not void so it object can be deleted.
    ...
};

```

### Type Traits
-   Rely on the power of full specialization, partial specialization and **SFINAE** to do some
    extremely useful things in your templates.
-   Non exhaustive list:
    -   Detect properties such as if type is a pointer, ref etc.
    -   Normalize the parameter that's passed in e.g. `std::remove_cv`
    -   Use inheritance to return specific values e.g. `integral_constant<T, T v>`
        -> `bool_constant` -> `true_type`/`false_type`
    -   Use `std::conditional` to return the type at compile time. Chain it for compile time
        `if/else` syntax.
