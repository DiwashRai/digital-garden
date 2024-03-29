---
title: "Modern Template Metaprogramming: A Compendium"
tags:
-   source
-   infomedia
-   cppcon
---
Link:  
-   [Back to Basics: Templates in C++ Part 1](https://www.youtube.com/watch?v=XN319NYEOcE)
-   [Back to Basics: Templates in C++ Part 2](https://www.youtube.com/watch?v=2Y9XbltAfXs)

Author: [[Walter Brown]]  
Topics: [[Software Engineering]]  

---

## Preview of examples

-   From the std::library
    -   `integral_constant`, `true_type`, `false_type`
    -   `is_same`, `is_void`
    -   `is_copy_assignable`, `is_move_assignable`
    -   `remove_const`, `remove_volatile`, `remove_cv`
    -   `conditional`, `enable_if`
    -   `is_integral`, `is_floating_point`
-   Not(yet?) in the std::library
    -   `abs`, `gcd`
    -   `type_is`, `bool_constant`
    -   `is_one_of`
    -   `void_t`!, `has_type_member`

## What is template metaprogramming
-   C++ _template metaprogramming_ uses _template instantiation_ to drive _compile-time evaluation_.
-   Template metaprogrammers exploit this machinery to improve:
    1.  source code flexibility
    2.  runtime performance

## When metaprogramming...
-   Run-time == compile-time in metaprogramming. So no relying on
    -   mutability
    -   virtual functions
    -   other RTTI

## abs

```cpp
template<int N>
struct abs {
    static_assert(N != INT_MIN);
    static constexpr int value = (N < 0) ? -N : N; // "return"
}
```

-   Using a _metafunction_
    -   A metafunction's args are supplied as the templates args.
    -   "Call" syntax is to request a template's published _value_ (or _type_);
        -   `abs<n>::value`

## Comparison with constexpr functions

-   Metafunctions have a slighly less familiar call syntax. i.e. angle brackets instead of normal
    brackets.
-   However, as **structs**, metafunctions give us more tools.
    -   Public member type declarations (e.g. _typedef_ or _using_).
    -   Public member data declartions (_static const_/_constexpr_, each initialized via a constant
        expression).
-   Public member function declarations and _constexpr_ member function definitions.
-   Public member templates, _static assert_s, and more!

## Compile-type recursion with specialization as base. example: gcd

-   Euclidean algorithm for gcd is a recursive function with a base case.
```cpp
int gcd(int a, int b)
{
    if (a == 0)
        return b;
    return gcd(b % a, a);
}
```

-   Here is the templated version.
```cpp
template<unsigned M, unsigned N>
struct gcd {
    static int constexpr value = gcd<N, M%N>::value; // euclid
}
```

-   For the base case we use a specialization.

```cpp
template<unsigned M>
struct gcd<M, 0> {
    static_assert(M != 0);  // gcd(0,0) is undefined, so disallow
    static int constexpr value = M;
}
```

## Metafunctions can take a type as an parameter/argument. Example: rank
-   `sizeof` is a built-int _type function_, but we can write our own.
-   Example: Obtain (compile-time) rank of an array type:

```cpp
// primary template handles scalar(non-array) types as base case:
template<class T>
struct rank {
    static size_t constexpr value = 0u;
};

// partial specialization recognizes any array type:
template<class U,  size_t N>
struct rank<U[N]> {
    static size_t constepxr value = 1u + rank<U>::value;
}
```

-   Usage:
    -   `using array_t = int[10][20][30];`
        -   `rank<array_t>::value` yields 3u at compile time.

## Metafunctions can produce a type as its result. Example: remove_const

```cpp
// primary template handles types that aren't const-qualified
template<class T>
struct remove_const { using type = T; }; // identity

// partial specialization matches const-qualified types
template<class U>
struct remove_const<const U> { using type = U; };
```

-   Usage:
    -   `remove_const<T>::type t;`
    -   `remove_const_t<T> t;` after C++ 14. Just an alias


## Metafunction convention 1

-   Metafunction with a _type result_ aliases that result to _type_.
-   Example: `template<class T> struct type_is { using type = T; };`
    -   This is just an identify function, but it is surprisingly useful...
-   Now we can use apply the convention via _inheritance_.
```cpp
// primary template handles types that are not volatile-qualified
template<class T>
struct remove_volatile : type_is<T> {};

// partial specialization recognizes volatile-qualified types:
template<class T>
struct remove_volatile<U volatile> : type_is<U> {};
```

## Compile time decision making - Example: enable_if


