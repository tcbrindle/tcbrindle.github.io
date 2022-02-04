---
layout: post
title: Ranges and Forwarding References
author: Tristan Brindle
permalink: /posts/ranges-and-forwarding-references
comments: false
tags: C++
---

Arthur O'Dwyer recently blogged about ["universal references" vs "forwarding references"](https://quuxplusone.github.io/blog/2022/02/02/look-what-they-need/), where he puts the case that `T&&` (where `T` is a deduced template parameter) should be properly called a "forwarding reference" rather than the original term (coined by Scott Meyers), "universal reference".

I happen to agree that "forwarding reference" is a better name. But Arthur goes on to say

> If you see code using `std::forward<T>` without an originating `T&&`, it’s almost certainly buggy. If you see code using (deduced) `T&&` without `std::forward<T>`, it’s either buggy or it’s C++20 Ranges. (Ranges ill-advisedly uses value category to denote lifetime rather than pilferability, so Ranges code tends to forward rvalueness much more conservatively than ordinary C++ code does.)

It's true that ranges uses value category as a proxy for lifetime in the shape of `borrowed_range`, which I've [written about before](https://tristanbrindle.com/posts/rvalue-ranges-and-views). But that's not actually why ranges uses `T&&` without `std::forward` -- it would do so even if it didn't care about trying to save you from dangling iterators. It's not buggy, it's the right thing to do, and it's something to consider for your own code as well where appropriate.

## What does ranges do, anyway?

Every ranges algorithm starts with a function<sup id="a1">[1](#f1)</sup> taking an iterator and a sentinel, much like the classic STL algorithms. We can write one ourselves (constraints, projections etc omitted for brevity):

```cpp
// Overload (0)
template <typename I, typename S, typename T>
size_t count(I iter, S last, const T& value)
{
    size_t n = 0;
    while (iter != last) {
        if (*iter == value) {
            ++n;
        }
        ++iter;
    }
    return n;
}
```

Now, like the `std::ranges` algorithms, we want to add a second overload taking a range as a whole, which in turn dispatches to the iter/sentinel version. We *could* take the range argument by value:

```cpp
// Don't do this
template <typename R, typename T>
size_t count(R rng, const T& value)
{
    return count(ranges::begin(rng), ranges::end(rng), value);
}
```

but of course this would be a silly thing to do, at least for non-`view` ranges. Instead, we should take the range by reference. In this particular case, we're not modifying the range elements, just examining them, so we could take the range parameter by const ref:

```cpp
// Overload (1)
template <typename R, typename T>
size_t count(const R& rng, const T& value)
{
    return count(ranges::begin(rng), ranges::end(rng), value);
}
```

This works perfectly well in many cases. Unfortunately not all cases though --  not every range is *const-iterable*, and in C++20 it's [not particularly difficult](https://godbolt.org/z/dncTPv4PY) to come up with examples which are not. So we're going to need a third overload, taking a range by non-const reference:

```cpp
// Overload (2)
template <typename R, typename T>
size_t count(R& rng, const T& value)
{
    return count(ranges::begin(rng), ranges::end(rng), value);
}
```

With these in place, we'll now do the right thing for both const- and non-const ranges:

```cpp
const std::vector vec1{1, 2, 3, 4, 5};
count(vec1, 3); // calls overload (1)

std::vector vec2{6, 7, 8, 9, 10};
count(vec2, 8); // calls overload (2)
```

So, are we done now? Alas not. Consider what happens if we call `count()` with an rvalue range:

```cpp
std::vector<int> get_vector();

count(get_vector(), 3);
```

Here, overload (2) is not viable, because we cannot bind an rvalue argument to a non-const lvalue reference. Overload (0) is of course not viable either as we have a mismatch in the number of arguments, so we'll end up calling overload (1) taking `const R&`. But as we've already mentioned, not all ranges are *const-iterable* -- and [sure enough](https://godbolt.org/z/s9s78n9xo) it's easy to come up with examples where this would be a problem.

So what's the solution? You guessed it: yet another `count()` overload, this time taking an rvalue reference as an argument:

```cpp
// Overload (3)
template <typename R, typename T>
    requires (not std::is_lvalue_reference_v<R>)
size_t count(R&& rng, const T& value)
{
    return count(ranges::begin(rng), ranges::end(rng), value);
}
```

Here, `R&&` is in fact a ~~*universal*~~ *forwarding reference*, so we've added a constraint that it's only viable when `R` is deduced to a non-reference type -- that is, when the argument is an rvalue. Now, things work as we would like:

```cpp
const std::vector vec1{1, 2, 3, 4, 5};
count(vec1, 3); // const lvalue, calls (1)

std::vector vec2{6, 7, 8, 9, 10};
count(vec2, 8); // non-const lvalue, calls (2)

std::vector<int> get_vector();
count(get_vector(), 99); // rvalue, calls (3)
```

Happy days! All we need to do is write four separate overloads for every ranges algorithm.

## How many overloads?!

But let's back up a moment. We added a constraint to overload (3) because we didn't want it to be called with lvalues. But let's just suppose we... didn't. What's the worst that could happen if we called it with lvalue arguments?

```cpp
template <typename R, typename T>
size_t forwarding_count(R&& rng, const T& value) {
    return count(ranges::begin(rng), ranges::end(rng), value);
}

const std::vector vec1{1, 2, 3, 4, 5};
forwarding_count(vec1, 3);
```

In this case, `R` would be deduced to be `const std::vector<int>&`, and after type substitution we'd end up with a specialisation like this:

```cpp
size_t forwarding_count(const std::vector<int>& rng, const int& value)
{
    return count(ranges::begin(rng), ranges::end(rng), value);
}
```

This is *exactly the same* function as we'd get from calling overload (1) of our original `count()`!

The same thing happens if we call `forwarding_count()` with a non-const lvalue argument:

```cpp
std::vector vec2{6, 7, 8, 9, 10};
forwarding_count(vec2, 8);
```

This time `R` would be deduced as (non-const) `std::vector<int>&` and we'd end up with the exact same function as we would from overload (2) of `count()`.

In other words, if we remove the constraint from overload (3) it will correctly handle const lvalues, non-const lvalues and rvalues -- meaning don't need overloads (1) or (2) at all!

Our complete and correct `count()` implementation thus becomes

```cpp
template <typename I, typename S, typename T>
size_t count(I iter, S last, const T& value)
{
    size_t n = 0;
    while (iter != last) {
        if (*iter == value) {
            ++n;
        }
        ++iter;
    }
    return n;
}

template <typename R, typename T>
size_t count(R&& rng, const T& value)
{
    return count(ranges::begin(rng), ranges::end(rng), value);
}
```

## ...and that's it

For examples like this, **using a forwarding reference does the right thing in all cases**, even though we aren't `std::forward`-ing anything. It's correct, it's not buggy, and it saves us from doing extra redundant work.

And *that's* why ranges does it.

---

<b id="f1">1</b> Ranges algorithms are in fact currently implemented using function objects, but that's not important for the present discussion [↩](#a1)