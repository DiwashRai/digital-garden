---
title: C++ and Beyond 2012 - Atomic Weapons
tags:
-   source
-   infomedia
---
Link: [C++ and Beyond - Atomic Weapons](https://www.youtube.com/watch?v=A8eCGOqgvH4)  
Author: [[Herb Sutter]]  
Topics: [[Software Engineering]]  

---

# Roadmap
-   Optimisations, Races, and the Memory Model
-   Ordering - What: Acquire and Release
-   Ordering - How: Mutexes, Atomics, and/or Fences
-   Other restrictions
-   Code Gen and Performance: x86/64, IA64, POWER, ARM, ...???
-   Bonus:
    -   Relaxed Atomics
    -   Coda: Volatile

# Optimisations, Races and the memory model
-   The machine everyone codes for
![[simplified-cpu-model.png]]  

-   Real HW from 2008: Intel Dunnington(+ store buffers)
![[real-cpu-diagram.png]]  

-   Does the computer execute the program you wrote?
    -   Nope. due to:
        -   Compiler optimisations
        -   CPU out of order execution
        -   Cache coherency

## Two key concepts
-   **Sequential consistency.**
    -   Defined in 1979 by Leslia Lamport:
    -   "the result of any execution is the same as if the reads and writes occurred in the same
        order, and the operations of each individual processor appear in this sequence in the order
        specified by its program"
-   **Race condition.**
    -   A memory location can be simultaneously accessed by two threads, and at least one thread
        is a writer.
        -   Simultaneously == without 'happens-before' ordering.

## Dekker's and Peterson's algorithms
```cpp
// Thread 1:

flag1 = 1;       // a) declare intent
if (flag2 != 0)  // b)
    // resolve contention
else
    // enter critical section

```

```cpp
// Thread 2:

flag2 = 1;       // c) declare intent
if (flag1 != 0)  // d)
    // resolve contention
else
    // enter critical section

```
-   Q: Could both threads enter the critical section?
    -   Maybe: if a can pass b, or c can pass d, this breaks
    -   Solution 1 (good): Use a suitable atomic type(e.g. Java "volatile", C++ "std::atomic" for
        flag variables.
    -   Solution 2 (good?): Use system locks instead of own implementation
    -   Solution 3 (problematic): Write memory barrier after a and c.


**How can line a) be switched with line b) or line c) with line d)?**  
![[atomiic-weapons-store-buffer-flush-example.png]]

The flags are written but the write goes into the store buffer, then the opposite threads flags are
read and there is no read buffer, so you see stale values.

## Optimisations
-   The compiler knows
    -   the memory operations **in this thread** including data dependencies
-   The compiler does not know:
    -   Which memory locations are "mutable shared" variables that could change asynchronously
-   Solution: Tell it

# Ordering: What? Acquire and Release

## Key general concept: Transaction
-   Logical operation on related data that maintains an invariant.
-   Atomic: all-or-nothing
-   Consistent: reads a consistent state, or takes data from one consistent state to another
-   Independent: Correct in the presence of other transactions on sama data.
-   Example:
    -   Bank transactions.
    -   When transferring money, don't expose one account getting credited without the other
        account also getting debited.

## Key general concept: Critical region
-   **Critical region** = code that must execute in isolation
-   Locks
```cpp
{
    lock_guard<mutex> hold(mut_x); // enter critical region (lock "acquire")
    //... read/write x...
}                                  // exit critical region (lock "release")
```
-   Ordered atomics
```cpp
while(whose_turn != me){}          // enter critical region (atomic read "acquires" value)
// .. read/write x...
whose_turn = someone_else;         // exit critical region (atomic write "release")
```
-   Transactional memory
```cpp
atomic {                           // enter critical region
    // ... read/write x...
}                                  // exit critical region
```

-   **Key Rule: Code can't move _out_**
-   But code can safely move _in_

## Key Concepts: "Acquire" and "Release"
-   "One-way barriers"
![[atomic-weapons-acquire-release-diagram.png]]

-   Acquire fences allow code to move down.
-   Release fences allow code to move up.
-   Full fences block both.

-   **Plain Acq/Rel vs SC Acq/Rel**
    -   Plain acquire and release can move past each other.
    -   Sequentially consistent(SC) acquire and release can not move past each other.

-   **Fact of life**
    -   Memory synchronisation ==actively works against important modern hardware optimisations==
    -   Use it as little as possible

# Ordering: How? Mutexes, Atomics, and/or Fences

## Automating Acquire and Release
-   Don't write fences by hand.
-   Make the compiler write barriers for you by using "critical region" abstractions: Mutexes and
    `std::atomic<>` variables

```cpp
mut_x.lock();               // "acquire" mut_x
// ..read/write x...
mut_x.unlock();             // "release" mut_x
```

```cpp
while(whose_turn != me) {}  // read whose_turn
// .. read/write x...
whose_turn = someone_else;  // write whose_turn
```

## Controlling reordering 1): Use Mutexes
-   Use mutex locks to protect code that reads/writes shared variables.
-   Advantage:
    -   Locks acquire/release induce ordering and nearly all reordering/invention/removal
        weirdness vanishes
-   Disadvantage:
    -   Requries care on every use of the shared variables.
        -   Deadlocks can happen i.e. two threads taking locks in opposite order.
        -   Livelock can happen when locks try to "back off" (Chip 'n' Dale effect)

## Controlling Reordering 2): std::atomic<>
-   Special atomic types are automatically safe from reordering.
```cpp
atomic<int> flag1 = 0, flag2 = 0;
// From Dekker's algorithm
flag1 = 1;
if (flag2 != 0) {...}
```

-   Advantage: Just tag the variable, not every place it's used.
-   Disadvantage: Writing correct atomics code is harder than it looks

**Ordered Atomics**
-   Java and .NET = `volatile`. - **Always SC**
-   C++ atomic<T> - **Default SC**
-   Semantics and operations:
    -   Each individual read/write is **atomic**. No torn reads, no locking required.
    -   Each thread's reads/writes guaranteed to execute in order
    -   ==Special ops: Compare-and-swap==. Conceptually atomic execution of:
```cpp
T atomic<T>::exchange(T desired) {
    T oldval = this-> value;
    this->value == desired;
    return oldval;
}

bool atomic<T>::compare_exchange_strong(T& expected, T desired) {
    if (this->value == expected) {
        this->value = desired;
        return true;
    }
    expected = this->value;
    return false;
}
```

**compare_exchange: Weak and Strong**
-   In C++, compare-and-swap -> compare\_exchange\_<weak/strong>
    -   Means: "Am I the one who gets to change val from expected to desired?"
    -   Often written in loops -> ==CAS loop==

-   weak vs strong: Weak allows spurious failures
    -   Prefer `weak` when you're going to write a CAS loop anyway
    -   Almost always want `strong` when doing a single check

## Controlling Reordering 3): Fences and Ordered APIs
-   Fences are explicit 'sandbars' against reordering.
-   Disadvantages:
    -   Nonpoartable. Different types on different processors
    -   Tedious. Have to be written at every point of use
    -   Error-prone. Hard to reason
    -   Performance. Usually too heavy. _Standalone barriers are especially pessimized._
