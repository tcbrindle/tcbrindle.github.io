---
layout: post
title: A more useful compile-time quicksort in C++17
author: Tristan Brindle
permalink: /posts/a-more-useful-compile-time-quicksort
comments: true
tags: C++
---

A few days ago a post from Björn Fahller did the rounds about a [compile-time quicksort in C++](https://playfulprogramming.blogspot.co.uk/2017/06/constexpr-quicksort-in-c17.html), using template metaprogramming. It is, as the author concludes, "of limited usefulness, but kind of cool". Today, I happened to see a [follow-up in D](https://dlang.org/blog/2017/06/05/compile-time-sort-in-d/), effectively pointing out that D's standard library `sort()` function can be used at compile-time.

This got me thinking: can we implement a normal `sort()` function in C++, usable at compile-time as in D, but avoiding the need for any unpleasant metaprogramming?

It turns out the answer is yes. Here it is, in all its glory:

```cpp
namespace cstd {

template <typename RAIt>
constexpr RAIt next(RAIt it,
                    typename std::iterator_traits<RAIt>::difference_type n = 1)
{
    return it + n;
}

template <typename RAIt>
constexpr auto distance(RAIt first, RAIt last)
{
    return last - first;
}

template<class ForwardIt1, class ForwardIt2>
constexpr void iter_swap(ForwardIt1 a, ForwardIt2 b)
{
    auto temp = std::move(*a);
    *a = std::move(*b);
    *b = std::move(temp);
}

template<class InputIt, class UnaryPredicate>
constexpr InputIt find_if_not(InputIt first, InputIt last, UnaryPredicate q)
{
    for (; first != last; ++first) {
        if (!q(*first)) {
            return first;
        }
    }
    return last;
}

template<class ForwardIt, class UnaryPredicate>
constexpr ForwardIt partition(ForwardIt first, ForwardIt last, UnaryPredicate p)
{
    first = cstd::find_if_not(first, last, p);
    if (first == last) return first;

    for(ForwardIt i = cstd::next(first); i != last; ++i){
        if(p(*i)){
            cstd::iter_swap(i, first);
            ++first;
        }
    }
    return first;
}

}

template<class RAIt, class Compare = std::less<>>
constexpr void quick_sort(RAIt first, RAIt last, Compare cmp = Compare{})
{
    auto const N = cstd::distance(first, last);
    if (N <= 1) return;
    auto const pivot = *cstd::next(first, N / 2);
    auto const middle1 = cstd::partition(first, last, [=](auto const& elem){
        return cmp(elem, pivot);
    });
    auto const middle2 = cstd::partition(middle1, last, [=](auto const& elem){
        return !cmp(pivot, elem);
    });
    quick_sort(first, middle1, cmp); // assert(std::is_sorted(first, middle1, cmp));
    quick_sort(middle2, last, cmp);  // assert(std::is_sorted(middle2, last, cmp));
}

template <typename Range>
constexpr auto sort(Range&& range)
{
    quick_sort(std::begin(range), std::end(range));
    return range;
}
```

There's nothing particularly magical about this code. I put it together in about 10 minutes, and I can't take any credit for it. The quicksort implementation is taken from [this excellent post](https://stackoverflow.com/questions/24650626/how-to-implement-classic-sorting-algorithms-in-modern-c) by user *TemplateRex* on StackOverflow, and the rest is just reimplementations of a few standard library algorithms which are (sadly) not marked as `constexpr` (even in C++17). I copied `partition()` and `find_if_not()` from the sample implementations on [cppreference](https://cppreference.com), and the others are trivial (for random access iterators, anyway).

(**EDIT:** TemplateRex -- again! -- has [pointed out on Reddit](https://www.reddit.com/r/cpp/comments/6fnbfr/a_more_useful_compiletime_quicksort_in_c17/dijml7i/) that `std::advance()`, `next()`, `prev()` and `distance()` *are* in fact `constexpr` in C++17. Unfortunately these changes didn't make it into GCC 7.1 which I used for testing, but at least the code above will get a little bit smaller once compilers/libraries fully implement the new standard.)

The cool thing is that this version is usable both at compile-time:

```cpp
int main()
{
    constexpr auto arr = sort(std::array{4, 2, 1, 3});

    std::copy(std::cbegin(arr), std::cend(arr),
              std::ostream_iterator<int>{std::cout, " "});
}
```

[Wandbox](https://wandbox.org/permlink/Cpt6zxp5A71pqrt7)

...and at run-time.

```cpp
int main()
{
    const auto vec = sort(std::vector{4, 2, 1, 3});

    std::copy(std::cbegin(vec), std::cend(vec),
              std::ostream_iterator<int>{std::cout, " "});
}
```

[Wandbox](https://wandbox.org/permlink/aWqKqFbdlgtpD8Bn)


And that's really all there is to it. While `std::sort()` is more complicated than a simple quicksort (from what I understand, `std::sort()` uses a variety of different algorithms in different situations), on the face of it there doesn't appear to be any fundamental reason why it (and the other STL algoithms) couldn't be made available at compile-time, as in D. Perhaps we'll see it in C++20?
