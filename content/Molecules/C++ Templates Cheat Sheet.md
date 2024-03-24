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

## Template categories

### Function Templates (C++98/03)

### Class Templates (C++98/03)

### Member Function Templates (C++98/03)

### Alias Templates (C++11)

### Variable Templates (C++14)

### Lambda Templates (C++20)

## Template Fundamentals

### Definition of a template
'thing' template is treated consistently.
-   _template_ is the noun, indicating a parametrized description
-   _thing_ is an adjective, specifying the family of things being parametrized.

For example:
-   Class Template -> is a parametrized description of a family of classes
-   Function Template -> is a parametrized description of a family of functions  

etc...  

### One-definition rule
A program must contain exactly one definition of every non-inline variable or function. Multiple
declarations are permitted.

Howver, for an inline variable or inline function, a definition is required per translation unit.
==The rules for inline variables annd functions also apply to templates==.

> [!tip] Brief
> Define templates in a header file and include the header wherever the template is
> needed.

### Template parameters and template arguments

Template parameters are the thing that comes after the `template` keyword. e.g.  
`template<typename T1, typename t2>`. T1 and T2 are parameters.  

Template arguments are the concrete items substituted for template parameters. e.g.
`pair<string, double> my_pair;`. string and double are the arguments.

Template parameters can come in three flavors:
-   Type parameters
-   Non-type template parameters(NTTPs)
    -   Constant values that can be determined at compile or link time.
        -   Integer or enumeration type
        -   Pointer or pointer-to-member type
        -   std::nullptr_t
        -   ...
-   Template-template parameters
    -   Placeholders for class or alias templates

**Template-template parameter example:**  
Example to create an adaptor template for a stack

```cpp
#include <vector>
#include <list>

template<class T, template<class U, class A =std::allocator<U>> class C>
struct Adaptor
{
    C<T> my_data;
    void push_back(T const& t) { my_data.push_back(t); }
};

Adaptor<int, std::vector> a1;
a1.push_back(0);
```

Template parameters can have default arguments.

```cpp
// class C has a default type of vector

template<class T, template<class U, class A =std::allocator<U>> class C = vector>
struct Adaptor { ... };
```

However, they must be at the end of the list for **class**, **alias** and **variable** templates.
```cpp
// OK - T0 with no default is at the start

template <class T0, class T1=int, class T2=int, class T3=int>
class quad;


// Not OK - T4 with no default is at the end after T1, T2 and T3 with defaults

template <class T0, class T1=int, class T2=int, class T3=int, class T4>
class quint;

```

Function templates do not have this requirement.

```cpp
template<class RT=void, class T>
RT* address_of(T& value)
{
    return static_cast<RT*>(&value);
}:

```

> [!tip] Brief
>   -   Template parameter flavors:
>       -   Types
>       -   Non-type template parameters e.g. int(enumeration), pointers, std::nullptr_t
>       -   Template-template parameters. 'Placeholder' for class or alias types.
>   -   Template parameters can have default arguments.
>       -   Must be at the end for class, alias and variable templates
>       -   Anywhere for function templates

### Specialization

-   Template is the recipe that tells us how to generate something useful.
-   A specialization is the useful thing built from that recipe.
-   Q: How do we get from template to specialization?
    -   A1: Instantiation
    -   A2: Explicit spcialization

