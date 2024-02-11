---
title: "Branchless Programming in C++"
tags:
- source
- infomedia
- cppcon
enableToc: true
---
Link:  [Branchless programming in C++ CppCon video](https://www.youtube.com/watch?v=g-WPhYREFjk)  
Author:  [Fedor Pikus](Authors/Fedor%20Pikus.md)  
Topics: [Software Engineering](Topics/Software%20Engineering.md)  

---

## Potential benefits

**Common code example 1:**
```cpp
f(bool b, unsigned long x, unsigned long& s)
{
    if (b) s+= x;
}
```
130M calls/sec -> 400M calls/sec optimised  
Optimised version:  
```cpp
f(bool b, unsigned long x, unsigned long& s)
{
    s+= b * x; // use boolean as int to multiply 'x'
}
```

**Common code example 2:**
```cpp
if (x[i] || y[i])
{
    // Do something
}
```
150M evaluations/sec -> 570M evaluations/sec
(Optimisation show in <a href="#2b-bitwise-optimisation">2.b. bitwise optimisation</a>)  

## Philosophy for performance

In order of priority:
- Get desired result with ==least work==
    - Use an _optimal_ algorithm
- Do not do any ==unnecessary work==
    - Efficiently use language
- Use ==all available resources==, ==at the same time==, ==all the time==.
    - Efficient hardware use.

### CPU compute resources
**Inefficient use of CPU:**
```cpp
unsigned long v1[N], v2[N];
unsigned long a = 0;
for (size_t i = 0; i < N; ++i)
{
    a += v1[i] * v2[i];
}
```

Why is it inefficient use of CPU?  
Because it does not actually load the CPU very much at all as we are only doing one multiplication per iteration. In fact, more calculations can be thrown in for free.  

The following code will run at the same speed on modern CPUs.  
```cpp
unsigned long v1[N], v2[N];
unsigned long a1 = 0, a2 = 0;
for (size_t i = 0; i < N; ++i)
{
    a1 += v1[i] * v2[i];
    a2 += v1[i] + v2[i];
    ...
    // in fact you can insert even more operations for 'free'
}
```

Sounds great! Not quite. You can almost never do that due to ==data dependencies==. In order to do the next computation, you need the result from the first. Things get even worse if there are ==code dependencies.== There may be branches and conditions which means that the CPU now must wait until it knows which ==instructions== to execute as well.  

Having so much compute power would be useless however if we didn't have some workarounds!

## Pipelining
```cpp
a += (v1[i] + v2[i]) * (v1[i] - v2[i]);
```

In this example you can do the addition and subtraction in the first cycle, then do the multiplication in the second cycle, whilst also doing the addition and subtraction for the next iteration. This creates two _streams_ of instructions that are interleaved that have no data dependency between them at a given cpu cycle.  

This results in multiple instruction streams that have
- Dependencies within each stream
- No data dependencies between streams

This is great and increases cpu utilisation. However, the next barrier is... **conditional code.**

## Branches
Pipelining requires a continuous stream of instructions. Conditions/Branches mean that the CPU is unsure of the next instruction to place into the pipeline. So what can we do?

**Branch prediction**. The CPU guesses which branch to take and continues pipelining. The ==performance is now based on accuracy of predictions==. However, the branch can be mispredicted and recovering from a ==branch misprediction== is very costly.  

But how costly is this branch misprediction?

## Experiments

### 1.a. always true -> same branch taken

```cpp

#include "benchmark/benchmark.h"

#include <stdlib.h>
#include <string.h>

static void always_true(benchmark::State& state) {
    srand(1);
    const unsigned int N = state.range(0);
    std::vector<unsigned long> v1(N), v2(N);
    std::vector<int> c1(N);
    for (size_t i = 0; i < N; ++i) {
       v1[i] = rand();
       v2[i] = rand();
       c1[i] = rand() >= 0; // always true
    }
    unsigned long* p1 = v1.data();
    unsigned long* p2 = v2.data();
    int* b1 = c1.data();
    for (auto _ : state) {
        unsigned long a1 = 0, a2 = 0;
        for (size_t i = 0; i < N; ++i) {
            if (b1[i]) {
                a1 += p1[i];
            } else {
                a2 *= p2[i];
            }
        }
        benchmark::DoNotOptimize(a1);
        benchmark::DoNotOptimize(a2);
        benchmark::ClobberMemory();
    }
    state.SetItemsProcessed(N*state.iterations());
}

BENCHMARK(always_true)->Arg(1<<22);

BENCHMARK_MAIN();

```

**Benchmark result:**
```shell
Run on (12 X 4467.28 MHz CPU s)
CPU Caches:
  L1 Data 32 KiB (x6)
  L1 Instruction 32 KiB (x6)
  L2 Unified 512 KiB (x6)
  L3 Unified 32768 KiB (x1)
Load Average: 0.26, 0.35, 0.45

------------------------------------------------------------------------------
Benchmark                    Time             CPU   Iterations UserCounters...
------------------------------------------------------------------------------
always_true/4194304    1741613 ns      1734898 ns          430 items_per_second=2.41761G/s

```

### 1.b. random -> 50/50 if branch vs else branch

```cpp

#include "benchmark/benchmark.h"

#include <stdlib.h>
#include <string.h>

static void random(benchmark::State& state) {
    srand(1);
    const unsigned int N = state.range(0);
    std::vector<unsigned long> v1(N), v2(N);
    std::vector<int> c1(N);
    for (size_t i = 0; i < N; ++i) {
       v1[i] = rand();
       v2[i] = rand();
       c1[i] = rand() & 0x1; // randomly true/false
    }
    unsigned long* p1 = v1.data();
    unsigned long* p2 = v2.data();
    int* b1 = c1.data();
    for (auto _ : state) {
        unsigned long a1 = 0, a2 = 0;
        for (size_t i = 0; i < N; ++i) {
            if (b1[i]) {
                a1 += p1[i];
            } else {
                a2 *= p2[i];
            }
        }
        benchmark::DoNotOptimize(a1);
        benchmark::DoNotOptimize(a2);
        benchmark::ClobberMemory();
    }
    state.SetItemsProcessed(N*state.iterations());
}

BENCHMARK(random)->Arg(1<<22);

BENCHMARK_MAIN();

```

**Benchmark result:**
```shell
Run on (12 X 4467.28 MHz CPU s)
CPU Caches:
  L1 Data 32 KiB (x6)
  L1 Instruction 32 KiB (x6)
  L2 Unified 512 KiB (x6)
  L3 Unified 32768 KiB (x1)
Load Average: 0.26, 0.36, 0.44

-------------------------------------------------------------------------
Benchmark               Time             CPU   Iterations UserCounters...
-------------------------------------------------------------------------
random/4194304   12131503 ns     12091002 ns           58 items_per_second=346.895M/s
```

As you can see, the mispredicted benchmark is significantly slower. By about 7.7 times. Branch mispredictions are extremely costly.

But how do we detect when branch misprediction is a problem? By using a cpu profiling tool such as **==perf==**.

### Using perf to detect branch mispredictions.

The command to run is `perf stat <program>`  
The result for the 1a the 'always true' program is:  
```shell
task-clock:u                     #    1.000 CPUs utilized
context-switches:u               #    0.000 /sec
cpu-migrations:u                 #    0.000 /sec
page-faults:u                    #   58.862 K/sec
cycles:u                         #    4.217 GHz                      (83.30%)
stalled-cycles-frontend:u        #    1.63% frontend cycles idle     (83.35%)
stalled-cycles-backend:u         #    0.17% backend cycles idle      (83.35%)
instructions:u                   #    3.68  insn per cycle
                          #    0.00  stalled cycles per insn  (83.34%)
branches:u                       #    4.399 G/sec                    (83.35%)
branch-misses:u                  #    0.00% of all branches          (83.31%)
```

0% (or usually close to that) branch-misses. Compare that to 1b.

```shell
task-clock:u                     #    1.000 CPUs utilized
context-switches:u               #    0.000 /sec
cpu-migrations:u                 #    0.000 /sec
page-faults:u                    #   52.165 K/sec
cycles:u                         #    4.258 GHz                      (83.30%)
stalled-cycles-frontend:u        #    0.87% frontend cycles idle     (83.31%)
stalled-cycles-backend:u         #    0.00% backend cycles idle      (83.31%)
instructions:u                   #    0.88  insn per cycle
                          #    0.01  stalled cycles per insn  (83.33%)
branches:u                       #    1.078 G/sec                    (83.40%)
branch-misses:u                  #   12.16% of all branches          (83.35%)
```

12% branch-misprediction. With the context of the second perf result, you can see that ==instructions per cycle== is also massively down from 3.68 -> 0.88.  

### 1.c. alternating if branch and else branch -> predictable branching

```cpp

#include "benchmark/benchmark.h"

#include <stdlib.h>
#include <string.h>

static void alternating(benchmark::State& state) {
    srand(1);
    const unsigned int N = state.range(0);
    std::vector<unsigned long> v1(N), v2(N);
    std::vector<int> c1(N);
    for (size_t i = 0; i < N; ++i) {
        v1[i] = rand();
        v2[i] = rand();
        if (i == 0) c1[i] = rand() >=0;
        else c1[i] = !c1[i-1]; // alternate true and false
    }
    unsigned long* p1 = v1.data();
    unsigned long* p2 = v2.data();
    int* b1 = c1.data();
    for (auto _ : state) {
        unsigned long a1 = 0, a2 = 0;
        for (size_t i = 0; i < N; ++i) {
            if (b1[i]) {
                a1 += p1[i];
            } else {
                a2 *= p2[i];
            }
        }
        benchmark::DoNotOptimize(a1);
        benchmark::DoNotOptimize(a2);
        benchmark::ClobberMemory();
    }
    state.SetItemsProcessed(N*state.iterations());
}

BENCHMARK(alternating)->Arg(1<<22);

BENCHMARK_MAIN();

```
**Benchmark result:**
```shell
Run on (12 X 4467.28 MHz CPU s)
CPU Caches:
  L1 Data 32 KiB (x6)
  L1 Instruction 32 KiB (x6)
  L2 Unified 512 KiB (x6)
  L3 Unified 32768 KiB (x1)
Load Average: 0.89, 0.46, 0.42
------------------------------------------------------------------------------
Benchmark                    Time             CPU   Iterations UserCounters...
------------------------------------------------------------------------------
alternating/4194304    2529638 ns      2523583 ns          280 items_per_second=1.66204G/s
```

**Perf result:**
```shell
task-clock:u                     #    0.999 CPUs utilized
context-switches:u               #    0.000 /sec
cpu-migrations:u                 #    0.000 /sec
page-faults:u                    #   57.934 K/sec
cycles:u                         #    4.229 GHz                      (83.30%)
stalled-cycles-frontend:u        #    0.74% frontend cycles idle     (83.32%)
stalled-cycles-backend:u         #    0.22% backend cycles idle      (83.31%)
instructions:u                   #    2.39  insn per cycle
                          #    0.00  stalled cycles per insn  (83.34%)
branches:u                       #    3.029 G/sec                    (83.39%)
branch-misses:u                  #    0.00% of all branches          (83.34%)
```

With an alternating pattern, you can see that it is slower than the 'alway true' program, but faster than the horribly mispredicted one. 440 vs 58 vs 280.  

The branch-misses are also at 0%. The CPU figured out the pattern!  

### 2.a. Predictable result, unpredictable branch

This is a case where you have an || condition.
```cpp

#include "benchmark/benchmark.h"

#include <stdlib.h>
#include <string.h>

static void random_predictable(benchmark::State& state) {
    srand(1);
    const unsigned int N = state.range(0);
    std::vector<unsigned long> v1(N), v2(N);
    std::vector<int> c1(N);
    std::vector<int> c2(N);
    for (size_t i = 0; i < N; ++i) {
        v1[i] = rand();
        v2[i] = rand();
        c1[i] = rand() & 0x1;
        c2[i] = !c1[i];
    }
    unsigned long* p1 = v1.data();
    unsigned long* p2 = v2.data();
    int* b1 = c1.data();
    int* b2 = c2.data();
    for (auto _ : state) {
        unsigned long a1 = 0, a2 = 0;
        for (size_t i = 0; i < N; ++i) {
            if (b1[i] || b2[i]) {
                a1 += p1[i];
            } else {
                a2 *= p2[i];
            }
        }
        benchmark::DoNotOptimize(a1);
        benchmark::DoNotOptimize(a2);
        benchmark::ClobberMemory();
    }
    state.SetItemsProcessed(N*state.iterations());
}

BENCHMARK(random_predictable)->Arg(1<<22);

BENCHMARK_MAIN();

```

the 'if' condtion is always true as b1 is true when b2 isn't and viceversa. Although the result is predictable, the branch taken is not. This is because the branch is either through b1 or b2. Here are the benchmark results  

```shell
Running ./02-random_predictable
Run on (12 X 4467.28 MHz CPU s)
CPU Caches:
  L1 Data 32 KiB (x6)
  L1 Instruction 32 KiB (x6)
  L2 Unified 512 KiB (x6)
  L3 Unified 32768 KiB (x1)
Load Average: 0.42, 0.54, 0.54
-------------------------------------------------------------------------------------
Benchmark                           Time             CPU   Iterations UserCounters...
-------------------------------------------------------------------------------------
random_predictable/4194304   12137243 ns     12055659 ns           58 items_per_second=347.912M/s

task-clock:u                     #    1.000 CPUs utilized
context-switches:u               #    0.000 /sec
cpu-migrations:u                 #    0.000 /sec
page-faults:u                    #   65.941 K/sec
cycles:u                         #    4.206 GHz                      (83.30%)
stalled-cycles-frontend:u        #    0.85% frontend cycles idle     (83.30%)
stalled-cycles-backend:u         #    0.45% backend cycles idle      (83.30%)
instructions:u                   #    1.01  insn per cycle
                          #    0.01  stalled cycles per insn  (83.36%)
branches:u                       #    1.195 G/sec                    (83.39%)
branch-misses:u                  #   10.82% of all branches          (83.35%)
```

As you can see the iterations value is down to 58 and branch-misses is up to 10.82%. Even though the result is the same every time.  

How can we optimise this?

### 2.b. bitwise optimisation
Here I will use addition. You can also use logical 'or'.
```cpp

#include "benchmark/benchmark.h"

#include <stdlib.h>
#include <string.h>

static void bitwise(benchmark::State& state) {
    srand(1);
    const unsigned int N = state.range(0);
    std::vector<unsigned long> v1(N), v2(N);
    std::vector<int> c1(N);
    std::vector<int> c2(N);
    for (size_t i = 0; i < N; ++i) {
        v1[i] = rand();
        v2[i] = rand();
        c1[i] = rand() & 0x1;
        c2[i] = !c1[i];
    }
    unsigned long* p1 = v1.data();
    unsigned long* p2 = v2.data();
    int* b1 = c1.data();
    int* b2 = c2.data();
    for (auto _ : state) {
        unsigned long a1 = 0, a2 = 0;
        for (size_t i = 0; i < N; ++i) {
            if (bool(b1[i]) + bool(b2[i])) {
                a1 += p1[i];
            } else {
                a2 *= p2[i];
            }
        }
        benchmark::DoNotOptimize(a1);
        benchmark::DoNotOptimize(a2);
        benchmark::ClobberMemory();
    }
    state.SetItemsProcessed(N*state.iterations());
}

BENCHMARK(bitwise)->Arg(1<<22);

BENCHMARK_MAIN();
```

The results:  
```shell
Running ./02-bitwise
Run on (12 X 4467.28 MHz CPU s)
CPU Caches:
  L1 Data 32 KiB (x6)
  L1 Instruction 32 KiB (x6)
  L2 Unified 512 KiB (x6)
  L3 Unified 32768 KiB (x1)
Load Average: 0.69, 0.50, 0.51

--------------------------------------------------------------------------
Benchmark                Time             CPU   Iterations UserCounters...
--------------------------------------------------------------------------
bitwise/4194304    2113774 ns      2097247 ns          337 items_per_second=1.99991G/s


task-clock:u                     #    1.000 CPUs utilized
context-switches:u               #    0.000 /sec
cpu-migrations:u                 #    0.000 /sec
page-faults:u                    #   73.555 K/sec
cycles:u                         #    4.164 GHz                      (83.33%)
stalled-cycles-frontend:u        #    1.04% frontend cycles idle     (83.33%)
stalled-cycles-backend:u         #    0.36% backend cycles idle      (83.33%)
instructions:u                   #    4.60  insn per cycle
                          #    0.00  stalled cycles per insn  (83.33%)
branches:u                       #    3.419 G/sec                    (83.33%)
branch-misses:u                  #    0.00% of all branches          (83.35%)
```

Here you can see branch-misses are down to 0% again and iterations are up from 58 to 337.

**Bear in mind,** the optimisation is that instead of evaluating both conditions separately, you calculate both and then check both at the same time. This means you are doing more instructions. ==That is the tradeoff==. This can be important if you, for example, battery life is important to you.  

Additionally, if the result was not predictable, this would also instead be a performance hit.

### 3.a. Branched unpredictable
Experiment 3 will consist of taking a loop with 1 branch and showing how we can eliminate the branch completely. The first case is then the branched example.

```cpp

#include "benchmark/benchmark.h"

#include <stdlib.h>
#include <string.h>

static void branched_unpredictable(benchmark::State& state) {
    srand(1);
    const unsigned int N = state.range(0);
    std::vector<unsigned long> v1(N), v2(N);
    std::vector<int> c1(N);
    for (size_t i = 0; i < N; ++i) {
       v1[i] = rand();
       v2[i] = rand();
       c1[i] = rand() & 0x1;
    }
    unsigned long* p1 = v1.data();
    unsigned long* p2 = v2.data();
    int* b1 = c1.data();
    for (auto _ : state) {
        unsigned long a1 = 0, a2 = 0;
        for (size_t i = 0; i < N; ++i) {
            if (b1[i]) {
                a1 += p1[i] - p2[i];
            } else {
                a2 *= p2[i] * p2[i];
            }
        }
        benchmark::DoNotOptimize(a1);
        benchmark::DoNotOptimize(a2);
        benchmark::ClobberMemory();
    }
    state.SetItemsProcessed(N*state.iterations());
}

BENCHMARK(branched_unpredictable)->Arg(1<<22);

BENCHMARK_MAIN();

```

**Result:**
```shell
Running ./03-branched-unpredictable
Run on (12 X 4467.28 MHz CPU s)
CPU Caches:
  L1 Data 32 KiB (x6)
  L1 Instruction 32 KiB (x6)
  L2 Unified 512 KiB (x6)
  L3 Unified 32768 KiB (x1)
Load Average: 0.76, 0.75, 0.63

-----------------------------------------------------------------------------------------
Benchmark                               Time             CPU   Iterations UserCounters...
-----------------------------------------------------------------------------------------
branched_unpredictable/4194304   13162172 ns     13070350 ns           54 items_per_second=320.902M/s

task-clock:u                     #    0.999 CPUs utilized
context-switches:u               #    0.000 /sec
cpu-migrations:u                 #    0.000 /sec
page-faults:u                    #   51.062 K/sec
cycles:u                         #    4.230 GHz                      (83.31%)
stalled-cycles-frontend:u        #    0.87% frontend cycles idle     (83.32%)
stalled-cycles-backend:u         #    0.20% backend cycles idle      (83.20%)
instructions:u                   #    0.88  insn per cycle
                          #    0.01  stalled cycles per insn  (83.41%)
branches:u                       #    1.029 G/sec                    (83.40%)
branch-misses:u                  #   11.79% of all branches          (83.36%)
```

54 iterations, 11.79% branch-misses.

### 3.b. branchless unpredictable
This example now shows how we can eliminate the branch completely.

```cpp

#include "benchmark/benchmark.h"

#include <stdlib.h>
#include <string.h>

static void branchless_unpredictable(benchmark::State& state) {
    srand(1);
    const unsigned int N = state.range(0);
    std::vector<unsigned long> v1(N), v2(N);
    std::vector<int> c1(N);
    for (size_t i = 0; i < N; ++i) {
       v1[i] = rand();
       v2[i] = rand();
       c1[i] = rand() & 0x1;
    }
    unsigned long* p1 = v1.data();
    unsigned long* p2 = v2.data();
    int* b1 = c1.data();
    for (auto _ : state) {
        unsigned long a1 = 0, a2 = 0;
        for (size_t i = 0; i < N; ++i) {
            unsigned long s1[2] = {0, p1[i] - p2[i]};
            unsigned long s2[2] = {p1[i] * p2[i], 0};
            a1 += s1[bool(b1[i])];
            a2 += s2[bool(b1[i])];
        }
        benchmark::DoNotOptimize(a1);
        benchmark::DoNotOptimize(a2);
        benchmark::ClobberMemory();
    }
    state.SetItemsProcessed(N*state.iterations());
}

BENCHMARK(branchless_unpredictable)->Arg(1<<22);

BENCHMARK_MAIN();

```

What we are doing is essentially calculating both branches and storing them in an array. Then using the bool 'b1' as an integer to add both branch results to 'a1' and 'a2'. The key being that s1 and s2 store a '0' value in position 0 and 1 respectively. So how effective is this?

```shell
Running ./03-branchless-unpredictable.cpp
Run on (12 X 4467.28 MHz CPU s)
CPU Caches:
  L1 Data 32 KiB (x6)
  L1 Instruction 32 KiB (x6)
  L2 Unified 512 KiB (x6)
  L3 Unified 32768 KiB (x1)
Load Average: 0.30, 0.40, 0.51

-------------------------------------------------------------------------------------------
Benchmark                                 Time             CPU   Iterations UserCounters...
-------------------------------------------------------------------------------------------
branchless_unpredictable/4194304    4246080 ns      4207040 ns          166 items_per_second=996.973M/s

task-clock:u                     #    0.998 CPUs utilized
context-switches:u               #    0.000 /sec
cpu-migrations:u                 #    0.000 /sec
page-faults:u                    #   48.458 K/sec
cycles:u                         #    4.253 GHz                      (83.31%)
stalled-cycles-frontend:u        #    1.16% frontend cycles idle     (83.34%)
stalled-cycles-backend:u         #    0.17% backend cycles idle      (83.37%)
instructions:u                   #    3.69  insn per cycle
                          #    0.00  stalled cycles per insn  (83.36%)
branches:u                       #    1.301 G/sec                    (83.29%)
branch-misses:u                  #    0.00% of all branches          (83.32%)
```

The iterations have increased from 54 to 166 and branch-misses are now 0% again.  

Note that the iterations has not increased as drastically as we are actually doing a lot more work, but there is still a significant performance boost.

This optimisation is effective under two circumstances:
1. Extra computations are small
2. Branch is poorly predicted

## Closing thoughts

- Predicted branches are cheap
- Mispredictions are **very** expensive
- **ALWAYS** use a profiler to detect optimisation locations.
- Don't fight the compiler as it can often do the optimisations for you.