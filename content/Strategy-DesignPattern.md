---
title: "Strategy Design  Pattern"
tags:
-   atom
-   design-pattern
---
Reference:  
Topics:  

---

### Summary
The strategy design pattern is a behavioural design pattern that allows you to define a 'family' of
algorithms, encapsulate each one, and make them interchangeable. This means the strategy pattern
allows the algorithm to vary independently from the clients that use it.

To achieve this, the algorithm is removed from the host class and placed in a separate class that
implements a strategy interface. The algorithm can now be passed in and vary at run time.

The strategy design pattern also allows closer adherence to the Open/Closed principle. The host
class can now be extended to use a new algorithm without modifying the host class itself.

### Implementation details
-   **Context:** This is the class that has a method that uses a strategy object to complete a task.
    It does not implement the task by itself, but takes help from the strategy object to perform the
    task.
-   **Strategy interface:** An interface common to all supported algorithms or strategies. Context
    uses this interface to call the method provided by the strategy class.
-   **Concrete strategies:** These are the classes that implement the strategy interface. They
    provide the concrete implementation of the algorithm.

### Code example

```cpp
#include <iostream>
#include <string>

// Strategy Interface
class CompressionStrategy {
public:
  virtual ~CompressionStrategy() = default;
  virtual void compress(const std::string& file) const = 0;
};

// Concrete Strategy 1
class ZipCompressionStrategy : public CompressionStrategy {
public:
  void compress(const std::string& file) const override {
    std::cout << "Compressing file '" << file << "' using ZIP compression.\n";
  }
};

// Concrete Strategy 2
class RarCompressionStrategy : public CompressionStrategy {
public:
  void compress(const std::string& file) const override {
    std::cout << "Compressing file '" << file << "' using RAR compression.\n";
  }
};

// Context
class CompressionContext {
private:
  CompressionStrategy* strategy_;

public:
  CompressionContext(CompressionStrategy* strategy) : strategy_(strategy) {}
  
  ~CompressionContext() {
    delete strategy_;
  }

  void setCompressionStrategy(CompressionStrategy* strategy) {
    delete strategy_;
    strategy_ = strategy;
  }

  void compressFile(const std::string& file) const {
    strategy_->compress(file);
  }
};

int main() {
  CompressionContext context(new ZipCompressionStrategy);
  context.compressFile("example.txt");  // Output: Compressing file 'example.txt' using ZIP compression.

  context.setCompressionStrategy(new RarCompressionStrategy);
  context.compressFile("example.txt");  // Output: Compressing file 'example.txt' using RAR compression.

  return 0;
}

```
