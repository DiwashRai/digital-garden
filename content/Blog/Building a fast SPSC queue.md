---
title: "Building a fast SPSC Queue"
tags:
-   blog-post
---
Topics: [[Software Engineering]]  

---

## Introduction
Lockfree queues are all the rage now in C++ but just how fast are they? How fast can they get?
I will be building a basic one from scratch and gradually layer on any and all optimisations I can
find online. I will also compare the results to a simple mutex queue and other common lockfree
SPSC queues I find.

## Benchmarking existing SPSC queues for a baseline
Firstly, we need to find our bearings and see how fast(or slow) a standard mutex based queue is.
I will use a benchmark where numbers 1,2,3...N are pushed into a queue and then popped out. The
total sum will be tracked and checked for correctness at the end. How complicated can it be to
create a benchmark after all?

### Creating the benchmark code
Here is the function for the producer and consumer thread. A barrier has been used to ensure the
threads both start at the same time. The producer will be responsible for recording the start time
and the consumer will be responsible for recording the stop time at the end.

```cpp
// Producer thread
template <typename Queue>
void producer(Queue* queue, const unsigned num_items, Barrier* barrier, cycles_t* start) {
    barrier->wait();
    const unsigned stop_flag = num_items + 1;

    *start = __builtin_ia32_rdtsc();

    for (unsigned n = 1; n <= stop_flag; ++n) {
        queue->push(n);
    }
}

// Consumer thread
template <typename Queue>
void consumer(Queue* queue, const unsigned stop_flag, Barrier* barrier, cycles_t* end,
              uint64_t* sum) {
    barrier->wait();

    uint64_t local_sum = 0;
    for (;;) {
        unsigned item;
        queue->pop(item);
        if (item == stop_flag) break;
        local_sum += item;
    }
    *end = __builtin_ia32_rdtsc();
    *sum = local_sum;
}
```

### Quick note about thread affinity

I will also make sure that the thread affinity can be controlled. This is for two main reasons:
-   My laptop has an intel CPU with E-cores and P-cores. At the very least, I need to ensure that
    the threads are not running on the E-cores at least.
-   There will be a difference in performance between having the producer and consumer on
    **different logical processors** but **same core** vs **different logical processors** on
    **different cores**.

## std::queue + mutex baseline
Now that we have our benchmark, first things first, lets see how fast a mutex queue is in an SPSC
situation.

```sh

----------- SPSC Benchmarks -----------
#### MutexDequeQueue
->   warmup: 10,197,679 msg/s
->  1 Producer  1 Consumer - avg:   10,272,751 msg/s - min:   10,288,286 msg/s - max:   10,248,186 msg/s

```

So a mutex based queue using STL components can do 10mill/s. Bit of a meaningless number right now
so lets add in some common lockfree SPSC implementations.

## Adding some lockfree baselines
We will be adding the following lock free spsc queues as a baseline:
-   `boost::lockfree::spsc_queue` due to widespread awareness in C++ community.
-   `rigtorp::SPSCQueue` for the following reasons:
    -   Apparently faster than facebooks `folly::ProducerConsumerQueue`
    -   Erik Rigtorps implementation have been cited in papers as well as the fact that his MPMC is
        used by game companies and by companies in their low latency trading infrastructure.

```sh

----------- SPSC Benchmarks -----------
#### MutexDequeQueue
->   warmup: 9,912,733 msg/s
->  1 Producer  1 Consumer - avg:    9,856,895 msg/s - min:    9,909,230 msg/s - max:    9,810,655 msg/s

#### BoostLockFreeSPSCQueue
->   warmup: 153,921,148 msg/s
->  1 Producer  1 Consumer - avg:  149,095,290 msg/s - min:  151,704,424 msg/s - max:  144,560,131 msg/s

#### RigtorpSPSCQueue
->   warmup: 261,363,316 msg/s
->  1 Producer  1 Consumer - avg:  260,011,027 msg/s - min:  267,892,886 msg/s - max:  253,488,347 msg/s

```

Now that we have some baselines, lets implement an SPSC and gradually layer on optimisations to
see the impact of each one.

## Lockfree SPSC no optimisations

Lets start with an extremely simple lock free SPSC implementation where we have a ring buffer
on the stack and two atomic head and tail indexes.

```cpp
namespace alpha {

template <typename T, unsigned SIZE, T NIL>
class spsc {
public:
    using value_type = T;
    using size_type = std::size_t;

    spsc() = default;
    spsc(spsc&) = delete;
    spsc& operator=(spsc&) = delete;

    // push
    void push(const T& item) {
        while (!try_push(item)) _mm_pause();
    }

    bool try_push(const T& item) {
        auto tail = tail_.load();
        auto head = head_.load();

        if (full(tail, head)) return false;

        new (&data_[tail % SIZE]) T(item);
        tail_.store(tail + 1);
        return true;
    }

    // pop
    void pop(T& item) {
        while (!try_pop(item)) _mm_pause();
    }

    bool try_pop(T& item) {
        auto head = head_.load();
        auto tail = tail_.load();

        if (empty(tail, head)) return false;

        item = data_[head % SIZE];
        head_.store(head + 1);
        return true;
    }

private:
    [[nodiscard]] static bool full(const size_type tail, const size_type head) {
        return tail - head >= SIZE;
    }

    [[nodiscard]] static bool empty(const size_type tail, const size_type head) {
        return tail <= head;
    }

    T data_[SIZE];
    std::atomic<size_type> head_{0};
    std::atomic<size_type> tail_{0};
};
}  // namespace alpha

```
