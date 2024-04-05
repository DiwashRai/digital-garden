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

However, for an inline variable or inline function, a definition is required per translation unit.
==The rules for inline variables and functions also apply to templates==.

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
>       -   Expressions that designate variables or functions.
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
-   Possible _ParameterType_ forms.
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

> [!tip] Brief
>   -   You can explicitly specify a template argument e.g. `min<double>(42, 3.14);`. This may be
>       necessary to avoid ambiguity.
>   -   If we do not specify explicity, _type inference_ takes place. _Expression types_ have to be
>       resolved to a _parameter type_.
>   -   _Reference collapsing_ can happen during type inference. Typically we use the least number
>       of '&'. See matrix for more details.
>   -   The special rules for parameter type _T&&_ allows _perfect forwarding_ to take place.

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

> [!tip] Brief
>   -   You can use auto for function templtaes to determine return type.
>   -   Another alternative is to use `std::common_type_t`.

### Class Templates - More Detail

**Stack example**

-   Where is the scope of the template parameter active?
```cpp
template<class T> // Scope starts when T is named
class Stack
{
    vector<T> m_data;

public:
    bool is_empty() const;
    void pop();
    void push(T const& t);
}; // an ends here


template<class T> // From here when T is introduced
void Stack<T>::push(T const& t)
{
    m_data.push_back(t);
} // and ends here
```

-   Do you have to use `Stack<T>` if you refer to the class itself?
    -   Not if you are in class scope. Therefore, most people omit it as it is visual noise.

```cpp
template<class T>
class Stack
{
    vector<T> m_data;

public:
    bool is_empty() const;
    void pop();
    void push(T const& t);
    Stack push_all_from(Stack const& other); // Don't need Stack<T> in class scope
};

```

-   Now what if you define the function outside of the class?
-   Class scope only begins after the `::` so the first `Stack<T>` is required but not in the
    function parameter list.

```cpp
template<class T>
Stack<T>::push_all_from(Stack const& other)
{
    Stack tmp(*this);
    m_data.insert(m_data.end(), other.m_data.cbegin(), other.m_data.cend());
    return tmp;
}
```

-   Now lets say we want a `begin()` and `end()` function that returns a `const_iterator`. We can
    just reuse the reverse iterator provided by `std::vector`.
-   We can use a `using` to type alias it however, we also need to use the keyword `typename` to
    tell the compiler that it is indeed a type.
-   This is required whenever you have a dependent name - something that is dependent on one or
    more template parameters.

```cpp
template<class T>
class Stack
{
    vector<T> m_data;

public:
    // typename required since `vector<T>` is dependent on T
    using const_iterator = typename vector<T>::const_reverse_iterator;

    const_iterator begin() const;
    const_iterator end() const;

    bool is_empty() const;

    void pop();
    void push(T const& t);
    Stack push_all_from(Stack const& other); // Don't need Stack<T> in class scope
};
```
-   You also need to do this for things outside of class scope

```cpp
template<class T> typename Stack<T>::const_iterator
Stack<T>::begin() const
{
    return m_data.crbegin();
}

// or a cleaner way to do the same thing
template<class T> auto
Stack<T>::begin() const -> const_iterator // don't need typename as we are now in class scope
{
    return m_data.crbegin();
}
```

-   How do we define static data members(after C++17)?

```cpp
template<class T>
class Stack
{
    vector<T> m_data;
    inline static int m_count = 0; // after C++17 we can use the inline keyword
    ...
}
```

_Full Specialization_

```cpp
template<>
class Stack<int>
{
    vector<int> m_data;

public:
    bool is_empty() const;
    int top() const;

    void pop();
    void push(int t);
    void (push_from(string const& s);
};

void
Stack<int>::push_from(string const& s)
{
    m_data.insert(m_data.end(), s.begin(), s.end());
}
```

_Partial specialization_
-   What if we want to customize behaviour for pointers specifically?

```cpp
template <class T> // Still need template parameters
class Stack<T*>    // But then need to specify this stack is for pointers
{
    vector<T*> m_data;
    ...
    T* pop(); // returns the pointer not void so it object can be deleted.
    ...
};

template<class T> // template declaration still required
T*
Stack<T*>::pop()
{
    T* tmp = m_data.back();
    m_data.pop_back();
    return tmp;
}
```
-   Partial specialization can be used to normalize internal representation if the user is using
    a mix of references and non reference types.

```cpp
template<class T, class U>
struct Pair
{
    T first;
    U second;
    Pair(T const& t, U const& u) {...};
};

// now partially specialise for different cases
template<class T, class U>
struct Pair<T&, U>
{
    T first; // internal representation is the same for all cases
    U second;
    Pair(T const& t, U const& u) {...};
};

// same if U is a ref
template<class T, class U>
struct Pair<T, U&>
{
    T first;
    U second; // internal representation is the same for all cases
    Pair(T const& t, U const& u) {...};
};
```

> [!tip] Brief
>   -   You do not need to specify the `<T>` when referring to the class within the class scope
>       itself.
>   -   For member function defined outside of the class, class scope begins after the `::`. This
>       means that you will need the `<T>` before that(e.g. for return type) but not after
>       that(e.g. in the function parameter list).
>   -   If you use `using` to introduce a _type alias_ that has a _dependent name_(relies one or
>       more template parameters T), then you need the `typename` keyword to let the compiler know
>       that it is a type.
>   -   Typename will also be required for definitions outside of the class scope but can be
>       quite elegantly avoided with `auto` and `-> return type` syntax.
>   -   Since C++17 you can define static variables in the class body with `inline`.
>   -   Class templates can be fully specialized with a similar syntax to function templates.
>   -   Member functions defined outside don't need the `template<class T>` in this case.
>   -   You can partially specialize class templates. For example `T*` for a class template that
>       takes in `T` to handle pointers in a specific way.
>   -   Partial specialization can be used in many ways such as to normalize internal types if
>       the passed Type varies (e.g. mix of reference and non reference types).

What else can we do with the power of _partial specialization_?

### Type Traits

```cpp
template<class T>
struct IsPointer
{
    static constexpr bool value = false; // false for the general case
}

template<class T>
struct IsPointer<T*>
{
    static constexpr bool value = true; // True when T is actually a T*
}

// Now we can create a template type alias
template<class T>
inline constexpr
bool IsPointer_V = IsPointer<T>::value;
```

-   Now we can do some template magic. The other branch is not even generated with an
    `if constexpr`.

```cpp
template<class T>
void foo(T t)
{
    if constexpr (IsPointer_V)
        // do one thing
    else
        // do something else
    ...
}
```

-   We can also use partial specialization to normalize properties of types.
    -   i.e. remove constness or volatile etc.

```cpp
template<class T>
struct RemoveCV
{
    using Type = T;
}

// partially specialize for cases where T has const or volatile or both and just return T as Type
template<class T>
struct RemoveCV<T const>
{
    using Type = T;
}

template<class T>
struct RemoveCV<T volatile>
{
    using Type = T;
}

template<class T>
struct RemoveCV<T const volatile>
{
    using Type = T;
}

template<class T>
using RemoveCV_T = typename RemoveCV<T>::type;
```

-   This use of traits can be used to improve templated code. If you do not trust your users, you
    can normalize the type parameters that are passed.

> [!tip] Brief
>   -   Can be use to detect properties such as if a type is a pointer.
>   -   Can also be used to detect and normalize properties of a type. i.e. Can remove constness,
>       volatile etc.

