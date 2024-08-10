---
title: "Glossary of Terms"
tags:
-   molecule
---
Topics:  
-   [[Software Engineering]]
Reference:  

---

## Concurrency
-   **Atomics**
-   **CAS (Compare and swap)**
-   **Lock-Free**: Algorithm or data-structure where during execution, at least one thread will
    always make progress.
    -   _Alternative wording:_ Lock-free algorithms have the property that there is always
        system-wide progress.
    -   _Informal definition:_ Doesn't use mutexes.
-   **Wait-Free:** Algorithm where each thread makes progress regardless of external factors such
    as other threads blocking and contention.
    -   Typically does not use cycles that can be affected by other threads (CAS loops).
-   **Obstruction-Free**

-   **Race condition**
-   **Data race**
-   **Producer-Consumer problem**
-   **ABA Problem**

-   **Barrier**

-   **Memory model**
-   **Context switch**

-   **Coroutine**


