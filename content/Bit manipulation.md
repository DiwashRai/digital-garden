---
title: "Bit manipulation"
tags:
- atom
---
Topics: [[Coding techniques]]  
Reference: Leetcode  

---

## Adding 1 to any number
When you add 1 to any number N, all bits to the right of **rightmost 0** (including the rightmost
zero) gets **flipped**. E.g.  
```
43: 101011
44: 101100
```
**Intuition**  
This should be obvious as adding 1 will flip the first column which will in turn flip the next and
so forth until a column that is zero is encountered which can be flipped without another carry
over 1 to the next column.

## Turn off rightmost set bit
if input is `n`, do bitwise and with `n-1`.
```
n = n & (n - 1);
// shortened to
n &= (n - 1);
```

