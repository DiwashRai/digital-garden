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

## Creating the baseline benchmarks for comparison

### std::queue + mutex baseline
Now that we have our benchmark, first things first, lets see how fast a mutex queue is in an SPSC
situation.

```sh

----------- SPSC Benchmarks -----------
#### MutexDequeQueue
-> warmup:  10,197,679 msg/s
->    avg:  10,272,751 msg/s - min:   10,288,286 msg/s - max:   10,248,186 msg/s

```

So a mutex based queue using STL components can do 10mill/s. Bit of a meaningless number right now
so lets add in some common lockfree SPSC implementations.

### Adding some lockfree baselines
We will be adding the following lock free spsc queues as a baseline:
-   `boost::lockfree::spsc_queue` due to widespread awareness in C++ community.
-   `rigtorp::SPSCQueue` for the following reasons:
    -   Apparently faster than facebooks `folly::ProducerConsumerQueue`
    -   Erik Rigtorps implementation have been cited in papers as well as the fact that his MPMC is
        used by game companies and by companies in their low latency trading infrastructure.

```sh

----------- SPSC Benchmarks -----------
#### MutexDequeQueue
-> warmup:  9,912,733 msg/s
->    avg:  9,856,895 msg/s - min:    9,909,230 msg/s - max:    9,810,655 msg/s

#### BoostLockFreeSPSCQueue
-> warmup:  153,921,148 msg/s
->    avg:  149,095,290 msg/s - min:  151,704,424 msg/s - max:  144,560,131 msg/s

#### RigtorpSPSCQueue
-> warmup:  261,363,316 msg/s
->    avg:  260,011,027 msg/s - min:  267,892,886 msg/s - max:  253,488,347 msg/s

```

Now that we have some baselines, lets implement an SPSC and gradually layer on optimisations to
see the impact of each one.


## Gradually layering on optimisations

### Lockfree SPSC no optimisations

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

This is an extremely naive implementation where even the memory order has not been specified which
means it defaults to sequentially consistent. Lets see what the result is.

```sh

----------- SPSC Benchmarks -----------
#### MutexDequeQueue
-> warmup:  9,854,112 msg/s
->    avg:  9,922,185 msg/s - min:   10,036,260 msg/s - max:    9,774,526 msg/s

#### BoostLockFreeSPSCQueue
-> warmup:  148,237,964 msg/s
->    avg:  149,004,250 msg/s - min:  149,993,539 msg/s - max:  147,950,625 msg/s

#### RigtorpSPSCQueue
-> warmup:  266,873,073 msg/s
->    avg:  265,607,559 msg/s - min:  268,235,238 msg/s - max:  262,831,869 msg/s

#### Naive implementation
-> warmup:  40,786,175 msg/s
->    avg:  40,317,279 msg/s - min:   40,843,627 msg/s - max:   39,813,059 msg/s

```

A naive implementation already seems to beat a mutex based implementation by 4x which is good to
know but we can get an easy win by specifying specific memory orders and not using the default.

### Relaxed and no-seq_cst memory order

We will change the memory order in the try_push and try_pop functions.

```cpp

bool try_push(const T& item) {
    auto tail = tail_.load(std::memory_order::relaxed);
    auto head = head_.load(std::memory_order::acquire);

    if (full(tail, head)) return false;

    new (&data_[tail % SIZE]) T(item);
    tail_.store(tail + 1, std::memory_order::release);
    return true;
}

bool try_pop(T& item) {
    auto head = head_.load(std::memory_order::relaxed);
    auto tail = tail_.load(std::memory_order::acquire);

    if (empty(tail, head)) return false;

    item = data_[head % SIZE];
    head_.store(head + 1, std::memory_order::release);
    return true;
}

```

By just specifying the memory order we gone from about 40M Msg/s to about 146M Msg/s. That is
about 3.65x. Not bad.

```sh

----------- SPSC Benchmarks -----------
#### MutexDequeQueue
-> warmup:    9,972,891 msg/s
->    avg:    9,932,316 msg/s - min:    9,994,098 msg/s - max:    9,866,449 msg/s

#### BoostLockFreeSPSCQueue
-> warmup:  150,199,507 msg/s
->    avg:  148,697,331 msg/s - min:  151,043,684 msg/s - max:  145,576,293 msg/s

#### RigtorpSPSCQueue
-> warmup:  267,328,749 msg/s
->    avg:  258,840,751 msg/s - min:  261,255,156 msg/s - max:  255,489,424 msg/s

#### Naive implementation
-> warmup:   41,014,949 msg/s
->    avg:   40,217,042 msg/s - min:   40,673,158 msg/s - max:   39,753,251 msg/s

#### relaxed/non seq_cst memory ordering
-> warmup:  145,534,005 msg/s
->    avg:  146,977,520 msg/s - min:  150,110,842 msg/s - max:  142,412,691 msg/s

```

### Cached head and cached tail indexes
This next optimisation aims to reduce the number of times each thread needs to acquire the atomic
value that the other thread releases by caching the last head/tail value that it most recently
acquired. This should reduce the contention to the atomic `head_` and `tail_` indexes as you only
`store()`/`load()` the atomic values if absolutely necessary.

```cpp
class spsc {
    //...

    bool try_push(const T& item) {
        auto tail = tail_.load(std::memory_order::relaxed);
        if (full(tail, cached_head_)) {
            cached_head_ = head_.load(std::memory_order::acquire);
            if (full(tail, cached_head_)) return false;
        }

        new (&data_[tail % SIZE]) T(item);
        tail_.store(tail + 1, std::memory_order::release);
        return true;
    }

    bool try_pop(T& item) {
        auto head = head_.load(std::memory_order::relaxed);
        if (empty(cached_tail_, head)) {
            cached_tail_ = tail_.load(std::memory_order::acquire);
            if (empty(cached_tail_, head)) return false;
        }

        item = data_[head % SIZE];
        head_.store(head + 1, std::memory_order::release);
        return true;
    }

    //...

    T data_[SIZE];
    std::atomic<size_type> head_{0};
    size_type cached_head_ = 0;
    std::atomic<size_type> tail_{0};
    size_type cached_tail_ = 0;
};

```

Unfortunately, we actually get a performance regression.

```sh

----------- SPSC Benchmarks -----------
#### MutexDequeQueue
-> warmup:   10,005,002 msg/s
->    avg:   10,027,332 msg/s - min:   10,074,475 msg/s - max:   10,003,196 msg/s

#### BoostLockFreeSPSCQueue
-> warmup:  154,255,067 msg/s
->    avg:  153,211,467 msg/s - min:  154,675,916 msg/s - max:  152,461,008 msg/s

#### RigtorpSPSCQueue
-> warmup:  271,997,921 msg/s
->    avg:  269,441,840 msg/s - min:  270,609,279 msg/s - max:  268,840,676 msg/s

#### Naive implementation
-> warmup:   41,278,800 msg/s
->    avg:   40,548,363 msg/s - min:   40,765,011 msg/s - max:   40,188,632 msg/s

#### relaxed/non seq_cst memory ordering
-> warmup:  146,407,352 msg/s
->    avg:  147,869,678 msg/s - min:  150,095,813 msg/s - max:  143,806,931 msg/s

#### Cached head and tail
-> warmup:  125,145,015 msg/s
->    avg:  131,339,664 msg/s - min:  132,660,935 msg/s - max:  129,997,173 msg/s

```

But why? Adding the cached head and tail has resulted in slightly less throughput. The answer is
false sharing. Although the algorithm has improved and logically _should_ give us more performance
we are encountering more false sharing which loses us any of the benefits.

### Prevent false sharing of indexes
The way to unlock the performance our caching should give us is to prevent the false sharing of
our indexes. We can achieve this pretty easily with the `alignas` keyword. This will force each
index to be in a separate cacheline to the other indexes so that they don't interfere with each
other.

The only changes are to add the `alignas` to the index data members.

```cpp

template <typename T, unsigned SIZE, T NIL>
class spsc {
    // ...
    T data_[SIZE];
    alignas(CACHE_LINE_SIZE) std::atomic<size_type> head_{0};
    alignas(CACHE_LINE_SIZE) size_type cached_head_ = 0;
    alignas(CACHE_LINE_SIZE) std::atomic<size_type> tail_{0};
    alignas(CACHE_LINE_SIZE) size_type cached_tail_ = 0;
};

```

This allows us to unlock the performance boost we expected. It results in a 6x boost over the
version without cached head and tail indexes.

```sh

----------- SPSC Benchmarks -----------
#### MutexDequeQueue
-> warmup:   10,033,431 msg/s
->    avg:   10,176,279 msg/s - min:   10,131,706 msg/s - max:   10,212,576 msg/s

#### BoostLockFreeSPSCQueue
-> warmup:  146,656,987 msg/s
->    avg:  151,560,982 msg/s - min:  150,426,258 msg/s - max:  153,368,289 msg/s

#### RigtorpSPSCQueue
-> warmup:  252,729,545 msg/s
->    avg:  272,166,661 msg/s - min:  265,879,466 msg/s - max:  277,707,559 msg/s

#### Naive implementation
-> warmup:   41,253,801 msg/s
->    avg:   41,562,526 msg/s - min:   41,438,022 msg/s - max:   41,657,855 msg/s

#### relaxed/non seq_cst memory ordering
-> warmup:  149,987,604 msg/s
->    avg:  151,495,432 msg/s - min:  149,356,810 msg/s - max:  154,251,798 msg/s

#### Cached head and tail
-> warmup:  139,535,907 msg/s
->    avg:  137,623,486 msg/s - min:  135,831,335 msg/s - max:  139,673,501 msg/s

#### Cached indexes no false sharing
-> warmup:  925,264,608 msg/s
->    avg:  913,674,449 msg/s - min:  895,144,557 msg/s - max:  931,635,381 msg/s

```

## Exploring other factors that affect performance

### Dynamic ring buffer allocation instead of static

### Ring buffer size not power of 2

### Producer and consumer thread on different cores(No shared L1 and L2 cache)

