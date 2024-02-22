---
title: "Observer Design Pattern"
tags:
- atom
- design-pattern
---
Reference:  
Topics:  

---

### summary
The 'Observer' design pattern is a behavioural design pattern that allows an object, known as the
subject, to automatically notify it's observers (other objects) about any changes to it's state.

In this pattern, a single 'subject' will have many dependants. Whenever a change happens, all of it's
dependants are notified. This means that the observer pattern is used when there is a one-to-many
relationship between objects.

### Implementation details
The observer design pattern works as follows:
-   Start with a subject object whose state changes should be tracked and notified.
-   Create observer objects that will be notified when the state of the subject changes.
-   Attach the observers to the subject. The subject will maintain a list of all attached observers.
-   When the state of the subject changes, it notifies all attached observers by calling their
    update method.
-   Observers use these updates to synchronise their state with the state of the object, thus
    maintaining consistency.

### Code example
**Observer and Subject interfaces**
```cpp
class Observer {
public:
    virtual ~Observer() = default;
    virtual void update(float temp, float humidity, float pressure) = 0;
};

class Subject {
public:
    virtual ~Subject() = default;
    virtual void registerObserver(Observer* o) = 0;
    virtual void removeObserver(Observer* o) = 0;
    virtual void notifyObservers() = 0;
};

```

**Concrete subject class `WeatherStation`**
```cpp
#include <vector>

class WeatherStation : public Subject {
private:
    std::vector<Observer*> observers;
    float temperature;
    float humidity;
    float pressure;

public:
    void registerObserver(Observer* o) override {
        observers.push_back(o);
    }

    void removeObserver(Observer* o) override {
        observers.erase(std::remove(observers.begin(), observers.end(), o), observers.end());
    }

    void notifyObservers() override {
        for (Observer* observer : observers) {
            observer->update(temperature, humidity, pressure);
        }
    }

    void measurementsChanged() {
        notifyObservers();
    }

    void setMeasurements(float temperature, float humidity, float pressure) {
        this->temperature = temperature;
        this->humidity = humidity;
        this->pressure = pressure;
        measurementsChanged();
    }
};

```

**Concrete observer class `Display`**
```cpp
#include <iostream>

class Display : public Observer {
public:
    void update(float temp, float humidity, float pressure) override {
        std::cout << "Display updated with temp: " << temp << ", humidity: " << humidity << ", pressure: " << pressure << std::endl;
    }
};

```

**Usage**
```cpp
int main() {
    WeatherStation weatherStation;
    Display display;

    weatherStation.registerObserver(&display);
    weatherStation.setMeasurements(25.0, 65.0, 1013.0);

    return 0;
}

```
