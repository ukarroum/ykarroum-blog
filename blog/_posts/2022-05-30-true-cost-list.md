---
layout: post
title: 'The true cost of linked lists'
---

Linked lists are often overused in introductory algorithmics courses, due to a heavy focus on theorical complexities. Unfortunatly, in practice, computers are complex beasts. They don't execute instructions sequentially [^1] with the same cost [^2]. This means that a data structure with faster theorical complexities does not necessarily translate to a more efficient data structure in practice.

In this article I'll illutrate this by comparing the real performances of linked lists versus a contiguous vector, and I'll show that even in some use cases where the list seems to be favored the vector is still a better choice. Of course, this doesn't mean you should never use a linked list, but that you should be aware of its practical limitations and more informed when making the decision to use it or not.

Although linked lists are not specific to a particular language, the following article we'll focus on C++ STL's `list` vs. C++ STL's `vector`.

All the examples and measurements were ran on the following platform:

* Ubuntu 20.04 (Linux 5.13.0-44-generic).
* Intel(R) Core(TM) i5-9300H CPU @ 2.40GHz
* C++ 14
* Clang 12

We'll also use _google's benchmark_ [^5] to make the measurements.

Finding (the last) element
--------------------------

Let's start with a simple scenario: Finding an element in our data structure. We'll focus on the worst case, when the element is at the end of the container.

The theorical complexities are:

* For list: O(n)
* For vector: O(n)

You may expect the measurement to give us a similar value.

We'll measure using the following code:

{% highlight cpp %}

#include <benchmark/benchmark.h>
#include <list>

using namespace std;

static void BM_ListFind(benchmark::State &state) {


    for (auto _: state) {
        state.PauseTiming();

        list<int> l, tmp_list;

        for(int i = 0; i < state.range(0); i++)
        {
            l.push_back(0);
            // This is used to have a more realistic benchmark, where you don't insert all your elements at once
            for(int j = 0; j < 10; j++)
                tmp_list.push_back(j);
        }

        l.push_back(1);

        state.ResumeTiming();

        find(l.begin(), l.end(), 1);
    }
}

static void BM_VectorFind(benchmark::State &state) {
    for (auto _: state) {
        state.PauseTiming();

        vector<int> v(state.range(0), 0);

        v.push_back(1);

        state.ResumeTiming();

        find(v.begin(), v.end(), 1);
    }
}

BENCHMARK(BM_ListFind)->Range(8, 8 << 10);
BENCHMARK(BM_VectorFind)->Range(8, 8 << 10);

BENCHMARK_MAIN();

{% endhighlight %}

This gives us the following results:

{% highlight bash %}

2022-05-31T19:16:37+02:00
Running ./a.out
Run on (8 X 4100 MHz CPU s)
CPU Caches:
  L1 Data 32 KiB (x4)
  L1 Instruction 32 KiB (x4)
  L2 Unified 256 KiB (x4)
  L3 Unified 8192 KiB (x1)
Load Average: 2.40, 1.62, 1.40
***WARNING*** CPU scaling is enabled, the benchmark real time measurements may be noisy and will incur extra overhead.
-------------------------------------------------------------
Benchmark                   Time             CPU   Iterations
-------------------------------------------------------------
BM_ListFind/8            2824 ns         2825 ns       247103
BM_ListFind/64          18212 ns        18206 ns        38773
BM_ListFind/512        153438 ns       153420 ns         4528
BM_ListFind/4096      1618680 ns      1618639 ns          446
BM_ListFind/8192      3758778 ns      3758624 ns          204
BM_VectorFind/8           573 ns          575 ns      1210734
BM_VectorFind/64          876 ns          877 ns       803397
BM_VectorFind/512        3224 ns         3227 ns       214354
BM_VectorFind/4096      21782 ns        21772 ns        32395
BM_VectorFind/8192      43581 ns        43578 ns        16272

{% endhighlight %}

Surprisingly enough, the vector version is almost 90 times faster than the list version for 8K.

How can we explain this big difference?

Let me introduce _spatial locality_ and _cache_.

Caches and locality
-------------------

When you run a program (like our benchmarking example), you usually communicate (a lot) with the main memory. Although computers CPUs speed has improved a lot in the recent years, memory latencies did not. A typical memory access has a 100 ns latency [^3]. To improve overall performances, all modern CPUs now have multiple levels of caches. The cache is a blazingly fast, but very small, memory. On intel's cpus you usually have 3 levels of cache (L1 -> L3), L1 being the fastest and smallest, and L3 (also commonly called LLC) being the slowest and biggest.

According to [^3], the latencies are:

* L1: 0.5 ns.
* L2: 7 ns.
* Main Memorry (DRAM): 100 ns.

To show the cache's size for your computer, you can use:

{% highlight bash %}

$ lscpu  | grep "cache:"
L1d cache:                       128 KiB
L1i cache:                       128 KiB
L2 cache:                        1 MiB
L3 cache:                        8 MiB

{% endhighlight %}

You can see that I have two types of L1 cache: _L1d_ for _data_ and _L1i_ for _instructions_.

The CPU will keep data based on _locality_. We have two types of locality: _spatial_ and _temporal_.

_Spatial locality_: When you access an address in memory, you'll probably need the content of the addresses closest to it.

_Temporal locality_: When you access an address in memory, you'll probably need the same value in the near future.

But what does any of this have with our initial problem ?

Well the memory layout of the vector heavily benefit from the spatial locality, this is what happens in our benchmarking example:

* We Retrieve the first element value from memory (cost ~100 ns).
* The CPU expects us to use the near addresses' contents, he'll then retrieve a whole cache line from memory (64 bytes, or 16 ints on my computer).
* The next iteration, we try to access the content of the second element, since the values are contiguous in memory the value is in cache (cost ~7ns).

For the linked list, things are a little different:

* We Retrieve the first element content from memory (cost ~100 ns).
* The CPU expects us to use the near addresses' contents, he'll then retrieve a whole cache line from memory (again 64 bytes).
* Then we fetch the next pointer, the next address is not necessarily the one next to the last one (cost 7ns to 100ns, in practice probably 100ns).

This is why I added the `tmp_list`, inserting all the elements at once will just put all the elements next to each other, which is not a realistic scenario.

When the CPU has to fetch from main memory instead of the cache, we say that the CPU did a _cache miss_.

Counting cache misses
---------------------

We can count the number of cache misses using `jevents` from `pmu_tools` [^4].

I'll explain in a future article how the PMU/PMC works. For now you just need to understand that the CPU has a bunch of "counters" that we can use to count specific events, in our case the cache misses event.

We can use the following code:

{% highlight cpp %}
#include <rdpmc.h>
#include <linux/perf_event.h>

static void BM_ListFind(benchmark::State &state) {
    struct rdpmc_ctx ctx;
    rdpmc_open(PERF_COUNT_HW_CACHE_MISSES, &ctx);
    unsigned long long start, end;
    unsigned long long total = 0;

    for (auto _: state) {
        // We initialise our list here with a the 0 elements and one last 1.

        start = rdpmc_read(&ctx);
        find(l.begin(), l.end(), 1);
        end = rdpmc_read(&ctx);

        total += end - start;
    }

    // We can use google benchmark counters to have a nice reporting at the end 
    state.counters["# cache misses"] = total / (double) state.iterations();

    rdpmc_close(&ctx);
}
{% endhighlight %}

{% highlight bash %}
-------------------------------------------------------------
Benchmark                   Time    Iterations # cache misses
-------------------------------------------------------------
BM_ListFind/8            3029 ns        242975      0.0562321
BM_ListFind/64          21968 ns         34644        2.45855
BM_ListFind/512        172679 ns          4051        14.7554
BM_ListFind/4096      1751986 ns           356        796.604
BM_ListFind/8192      3430648 ns           202       1.69684k
BM_VectorFind/8           624 ns       1103030       6.54833m
BM_VectorFind/64          925 ns        754477       9.99765m
BM_VectorFind/512        3385 ns        210652      0.0321431
BM_VectorFind/4096      22554 ns         30999       0.662796
BM_VectorFind/8192      45021 ns         15576        1.29173

{% endhighlight %}

You may find it odd that we have less than one cache miss for some rows, I mean we should at least need to access DRAM once right ?
This is due google benchmarks running multiple iterations, first iteration will put the data in cache (precisly L2 and L3 that are big enough to keep it) and latter iterations will just use those low level caches. We can remove this effect by increasing the number of elements inserted in `tmp_list`.

As you can see for 8K we make, in average 941 cache misses.

A linked list on contiguous memorry
---------------------------

We can now remove the `tmp_list` to have a linked list on contiguous memory, although in practice you'll rarely use a list for a similar use case:

{% highlight cpp %}
#include <benchmark/benchmark.h>
#include <list>

using namespace std;

static void BM_ListFind(benchmark::State &state) {


    for (auto _: state) {
        state.PauseTiming();

        list<int> l(state.range(0), 0);

        l.push_back(1);

        state.ResumeTiming();

        find(l.begin(), l.end(), 1);
    }
}

static void BM_VectorFind(benchmark::State &state) {
    for (auto _: state) {
        state.PauseTiming();

        vector<int> v(state.range(0), 0);

        v.push_back(1);

        state.ResumeTiming();

        find(v.begin(), v.end(), 1);
    }
}

BENCHMARK(BM_ListFind)->Range(8, 8 << 10);
BENCHMARK(BM_VectorFind)->Range(8, 8 << 10);

BENCHMARK_MAIN();


{% endhighlight %}

{% highlight bash %}

2022-05-31T19:17:04+02:00
Running ./a.out
Run on (8 X 4100 MHz CPU s)
CPU Caches:
  L1 Data 32 KiB (x4)
  L1 Instruction 32 KiB (x4)
  L2 Unified 256 KiB (x4)
  L3 Unified 8192 KiB (x1)
Load Average: 2.83, 1.79, 1.46
***WARNING*** CPU scaling is enabled, the benchmark real time measurements may be noisy and will incur extra overhead.
-------------------------------------------------------------
Benchmark                   Time             CPU   Iterations
-------------------------------------------------------------
BM_ListFind/8             827 ns          828 ns       839339
BM_ListFind/64           2784 ns         2787 ns       252559
BM_ListFind/512         18073 ns        18073 ns        38276
BM_ListFind/4096       144158 ns       143987 ns         4808
BM_ListFind/8192       292014 ns       291886 ns         2421
BM_VectorFind/8           614 ns          615 ns      1119671
BM_VectorFind/64          918 ns          920 ns       735972
BM_VectorFind/512        3253 ns         3255 ns       214164
BM_VectorFind/4096      23021 ns        23015 ns        32041
BM_VectorFind/8192      44067 ns        44072 ns        16161

{% endhighlight %}

The _vector_ version is _only_ 7 times faster.

The remaining difference can be explained by the size of the list node (32 bytes), a cache line can only hold 2 nodes while it can hold 16 integers.

We can count the number of cache misses:

{% highlight bash %}

--------------------------------------------------------------
Benchmark                   Time     Iterations # cache misses
--------------------------------------------------------------
BM_ListFind/8             869 ns         813883      0.0198431
BM_ListFind/64           2934 ns         236903       0.146575
BM_ListFind/512         19266 ns          37231        2.06008
BM_ListFind/4096       153910 ns           4617         70.506
BM_ListFind/8192       310404 ns           2273        202.148
BM_VectorFind/8           619 ns        1054643      0.0176287
BM_VectorFind/64          936 ns         732034      0.0277105
BM_VectorFind/512        3401 ns         207771      0.0983631
BM_VectorFind/4096      23231 ns          30661        1.17905
BM_VectorFind/8192      45480 ns          15304        3.28679

{% endhighlight %}

We can see that the number of cache misses has greatly decreased.

There is also the factor that by constructing the list once, we only do one _malloc_, instead of _n_ mallocs which is also a major drawback of using a _list_ instead of _vector_.

What about inserting in the middle of the list ?
------------------------------------------------

We measured the performance differences in the use case where both data structures have the same theoretical complexity, but what about when the list is supposed to clearly win ? Let see what happens if we try to insert in the middle of the data structure.

We can measure using the following code:

{% highlight cpp %}

#include <benchmark/benchmark.h>
#include <list>

using namespace std;

static void BM_ListInsert(benchmark::State &state) {


    for (auto _: state) {
        state.PauseTiming();

        list<int> l(state.range(0), 0);

        state.ResumeTiming();

        for(int i = 0; i < state.range(1); i++){
            auto sec_it = l.begin();
            sec_it++;
            l.insert(sec_it, 1);
        }
    }
}

static void BM_VectorInsert(benchmark::State &state) {
    for (auto _: state) {
        state.PauseTiming();

        vector<int> v(state.range(0), 0);

        state.ResumeTiming();

        for(int i = 0; i < state.range(1); i++)
            v.insert(v.begin() + 1, 1);

    }
}

BENCHMARK(BM_ListInsert)->Ranges({ {8, 8 << 10}, {8, 8 << 10} });
BENCHMARK(BM_VectorInsert)->Ranges({ {8, 8 << 10}, {8, 8 << 10} });

BENCHMARK_MAIN();

{% endhighlight %}

The results (at this point I'm starting to think that plots would have been better):

{% highlight bash %}

--------------------------------------------------------------------
Benchmark                          Time             CPU   Iterations
--------------------------------------------------------------------
BM_ListInsert/8/8               1446 ns         1448 ns       484951
BM_ListInsert/64/8              2816 ns         2819 ns       248839
BM_ListInsert/512/8            13441 ns        13449 ns        51717
BM_ListInsert/4096/8           98892 ns        98894 ns         6984
BM_ListInsert/8192/8          197436 ns       197415 ns         3550
BM_ListInsert/8/64              6549 ns         6550 ns       106478
BM_ListInsert/64/64             7876 ns         7878 ns        88595
BM_ListInsert/512/64           18704 ns        18721 ns        37360
BM_ListInsert/4096/64         104789 ns       104791 ns         6711
BM_ListInsert/8192/64         209456 ns       209441 ns         3430
BM_ListInsert/8/512            46935 ns        46946 ns        14982
BM_ListInsert/64/512           48067 ns        48089 ns        14506
BM_ListInsert/512/512          61742 ns        61763 ns        11842
BM_ListInsert/4096/512        149032 ns       149036 ns         4802
BM_ListInsert/8192/512        253049 ns       253037 ns         2881
BM_ListInsert/8/4096          400709 ns       400703 ns         1874
BM_ListInsert/64/4096         375582 ns       375586 ns         1783
BM_ListInsert/512/4096        415123 ns       415109 ns         1704
BM_ListInsert/4096/4096       505394 ns       505322 ns         1477
BM_ListInsert/8192/4096       575418 ns       575379 ns         1134
BM_ListInsert/8/8192          816366 ns       816346 ns          856
BM_ListInsert/64/8192         758503 ns       758501 ns          842
BM_ListInsert/512/8192        771846 ns       771823 ns          891
BM_ListInsert/4096/8192       866835 ns       866779 ns          793
BM_ListInsert/8192/8192       973025 ns       972948 ns          718
BM_VectorInsert/8/8             1201 ns         1202 ns       577995
BM_VectorInsert/64/8            1247 ns         1249 ns       553982
BM_VectorInsert/512/8           1463 ns         1464 ns       476086
BM_VectorInsert/4096/8          2882 ns         2888 ns       242229
BM_VectorInsert/8192/8          4770 ns         4772 ns       147160
BM_VectorInsert/8/64            6142 ns         6144 ns       113078
BM_VectorInsert/64/64           5923 ns         5923 ns       117244
BM_VectorInsert/512/64          7290 ns         7290 ns        94952
BM_VectorInsert/4096/64        18229 ns        18238 ns        38540
BM_VectorInsert/8192/64        30272 ns        30273 ns        22990
BM_VectorInsert/8/512          48338 ns        48340 ns        14481
BM_VectorInsert/64/512         49249 ns        49250 ns        14209
BM_VectorInsert/512/512        59721 ns        59720 ns        11596
BM_VectorInsert/4096/512      149113 ns       149106 ns         4685
BM_VectorInsert/8192/512      253002 ns       252983 ns         2775
BM_VectorInsert/8/4096        735142 ns       735092 ns          950
BM_VectorInsert/64/4096       745214 ns       745173 ns          937
BM_VectorInsert/512/4096      835744 ns       835693 ns          835
BM_VectorInsert/4096/4096    1554628 ns      1554518 ns          451
BM_VectorInsert/8192/4096    2494333 ns      2494082 ns          281
BM_VectorInsert/8/8192       2290270 ns      2290135 ns          305
BM_VectorInsert/64/8192      2311752 ns      2311568 ns          302
BM_VectorInsert/512/8192     2496926 ns      2496692 ns          281
BM_VectorInsert/4096/8192    4074648 ns      4074257 ns          172
BM_VectorInsert/8192/8192    5907307 ns      5906365 ns          113

{% endhighlight %}

As you can see the _vector_ is still faster on a lot of sizes, the _list_ data structures only become faster when we start inserting 4k elements!
Why? Well each time we insert we pay a malloc, and _malloc_ can be really costly.

People tend to focus on the cost of moving the whole vector just to insert one element, but in practice (unless you're doing it too often), the _mallocs_ cost will still be greater.

Conclution
----------

Should we then stop using linked lists? No, there are still use cases where _list_ will outperform a vector (as we saw earlier if you have a lot of insertions), but the point of this article is to show that we need to be more careful and informed before committing to using it, and one single insert in the middle of the list doesn't justify the cost. It's important to measure and to understand that theoretical model may not take into consideration things like cache costs and mallocs costs.

The same reasoning can be used on other data structures as well, like trees (std::map and std::set are usually implemented using a red black tree).

Footnotes
---------

[^1]: See Instruction Level Paralelism or Out of Order Execution.
[^2]: A typical load from memory is around 100 slower than a simple `ADD`.
[^3]: [http://www.cs.cornell.edu/projects/ladis2009/talks/dean-keynote-ladis2009.pdf](http://www.cs.cornell.edu/projects/ladis2009/talks/dean-keynote-ladis2009.pdf) (2009)
[^4]: [https://github.com/andikleen/pmu-tools/tree/master/jevents](https://github.com/andikleen/pmu-tools/tree/master/jevents) 
[^5]: [https://github.com/google/benchmark](https://github.com/google/benchmark)
