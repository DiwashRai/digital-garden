---
title: "Decorator Design Pattern"
tags:
- atom
- design-pattern
---
Reference:  
Topics:  

---

### Summary
The decorator design pattern allows behaviour to be added to an object, either statically or
dynamically, without affecting the behaviour of other objects from the same class. It is a structural
design pattern that involves a set of decorator classes that are used to wrap concrete components.
Decorator classes have the same interface as the components they decorate.

### Implementation details
The decorator pattern works as follows:
- You start with the basic object. This is the lowest level and will have extra functionality
layered onto it.
- You create another object with a reference to the first object. This second object will decorate
the basic object i.e. it 'wraps' the original object.
- The decorator object will mirror the type of the original object, but will add or override
behaviour.
- You can now layer on additional decorators, each one referencing the previous layer. This allows
features to be built up incrementally, at runtime.

### Code example
**Define abstract base class(interface) `Coffee`**
```cpp
class Coffee {
public:
    virtual ~Coffee() = default;
    virtual double getCost() = 0;
    virtual std::string getDescription() = 0;
};
```

**Create `BasicCoffee` that implements `Coffee` interface**
```cpp
class BasicCoffee : public Coffee {
public:
    double getCost() override {
        return 1.0;
    }

    std::string getDescription() override {
        return "Basic coffee";
    }
};
```

**Create abstract decorator `CoffeeDecorator` that also implements `Coffee` interface**
```cpp
class CoffeeDecorator : public Coffee {
protected:
    std::unique_ptr<Coffee> m_DecoratedCoffee;
public:
    CoffeeDecorator(std::unique_ptr<Coffee> coffee) : m_DecoratedCoffee(std::move(coffee)) {}
    virtual ~CoffeeDecorator() = default;
};
```

**Create specific decorators**
```cpp
class MilkDecorator : public CoffeeDecorator {
public:
    MilkDecorator(std::unique_ptr<Coffee> coffee) : CoffeeDecorator(std::move(coffee)) {}

    double getCost() override {
        return m_DecoratedCoffee->getCost() + 0.5;
    }

    std::string getDescription() override {
        return m_DecoratedCoffee->getDescription() + ", Milk";
    }
};

class CaramelDecorator : public CoffeeDecorator {
public:
    CaramelDecorator(std::unique_ptr<Coffee> coffee) : CoffeeDecorator(std::move(coffee)) {}

    double getCost() override {
        return m_DecoratedCoffee->getCost() + 0.75;
    }

    std::string getDescription() override {
        return m_DecoratedCoffee->getDescription() + ", Caramel";
    }
};
```

**Use decorators to decorate the basic object**
```cpp
int main() {
    auto coffee = std::make_unique<BasicCoffee>();
    coffee = std::make_unique<MilkDecorator>(std::move(coffee));
    coffee = std::make_unique<CaramelDecorator>(std::move(coffee));

    std::cout << coffee->getDescription() << " costs $" << coffee->getCost() << std::endl;
    return 0;
}
```