---
title: "Concurrency Cheat Sheet"
tags:
-   molecule
---
Topics: [[Software Engineering]]  
Reference:
-   [[C++ Concurrency in action]] - [[Anthony Williams]]  
-   [Back To Basics: C++ Concurrency - David Olsen](https://www.youtube.com/watch?v=8rEGu20Uw4g)

---

## Definition

|   "Multiple logical threads of execution with some inter-task dependencies"  

i.e. Doing things at the same time when some of the things need to happen before others

**Differences to parallelism**

|   "Multiple logical threads of execution with ==NO== inter-task dependencies"

## Why use concurrency
Performance. Use all the cpu cores to do work faster.

## Sharing data between threads

### mutexes
See [[Talking Stick Mental Model]] for an intuitive way to think about how mutexes work. Essentially
you lock the mutex before using the shared resource, then unlock it when you are done.


## Busy waiting (polling)
Also known as spin-waiting, is when a thread or process continuosly checks if a condition has been
met without yielding control. Usually resource is polled at regular intervals, which can
lead to inefficient CPU usage as the processing time is spent essentially doing nothing but
waiting.  

std::condition_variable is a dedicated synchronisation primitive that allows threads to wait for
a condition without busy waiting. It is used alongside a mutex to safely access shared data. The
waiting thread calls the `wait()` function which accepts the lock and a function that acts as a
predicate. If the predicate returns false, then it locks the mutex. Threads can then be unlocked by
calling `notify_one()` or `notify_all()`.



