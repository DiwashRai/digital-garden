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

## Compile time decision making - Example: IF/IF_t

-   Imagine a metafunction, IF/IF_T to select one of two types:
```cpp
template<bool p, class T, class F>
struct IF : type_is<...> { }; // p ? T : F
```
-   Something like this would let us write _self-configuring code_:
    -   Assume: `int ocnst q = ...; // user's configuration parameter`
    -   `IF_t<(q<0), int, unsigned> k; // k is int if less than 0, otherwise unsigned`

### Implementing IF

```cpp
// primary tempalte assumes the bool value is true
template<bool, class T, class > // needn't name unused params
struct IF : type_is<T> {};

// partial specialization recognizes a false value
template<bool, class T, class U>
struct IF<false, T, F> : type_is<F> {};
```

-   IF is called `conditional` in the `stl`.

### Implementing a single-type variation on conditional

-   "if _true_, use the given type; if _false_, use no type at all":
```cpp
// primary tempalte assumes the bool value is true
template<bool, class T = void> // default is useful, not essential
struct enable_if : type_is<T> {};

// partial specialization recognizes a false value, computing nothing:
template<class T>
struct enable_if<false, T> {}; // no member named type!
```

-   Why is this useful? Let's consider a meta-call `enable_if<false, ...>::type`
    -   Always an error?
    -   Nope. Only sometimes. Welcome to **SFINAE**


## SFINAE applies during implicit template instantiation
SFINAE: Substitution Failure Is Not An Error.

During template instantiation, the compiler will:
1.  Obtain (or figure out) the template arguments:
    -   Taken verbatim if explicitly supplied.
    -   Else _deduced_ from function arguments at point of call.
    -   Else taken form the declartion's _default template arguments_
2.  Replace each template parameter, throughout the template, but it's corresponding template
    argument. _Substituion_.
    -   If these steps produce well-formed code, the instantiation succeeds.
    -   BUT if the resulting code is ill-formed, it is considered not _viable_ (due to
        _substitution failure_) and is _silently discarded_.

### SFINAE in use

-   Example: Want one algorithm `f` taking integral types T, and overload it with a second `f`
    taking floating-point types T.
```cpp
template<class T>
enable_if_t<is_integral<T>::value, maxint_t>
f(T val) { ... };

template<class T>
enable_if_t<is_floating_point<T>::value, long double>
f(T val) { ... };
```
-   Only one can be viable at a time. If neither is viable, the compiler will generate an error.
-   For a quick taste of concepts, the above templates could be written like so:
```cpp
template<Integral T> // constrained template (short form)
maxint_t
f(T val) { ... };
```
-   This allows you to avoid the `enable_if_t` shenanigans and also allows clearer compiler
    messages.

## Metafunction convention 2
-   A metafunction with a _value result_ has:
    -   A `static constexpr` member, _value_, giving it's result, and...
    -   A few convenience member types and `constexpr` functions.
-   Canonical value-returning metafunction (equivalent to `type_is`):
```cpp
template<class T, T v>
struct integral_constant {
    static constexpr T v;
    constexpr   operator T() const noexcept { return value; }
    constexpr T operator()() const noexcept { return value; }
    ... // remaining members are only occasionally useful
}
```
-   Inheriting from `integral_constant` provides more options for meta-call syntax.

### Revised rank metafunction
-   Example: obtain the (compile-time) rank of an array type:
```cpp
// primary template handles scalar (non-array) types as base case
template<class T>
struct rank : integral_constant<size_t, 0u> { };

// partial specialization that handles unboudned arrays
template<class U, size_t N>
struct rank<U[N]>
: integral_constant<size_t, 1u + rank<U>::value> { };

// partial specialization for unbounded array.
template<class U>
struct rank<U[]>
: integral_constant<size_t, 1U + rank<U>::value> { };
```

### Some integral_constant conveniences
-   A useful convenience alias:
```cpp
template<bool b>
using bool_constant = integral_constant<bool, b>;
```
-   This then allows the following:
```cpp
using true_type = bool_constant<true>;
using false_type = bool_constant<false>;
```
