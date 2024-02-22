---
title: "Dynamic Programming"
tags:
- atom
---
Topics: [[Coding techniques]]  
Reference: LeetCode DP module  

---

## Intro
Dynamic programming can systematically and efficiently explore all possible solutions to a problem.
The wide variety of problems it can solve often have the following characteristics:
-   Problem can be broken down into =='overlapping subproblems'== - smaller versions of the
    original problem re-used multiple times.
-   The problem as an =='optimal substructure'== - an optimal solution can be formed from optimal
    solutions to the overlapping subproblems.

## Top-down and Bottom-up

### Bottom-up - Tabulation
Bottom-up is implemented iteratively and starts with the base case. The base cases for the
fibonacci sequence are `F(0) = 0` and `F(1) = 1`. With bottom-up, we use these bases cases to
calculate F(2), and then use that to calculate F(3) and so on.

### Top-down - Memoization
Top-down is implemented with recursively and made efficient with memoization. If we want to find
the nth fibonacci number, we try to find this by finding F(n - 1) and F(n - 2). This defines a
recursive pattern until we reach the base case which is `F(0) = F(1) = 1`.

![](Pasted%20image%2020230730134659.png)

The recursive tree shows that F(2) would have to be calculated 3 times even if we're calculating
a small fibonacci number. To reduce the repeated computation we ==memoize== the results.

After we calculate F(2) we can store the results somewhere (usually a hashmap). In the future
when we need to find F(2), we can just refer to the calculated value instead of re-calculating it
all over again.

### Which is better?
DP can be implemented with either method and there can be reasons for choosing one over the other.
The main advantage for either is:
-   **Bottom up:** Runtime is usually faster as iteration does not have overhead of recursion.
-   **Top down:** Usually much easier to write. This is because the ordering of subproblems does
    not matter, whereas with tabulation, we need to go through a logical order of subproblems.

## Framework
To solve a DP problem, 3 things are necessary:
1.  A function or data structure that will compute/contain the answer to the problem for every
    given state.
2.  A recurrence relation to transition between states.
3.  Base cases so our recurrence relation does not go on infinitely.
