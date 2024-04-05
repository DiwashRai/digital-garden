---
title: "Modern Template Metaprogramming - A Compendium"
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
// primary template assumes the bool value is true
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
// primary template assumes the bool value is true
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

## Using inheritance and specialization together
-   Example 1: `is_void`
```cpp
// primary template for non-void types
template<class T> struct is_void : false_type {};

// specializations recognize each of the four void types:
template<> struct is_void<void> : true_type {};
template<> struct is_void<void const> : true_type {};
...
```

-   Example 2: `is_same`
```cpp
// primary template for distinct types
template<class T, class U> struct is_same : false_type {};

// specialization recongnizes identical type
template<class T> struct is_same<T, T> : true_type {};
```

## Forwarding/delegating to other metafunctions

-   Given a type, is it a void type? (`is_void` but different implementation)
```cpp
template<class T>
using is_void = is_same<remove_cv_t<T>, void>;

// where remove_cv is simply
template<class T>
using remove_cv = remove_volatile<remove_const_t<T>>;
```

## Using parameter pack in a metafunction
-   Example: generalize `is_same` into `is_one_of`
```cpp
// primary template: is T the same as any of the types P0toN...?
template<class T, class... P0toN>
struct is_one_of; // declare interface only

// base #1: specialization recognizes empty list of types
template<class T>
struct is_one_of<T> : false_type {};

// base #2: Specialization recongizes match at head of list of types
template<class T, class... P1toN>
struct is_one_of<T, T, P1toN...> : true_type {};

// specialization recognizes a mismatch at head of list of types
template<class T, class P0, class... P1toN>
struct is_one_of<T, P0, P1toN...> : is_one_of<T, P1toN...> {};
```

-   Example: redoing `is_void` with parameter packs
```cpp
template<class T>
using is_void = is_one_of<T, void, void const, void volatile, void const volatile>;
```

## Unevaluated operands
-   Recall that operands of `sizeof`, `typeid`, `decltype` and `noexcept` are _never_ evaluated,
    not even at compile time:
    -   Implies that no code is generateed for such operand expressions and...
    -   Implies that we need only a declaration, not a definition, to use a (function's or object's)
        name in these contexts.
-   An unevaluated function call (e.g. to `foo`) can usefully map one type to another:
    -   `decltype(foo(declval<T>()))`
        -   Give's foo's return type, were it called with a T rvalue
    -   The unevaluated call `std::declval<T>()` is declared to give an rvalue result of type T.
        -   (`std::declval<T&>()` gives lvalue)

### Example: is_copy_assignable
```cpp
template<class T>
struct is_copy_assignable {
private:
    template<class U, class = decltype(declval<U&>() = declval<U const&>())>
    static true_type try_assignment(U&&);

    static false_type try_assignment(...);

public:
    using type = decltype( try_assignment(declval<T>()));
}
```
-   Key concepts to understanding how this works:
    -   `try_assignment` is a function within the is_copy_assignable struct that is called to
        generate the return `type` for the metafunction.
    -   `try_assignment` does not need a body, only a declaration as code is not actually being
        called at runtime.
    -   `U&&` is utilised not for perfect forwarding, but to ensure the try_assignment function
        accepts any type (const U, U, U&...);

## Type trait void_t
```cpp
template<class...>
using void_t = void;
```
-   Whatever you give it, it returns `void`. Can give it any number of types.
-   What's the point?
-   Acts as a metafunction that maps any ==well-formed== type(s) into the(predictable!) type void.
-   Could think of it as a `is_well_formed` metafunction that returns a _predictable_ type.
-   Example: detect the presence/absence of a type member
```cpp
// primary template
template<class, class = void> // second parameter default to void is crucial
struct has_type_member : false_type {};

// partial specialization:
template<class T>
struct has_type_member<T, void_t<typename T::type>> : true_type {};
```
**Breakdown**
-   Called via `has_type_member<T>::value` or equivalent
-   When T _does_ have a type member named _type_:
    -   The partial specialization is well formed, `void_t` returns void and the specialization
        inheriting from `true_type` is selected.
    -   If the `typename T::type` is not well formed, **SFINAE**, so the primary template has to
        be used.

### Lets revisit is_copy_assignable with our new tools

```cpp
// helper alias for the result type of a valid copy assignment
template<class T>
using copy_assignment_t = decltype( declval<T&>() = declval<T const&>());

// primary template handles all the non-copy assignable types:
template<class T, class = void> // default arg essential
struct is_copy_assignable : false_type {};

// specialization recognizes and validates only copy-assignable types:
template<class T>
struct is_copy_assignable<T, void_t<copy_assignment_t<T> >
: is_same<copy_assignment_t<T>, T&> {};
```

-   Want `is_move_assignable`? Change `T const&` -> `T&&`

```cpp
// helper alias for the result type of a valid copy assignment
template<class T>
using move_assignment_t = decltype( declval<T&>() = declval<T&&>());
```
