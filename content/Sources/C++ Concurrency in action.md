---
title: C++ Concurrency in action (2nd edition)
tags:
  - source
  - book
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

-   Arguments can be passed to a std::thread function like so:

```cpp

void f(int i, std::string& str);
std::thread t(f, 3, "hello"); // note use of char const*

```

-   ==By default, arguments are copied into the thread objects internal storage by value==.
-   This means that if you pass arguments as a pointer, the real variable being pointed to might
    have gone out of scope.

```cpp

void f(int i, std:;string const& str);
void oops(int some_param)
{
    char buffer[1024];
    sprintf(buffer, "%d", some_param);
    std::thread t(f, 3, buffer); // buffer passed in as char*
    t.detach(); // detached and buffer will go out of scope
}

```
-   This can be fixed by doing the following instead: `std::thread t(f, 3, std::string(buffer));`
-   **A problem** that now happens due to the fact that std::thread takes parameters in by
    value is that if the function takes in parameters by ==reference==, you cannot simply provide
    a value hoping it takes it by reference.
-   Since it is always by value, you need to wrap variables meant to be passed by reference in a
    reference wrapper like so: `std::thread t(update_widget, w, std::ref(data));`
-   Finally, there is another peculiar scenario with objects that can only be moved e.g.
    `std::unique_ptr`.
-   In this situation you just have to use `std::move`.

```cpp

void process_big_obj(std::unique_ptr<big_object>);
std::unique_ptr<big_object> p(new big_object);
std::thread t(process_big_obj, std::move(p));

```

### Transferring ownership of a thread

-   `std::thread` can be moved into another `std::thread` object.
-   However, if the target object already had an associated thread, `std::terminate` will be
    called to be consistent with the destructor.
-    One benefit of move support for `std::thread` is that you can create a 'scoped_thread' class
    that joins automatically when going out of scope.
-   ==This is now in C++20 as std::jthread==.

### Choosing the number of threads at runtime
-   `std::thread::hardware_concurrency()` returns an indication of the number of threads that a
    system can run. Returns 0 if information not available.

### Identifying threads
-   `std::thread::id` can be retrieved using `getId()` member function of a `std::thread` or by
    calling `std::this_thread::get_id()`.

## 3 - Sharing data between threads
**Chapter contents**
-   Problems with sharing data between threads
-   Protected data with mutexes
-   Alternative way to protected shared data

### Problems with sharing data between threads
-   All due to ==modifying data==. If all shared data is _read-only_ there's no problem.

> [!tip] Invariants
> A useful concept to reason about code are invariants. These are statements that are always true
> about a data structure. These are often broken during an update. e.g. variable containing the
> number of items in a list whilst the list is being updated.

-   The simplest potential issue with modifying shared data is broken invariants.
-   Breaking the invariant for a doubly linked list could result in a node being skipped if one
    thread is reading left to right _OR_ permanently corrupting the data structure and crashing
    the program if one thread is trying to delete the right most node.
-   This is one of the most common cause of bugs in concurrent code: a **Race condition**

**Race conditions**
-   Alternative word for problematic race condition is a **data race**.
-   In concurrency, a race condition is when the outcome depends on the relative ordering of
    operations of two or more threads.
-   Race conditions can be ==hard to find== and ==hard to duplicate== because the window of
    opportunity is small. Consecutive cpu instructions have a low likelihood of the problem
    showing up, but as the load on the system increases, the chance of problematic execution
    sequences showing up increases.

**Avoiding problematic race condtions**
-   Simplest way to avoid race conditions is to ensure only a _single_ thread can see the
    intermediate states during the modification of a data structure (mutexes)
-   Another option is to modify the data structure and the invariants so the modifications are
    done as a series of indivisible changes, each preserving the invariants (lock-free programming)

### Protecting shared data with mutexes
-   Mutexes can allow you to mark pieces of code that access the data structure as _mutually
    exclusive_, so that if two threads tried to access the data structure, one would have to wait.
-   Mutexes are the most general data-protection mechanism but still have their problems:
    -   _deadlocks_
    -   Protecting too much or too little data.

**Using mutexes in C++**
```cpp

std::list<int> some_list;
std::mutex some_mutex;

void add_to_list(int val)
{
    std::lock_guard<std::mutex> guard(some_mutex);
    some_list.push_back(val);
}

bool list_contains(int value_to_find)
{
    std::lock_guard<std::mutex> guard(some_mutex);
    return std::find(some_list.begin(), some_list.end(), valuee_to_find)
        != some_list.end();
}
```

**Structuring code for protecting shared data**
-   _Don't pass pointers and references to proteced data outside the scope of the lock, whether
    by returning them from a function, storing them in visible memory, or passint them as
    arguments to to user supplied functions_

**Spotting race conditions inherent in interfaces - a case study**
-   Consider a stack data structure with 5 operations:
    -   `push()`
    -   `pop()`
    -   `top()`
    -   `size()`
    -   `empty()`
-   Even if each individual operation is safe, this interface is inherently subject to race
    conditions.
-   _Problem_: results of `empty()` and `size()` can't be relied on ever in a multi-threaded
    context.
-   The problem is inherent to the design of the interface so the solution has to be some sort of
    change to the interface.
-   A bad solution would be to delcare `top()` can throw an exception if the stack turns out
    to be empty, even if you just checked with a call to `empty()`. This means that any empty
    call checks are just optimisatitons to avoid an exception, not a necessary part of the design.
-   ==Another more sinister race condition==. This time between possible calls to `top()` and
    `pop()`

![[push-pop-stack-race-condition.png]]

-   If you analyse this sequence of events, you will notice that the same value is read by both
    threads in `top()` and then two values are discarded from the stack. This is more insidious
    as there might never be anything obviously going wrong and the consequences of the bug are
    felt far from the cause.
-   A function that combines `top()` and `pop()` calls can lead to issues if the copy-constructor
    for the object on the stack throws an exception. If `pop()` returned the popped value as well
    as remove it from the stack, you have a problem where the popped value is only returned after
    the stack has been modified. If an exception is thrown when copying the data, the popped data
    is lost.
-   **Option 1: pass in a reference**
    -   This works well as it requires the calling code to construct an instance of the stack's
        value prior to the call.
    -   Sometimes not practical as constructing an instance could be expensive.
    -   Sometimes not possible as the construction could require parrameter's that are not
        available.
    -   Requires stored type to be assignable.
-   **Option 2: Require a no-throw copy constructor or move constructor**
    -   A value returning `pop()` only has a problem dur to exceptions. There are types that have
        copy constructors that are nothrow and even more types that have move constructors that
        don't throw exceptions even if the copy constructor does.
    -   This is safe but not ideal as it is quite limiting as there are many user-defined types
        with copy constructors that can throw and don't have move constructors.
-   **Option 3: Return a pointer to the popped item**
    -   Pointers can be freely copied without throwing an exception.
    -   Disadvantage is having to manage the memory to the allocated object. For simple types such
        as integers, the overhead can exceed the cost of return by value.
    -   For an interface that uses this option `std::shared_ptr` would be a good choice.
-   **Option 4: Provide both option 1 and either option 2 or 3**
    -   Flexiblity is a valuable trait in generic code. If option 2 or 3 is chosen, it should be
        easy to also provide 1.

**Locking granularity**
-   As the `top()` and `pop()` issues shows, problematic race conditions can arise in interfaces
    due to too small a granularity.
-   However, if we use too large of a granularity such as a global mutex for all shared data, we
    can then wipe out all the performance benefits of concurrency.
-   Another issue of fine-grained locking schemes is that you might need more than one mutex to
    protect all operations.
-   If you end up having to lock two or more mutexes for a single operation, you can then run into
    another problem: _deadlock_

**Deadlock: the problem and a solution**

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
