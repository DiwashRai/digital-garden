---
title: "shared_ptr and weak_ptr"
tags:
-   atom
---
Reference:  
Topics: [[Software Engineering]]  

---

`std::shared_ptr` is a smart pointer that manages the shared ownership of a dynamically allocated
resource. It utilises reference counting to track how many shared_ptr instances are pointing to
the same object and deletes it when the count goes to 0.

`std::weak_ptr` is a smart pointer that provides a **non-owning** reference to an object that is
managed by `std::shared_ptr`. It's main use is to prevent cyclic references and provide a safe way
to access objects managed by `std::shared_ptr`

```cpp
template<typename T>
class shared_ptr
{
private:
    // sizeof = 16
    T* ptr;
    ControlBlock* pControl;

    struct ControlBlock
    {
        std::size_t = shared_count;
        std::size_t = weak_count;
        T* managed_object;
    };
};
```

```cpp
template<typename T>
class shared_ptr
{
private:
    // sizeof = 16
    T* ptr;
    ControlBlock* pControl;

    struct ControlBlock
    {
        std::size_t = shared_count;
        std::size_t = weak_count;
        T* managed_object;
    };
};
```
