---
title: "Talking stick mental model for mutexes"
tags:
-   atom
-   mental-model
---
Reference:
-   [Back To Basics: C++ Concurrency - David Olsen](https://www.youtube.com/watch?v=8rEGu20Uw4g)

Topics:
-   [[Software Engineering]]  

---

## Summary
Think of a mutex as a talking stick in a group therapy session. The purpose of the stick (mutex),
is to avoid people talking over/interrupting each other. As such, there is a rule that you are
only allowed to speak if you have the stick. If you would like to speak, you need to wait until
the current speaker is done talking (releasing the mutex) and then you have to take the stick
yourself (locking the mutex again).

## Insights and takeaways
It is just an analogy to simplify thinking about concurrency issues and race conditions. Other
analogies could be:
-   Aeroplanes and a shared runway. Flying is a parallel activity, but landing/taking off uses
    a shared resource.
-   Toilet on a plane. The lock on the door is like the mutex that prevents someone else coming
    in. And I supposed a 'deadlock' can be caused if the door gets locked but no one is inside
    to unlock it after they're done.
