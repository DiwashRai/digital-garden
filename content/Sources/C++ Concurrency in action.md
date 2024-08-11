
---
title: "C++ Concurrency in action (2nd edition)"
tags:
-   source
-   textbook
---
Author: [[Anthony Williams]]  
Topics: [[Software Engineering]]  

---

## 1 - Hello, world of concurrency in C++!
**chapter contents** 
-   What is meant by concurrency and multithreading.
-   Why you might want to use concurrency/multithreading
-   History of concurrency support in C++
-   What a simple multithreaded C++ program looks like

### What is concurrency
-   "A single system performing multiple independent actions in parallel, rather than
    sequentially"(pg 2)
-   Historically it used to be more of an illusion as single threaded computers switch tasks many
    times per second. Now we have processors genuinely capable of concurrency.
-   _Hardware concurrency_ is when a machine has multiple processors or multiple cores within a
    processor or both and are genuinely capable of running more than one task in parallel.
-   Even a system with genuine hardware concurrency can have more tasks than the hardware can
    run in parallel. This means that task switching will happen which can lead to irregular
    scheduling.

#### Approaches to concurrency
-   Multi-process vs multi-threaded.
-   Analogy: Software project with 2 developers.
    -   Multi-process: Separate offices.
        -   No shared equipment.
        -   Less disturbances.
        -   Communication overhead increased.
        -   Overhead of separate offices and buying equipment for both.
    -   Multi-threaded: Same office.
        -   Very easy to communicate/discuss.
        -   Harder to concentrate.
        -   Sharing resources.

**Concurrency with multiple processes**
-   Done with interprocess communication channels accomdated via the OS e.g. signals, sockets, files
    pipes etc.

**Concurrency with multiple threads**
-   Much more light weight.
-   All threads share same address space so all data can be accessed directly by all threads.

**Concurrency vs parallelism**
-   Quick note: Largely overlapping meanings but often used in slightly different contexts.
    -   Parallelism: primary concern is using available hardware to increase performance of bulk
        data processing.
    -   Concurrency: primary concern is separation of concerns, or responsiveness.

### Why use concurrency?
Almost only two reasons ever to use concurrency:
-   Separation of concerns: e.g. background tasks thread, UI thread, IO thread etc.
-   Performance (task and data parallelism): 
    -   _Task parallelism_: Split single task into parts and have each thread run each part in
        parallel.
    -   _Data parallelism_: Different threads run the same algorithm for different parts of the
        data.

#### When not to use concurrency
-   Code using concurrency is harder to understand and maintain.
-   Additional complexity can cause more bugs.
-   Performance gain not be as large as expected. Measure.
-   Launching a thread has inherent overhead for the OS which can make the system as a whole run
    slower.
-   Each thread requires some memory(1MB is apparently typical) which can use up resources.
    -   Example where this is not suitable woulf be for client/server applications if each
        connection was to run in its own thread.
-   The more threads there are, the more context switching will have to take place.

## 2 - Managing threads
**Chapter contents**
-   Starting threads.
-   Waiting for threads to finish vs leaving it to run.
-   Uniquely identifying threads.

### Basic thread management
-   Starting a thread using the STL will always boil down to constructing a `std::thread` object.

```cpp

// constructing a thread with a normal function
void do_some_work();
std::thread my_thread(do_some_work);

// constructing a thread with a functor
class background_task
{
public:
    void operator()() const
    {
        do_something();
        do_something_else();
    }
}
background_task f;
std::thread my_thread(f);

// constructing a thread with a lambda
std::thread my_thread([]{
    do_something();
    do_something_else();
});

```
`std::thread` works with any callable type.

-   Once a thread has been started, you need to explicitly decide to wait for it to finish(`join`)
    or to leave it to run on its own(`detach`). If this decision is note made before `std::thread`
    object is destroyed, `std::terminate` will be called terminating your program.
-   It is therefore imperative to ensure the thread is correctly joined or detached, _even_ in the
    presence of _exceptions_(_thread guards_).

**Waiting in exceptional circumstances**
-   You can use a `try/catch` block although it can be easy to get the scope slightly wrong.
```cpp

struct func
{
    int& i;
    func(int& i_) : i(i_){}
    void operator()()
    {
        for(unsigned j = 0; j < 1000000; ++j)
        {
            do_something(i);
        }
    }
};

void f()
{
    int some_local_state = 0;
    func my_func(some_local_state);
    std::thread t(my_func);
    try
    {
        do_something_in_current_thread();
    }
    catch(...)
    {
        t.join();
        throw;
    }
    t.join();
}
```
-   A simple way to ensure the thread is joined for all exit paths is using _RAII_;
```cpp

class thread_guard
{
    std:;thread& t;
public:
    explicit thread_guard(std::thread& t_) : t(t_){}
    ~thread_guard()
    {
        if (t.joinable()) // error to call join more than once on a thread.
            t.join();
    }
    thread_guard(thread_guard const&) = delete;
    thread_guard operator=(thread_guard const&) = delete;
};

```
-   Copy-constructor and copy-assignment marked `delete` as copying or assigning this sort of
    object is dangerous.

### Passing arguments to a thread function

## 3 - Sharing data between threads
**Chapter contents**

## 4 - Synchronizing concurrent operations
**Chapter contents**

## 5 - The C++ memory model and operations on atomic types
**Chapter contents**

## 6 - Designing lock-based concurrent data structures
**Chapter contents**

## 7 - Designing lock-free concurrent data structures
**Chapter contents**

## 8 - Designing concurrent code
**Chapter contents**

## 9 - Advanced thread management
**Chapter contents**

## 10 - Parallel algorithms
**Chapter contents**

## 11 - Testing and debugging multithreaded applications
**Chapter contents**
