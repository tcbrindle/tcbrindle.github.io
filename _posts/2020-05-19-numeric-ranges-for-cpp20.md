---
layout: post
title: Numeric Range Algorithms for C++20
author: Tristan Brindle
permalink: /posts/numeric-ranges-for-cpp20
comments: false
tags: C++
---

*TL;DR: I've written C++20 range-based versions of the algorithms in the `<numeric>` header, and you can get them [here](https://github.com/tcbrindle/numeric_ranges)*

---


C++20 brings with it updated versions of the many, many algorithms in the `<algorithm>` header. Sadly, as [Peter Dimov recently noted](https://pdimov.github.io/blog/2020/05/16/c20-ranges-to-the-rescue/), it does not do so for the "other" algorithms in the `<numeric>` header.

The reason for this is simply one of time. It turns out that correctly defining the concepts for the numeric algorithms is very tricky to do, and so to avoid ranges "missing the bus" with C++20 these needed to go on the back-burner for a little while. But Christopher Di Bella is [currently working on it](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2019/p1813r0.pdf), so I'm very hopeful we'll see conceptified, range-based versions of `accumulate()` and friends in C++23.

As a stop-gap solution in the mean time, I've written [`numeric_ranges.hpp`](https://github.com/tcbrindle/numeric_ranges) -- modern implementations of the most common `<numeric>` algorithms which you can use with C++20 ranges. You can get it [here](https://github.com/tcbrindle/numeric_ranges).

These implementations simply ignore the hardest part of the problem -- actually defining the numeric concepts -- and so, unlike the other algorithms in the `std::ranges` namespace, they are *unconstrained* <sup>[1](#1)</sup> and don't offer any protection from ugly, C++98-style template error messages when you get things wrong.

Nonetheless, they do still offer the other benefits of the `std::ranges` algorithms, for example:

 * They have range-based overloads (of course)
 * They accept optional projections
 * They're `constexpr`-callable
 * They're [CPOs](https://stackoverflow.com/questions/53495848/what-are-customization-point-objects-and-how-to-use-them), so you can (for example) pass them to other algorithms without needing to wrap them in lambdas.

```cpp
constexpr std::array arr{1, 2, 3, 4};
std::vector<int> out;

tcb::partial_sum(arr, std::back_inserter(out));
// out contains [1, 3, 6, 10]

const int prod = tcb::inner_product(arr, out, 0);
// prod = (1 * 1) + (2 * 3) + (3 * 6) + (4 * 10) = 65
assert(prod == 65);

constexpr auto sq = [](int i) { return i * i; };
constexpr int sum = tcb::accumulate(arr, 0, {}, sq);
// sum = 1 + 4 + 9 + 16 = 30
static_assert(sum == 30);
```
[Try it out!](https://gcc.godbolt.org/z/efj74i)

## Not using C++20 yet? ##

At the time of writing the only standard library implementation of ranges comes with the brand-new GCC 10.1. Other implementations are sure to follow in due time, and when they do, `numeric_ranges.hpp` will work with them.

But if you're not using C++20 or a brand new GCC yet, don't despair! You can still use all the new ranges goodness in C++17 with [NanoRange](http://github.com/tcbrindle/nanorange) -- and, of course, `numeric_ranges.hpp` works with NanoRange too<sup>[2](#2)</sup>.

Enjoy!

<a name="1">1</a>: Or rather, constrained just enough to avoid ambiguous calls, and no more.

<a name="2">2</a>: Before anyone asks, Range-V3 comes with [its own implementations](https://github.com/ericniebler/range-v3/tree/master/include/range/v3/numeric) of these algorithms, so `numeric_ranges.hpp` isn't needed there.