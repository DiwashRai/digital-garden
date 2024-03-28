---
title: "Back to Basics: Templates in C++"
tags:
-   source
-   infomedia
-   cppcon
---

Link:
-   [Back to Basics: Templates in C++ Part 1](https://www.youtube.com/watch?v=XN319NYEOcE)
-   [Back to Basics: Templates in C++ Part 2](https://www.youtube.com/watch?v=2Y9XbltAfXs)

Author: [[Bob Steagall]]  
Topics: [[Software Engineering]]  

---

## Template categories
-   Function Templates (C++98/03)
-   Class Templates (C++98/03)
-   Member Function Templates (C++98/03)
-   Alias Templates (C++11)
-   Variable Templates (C++14)
-   Lambda Templates (C++20)

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

**Definitions**
-   Template Instantiation: process when the compiler substitutes template arguments for template
    parameters in order to ==define== an entity.
    -   Results in the generation of a specialization of a template
-   Spcializations are often referred to informally as an 'instantiated class' (or instantiated
    function etc).
-   Then the term evolved such that specializations are also known as 'instantiations'.

> [!tip] Brief
>   -   Template instantiation is the process or act(verb).
>   -   Specialization is the result of the process
>   -   Specialization is also referred to as 'instantiations' or 'instantiated class' etc.


**Implicit Instantiation**  
-   Is what you usually use. Occurs when the compiler sees the name of a template alongside concrete
    parameters (specialization). The compiler will then try to generate the specialization.
-   Also known as **on-demand**, or **automatic** instantiation.
-   Compiler will decide where, when, and how much of a specialization to create.
    -   For class templates, ==implicit== instantiation does not always generate all members of the
        class.

**Explicit Instantiation**  
Looks like this:
```cpp
//- Source file e.g. MyFoo.cpp
template class vector<foo>;                     //- definition
template class vector<foo, my_allocator<foo>>;  //- definition

template void swap<foo>(foo&, foo&);            //- definition
template void swap(bar&, bar&);                 //- definiiton
```

-   Explicit instantiation of a class template instantiates **all** members.
-   But, you can also choose to instantiate individual member functions.

```cpp
template void vector<foo, my_allocator<foo>>::push_back(foo const&); //- definition
```

-   However, explicit instantiation can result in the one definition rule being violated. To
    deal with this when you explicitly instantiate a template you can do the following:

```cpp
//- Header file e.g. MyFoo.h
extern template class vector<foo>;                      //- Declared, not defined
extern template class vector<foo, my_allocator<foo>>;   //- Declared, not defined

extern template void swap<foo>(foo&, foo&);             //- Declared, not defined
extern template void swap(bar&, bar&);                  //- Declared, not defined
```

> [!example] Usecase
>   -   Compiling expensive templates upfront to avoid paying the cost per translation unit
>   -   Instantiating and putting templates into a DLL without users having to instantiate
>       the templates themselves as well.

**Explicit specialization**  
-   What if you want to specify a behaviour for specific situations. e.g.

```cpp
// primary template
template<class T>
T const& min(T const& a, T const& b)
{
    return (a < b) ? a : b;
}

// Full specialization with all the parameters filled in.
// Valid ONLY if function primary template has already been defined.
template<>
char const* min(char const* pa, char const* pb)
{
    return (strcmp(pa, pb) < 0) ? pa : pb;
}
```

### Value categories

-   Every C++ expression has an associated _type_ and belonges to a _value category_.
-   The standard classifies all expressions into one of 5 value categories:
    -   Two are _composite_ categories. _glvalue_ and _rvalue_.
    -   Three are _core_ categories. _lvalue_, _xvalue_, and _prvalue_.


-   _glvalue_: Informally stands for 'generalised' lvalue.
    -   Has storage
    -   Has a name
    -   Has an address that can be taken(some exceptions)
    -   Non-const glvalues can be assigned to(also some different exceptions)
-   _rvalue_: An expression that is either a prvalue or xvalue. It is a set of both of those.


-   _prvalue_: Informally stands for 'pure' rvalue.
    -   When the compiler is evaluating operations it frequently creates temporary objects which are
        these prvalues.
    -   Has no name
    -   Has no storage*
-   _xvalue_: Informally stands for 'expiring' value.
    -   Usually denotes a _glvalue whos value will no longer matter_. e.g. By using std::move
-   _lvalue_: a glvalue that is not an xvalue.

> [!tip] Brief
>   -   **lvalue** examples:
>       -   Expressions that designate variables or funcitons.
>       -   Class data members.
>       -   A call to a function that returns an lvalue reference
>   -   **prvalue** examples:
>       -   Literals like enumerations, integer constants, floating point constants.
>       -   Application of built-in arithmetic operators.
>       -   A call to a function with a non-reference return type.
>   -   **xvalues** examples:
>       -   A cast to an rvalue reference to an object type. i.e. std::move
>       -   A call to a function that returns an rvalue reference to an object type.
>           -   i.e. Returning a reference to an object that can be moved from
>   -   Compositive types:
>       -   _glvalue_: lvalue and xvalue.
>       -   _rvalue_: prvalue and xvalue.

### Template argument deduction

Here is a commonly use function:

```cpp
template<class T>
T const& min(T const& a, T const& b)
{
    return (b < a) ? b : a;
}
string s0 = "foo";
string s0 = "bar";
string s2 = min<string>(s0, s1); // Explicitly specifying template argument
string s3 = min(s0, s1);         // Template argument is deduced as a string

```
What if the template argument is ambiguous? We can help the compiler by being explicit.

```cpp
int i = 42;
double d = 3.14;
auto x = min(i, d);         // Error. Ambiguity. Is T int or double.
auto y = min<double>(i, s); // OK. Force T to be double.

```

If we do not explicitly tell the compiler what type the template parameter is, we enter
==type inference==.

**Type Inference**
-   Possible _PatameterType_ forms.
    1.  T
    2.  T*
    3.  T const*
    4.  T&
    5.  T const&
    6.  T&&
    7.  T const&&
-   Possible _expression_ types
    1.  int
    2.  int const
    3.  int*
    4.  int const*
    5.  int&
    6.  int const&
    7.  int&&
    8.  int const&&

So the question is, how does the compiler resolve an expression type depending on the template
parameter type encountered? It uses reference collapsing.
-   & + & -> &
-   & + && -> &
-   && + & -> &
-   && + && -> &

Can think of it as using the least number of '&' characters.

![[template argument deduction matrix.png]]

Here is a practical example.

```cpp
using RI = int&;

int         x = 42;
RI         rx = x;  // rx is int&
RI const& rrx = rx; // "int& const&", drop outer const, & + & -> &, rrx is int&

using RCI = int const&;
RCI&&     rcx = x;  // "int const& const&&", no outer cv-qualifier, & + && -> &
                    // rcx is int const&

using RRI = int&&;
RRI const&& rrcx = x;   // "int&& const&&", drop outer const, && + && -> &&
                        // rrcx is int&& 
```

You can notice that T&& should pretty much deduce to whatever the expression type is. However, it
has special rules for 'int const&&' and 'int&&'. This is to solve the ==perfect forwarding problem==.

For these two cases the '&' gets stripped off and the bare type is deduced. **This change to one
type of template parameter is what allows perfect forwarding**. We can use std::forward with the
correct template parameter type(T&&) and have it perfectly forward through any chain of function
calls and maintain 'prvalue', 'xvalue' or 'lvalue'.

### Return type deduction

```cpp
// can only compare same types
template<class T>
T min(T a, T b)
{
    return (b < a) ? b : a;
}

// auto can be used to let the compiler determine the return type
template<class T1, class T2>
auto min(T a, T b)
{
    return (b < a) ? b : a;
}

// std::common_type_t checks if there is a type that is capable of representing both T1 and T2
// and returns that
template<class T1, class T2>
std::common_type_t<T1, T2> min(T a, T b)
{
    return (b < a) ? b : a;
}
```
