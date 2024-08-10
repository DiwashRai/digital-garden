---
title: "Lock-Free SPSC Queue"
tags:
-   atom
---
Topics: [[Software Engineering]]  
Reference: [Single Producer Single Consumer Lock-free FIFO from the Ground Up](https://www.youtube.com/watch?v=K3P_Lmq6pw0)  

---

## Optimisations

**Operations/sec**
-   mutex (not lock free)
    -   5,756,232
-   Basic/Naive. No memory order specified.
    -   12,569,860
-   Relaxed atomics + no false sharing.
    -   42,133,450
-   Cached cursor/index
    -   164,687,821
-   Proxies
    -   162,540,387

