---
title: "Singleton Design Pattern"
tags:
- atom
- design-pattern
---
Reference:  
Topics:  

---

### Summary
Is a type of creational pattern that ensures a class has only one instance and provides global access
to this instance. The name comes from the fact that it restricts instantiation of a class to a single
object.

The pattern is commonly used in situations where a class should only have a single instance such as
managing database connections, logging, file management, printer spoolers, and providing a global
state for configuration used across the application.

### Implementation details
A singleton can be implemented by making the constructor of a class private and defining a static
method that creates a static instance of the class.

### Code example

```cpp
#include <iostream>

class Singleton
{
public:
    static Singleton& getInstance()
    {
        static Singleton instance;
        return instance;
    }

private:
    Singleton()
    {
        std::cout << "Singleton constructor\n";
    }
    ~Singleton() = default;
    Singleton(const Singleton&) = delete;
    Singleton& operator=(const Singleton&) = delete;
};

int main()
{
    // constructor should only be called once
    Singleton& s1 = Singleton::getInstance();
    Singleton& s2 = Singleton::getInstance();

    // memory addresses should match
    std::cout << &s1 << '\n';
    std::cout << &s2 << '\n'; 

    return 0;
}
```