---
title: "Code review protocols"
tags:
-   molecule
---
Topics: [[Software Engineering]]  
Reference:  

---

## General approach
The general idea is to do multiple passes, each time focusing on a separate area. You don't actually
have to do multiple passes if you think you can do some/all of them simultaneously.
-   Syntax/Spelling/silly errors (e.g. todos or commented out code left in)
    -   Large parts of this can most likely and should be automated.
-   Coding style/effective use of language features (e.g. C++ structured bindings, lambdas etc)
-   Readability and maintainability
    -   Naming conventions as well as the names themselves
    -   Code structure
-   Documentation and test coverage
-   Logical correctness
-   Consider alternative approaches that might be better
    -   Do this last as during the previous steps you will have deepened your understanding of the
        code.

