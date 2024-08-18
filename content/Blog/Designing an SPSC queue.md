---
title: "Designing an SPSC Queue"
tags:
-   blog-post
---
Topics: [[Software Engineering]]  

---

## Rules/guidelines to remember

-   Rule of 5 or 0
-   Implement a no throw swap if possible

## The interface

```cpp

template<typename T>
class spsc_queue
{
public:
    explicit spsc_queue(std::size_t size);

    bool push(T const& item);
    bool pop(const& item);
    void reset();
};

```
