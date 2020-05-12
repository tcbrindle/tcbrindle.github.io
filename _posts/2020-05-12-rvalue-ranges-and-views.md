---
layout: post
title: Rvalue Ranges and Views in C++20
author: Tristan Brindle
permalink: /posts/rvalue-ranges-and-views
comments: false
tags: C++
---

Back in the mists of time, when the world was a more normal place (well, September last year) I gave a talk at CppCon entitled ["An Overview of Standard Ranges"](https://youtu.be/SYLgG7Q5Zws). Unfortunately I ran a little long and didn't have time to take questions on video at the end, but since my talk was right before lunch I ended up having a informal Q&A session in the room afterwards for about 45 minutes. One of the questions that came up a few times was about the relationship between rvalue ranges and views, and what you can do with them. At the time it occurred to me that it might be useful to write a blog post clarifying things, but since the nomenclature and mechanisms were still somewhat in flux at the time I decided I'd wait until C++20 was finalised.

Well, that a couple of months ago and now I've run out of excuses, so here we go.

## Algorithms, rvalues and dangling iterators ##

Arguably the best thing about the new algorithms in the `std::ranges` namespace is that we can, at long last, pass range objects to them directly -- so for example we can now say

```cpp
std::vector<int> get_vector();

auto vec = get_vector();
// returns an iterator to the first element with value 99
auto it = std::ranges::find(vec, 99);
```

rather than having to type

```cpp
auto it = std::find(vec.begin(), vec.end(), 99);
```

as we have done for the past 25 years or so.

This is great, but there is a potential problem here. `find()` returns an iterator into the passed-in range -- so what happens if we call it with an rvalue `vector`, for example?

```cpp
auto it = std::ranges::find(get_vector(), 99);
```

The temporary returned by `get_vector()` will be destroyed at the end of the statement, meaning that `it` will be unsafe to use -- it is a *dangling iterator*. Dangling pointers and iterators are already a major problem in C++, and we don't want to make the situation worse by allowing this sort of dangerous behaviour when using ranges. So, what can we do about it?

One solution would to simply forbid passing rvalue containers to algorithms. In this case the third call to `find()` above would generate a compile error, which is certainly safer!

This is a tempting solution, and in fact it was briefly used at one point during the ranges standardisation process. However, there are times when passing an rvalue container into an algorithm is perfectly safe, and indeed leads to cleaner code. Consider:

```cpp
// Print each element separated by spaces
std::ranges::copy(get_vector(),
                  std::ostream_iterator<int>(std::cout, " "));
```

Forbidding all uses of rvalue containers would make the above a compile error, forcing you to create a variable to hold the result of `get_vector()` before passing that to `copy()`. This seems overly heavy-handed -- after all, the problem is not with the use of rvalue ranges per se, but with **returning iterators** into such ranges that people might later try to use.

The solution in C++20 is to have a special type, [`std::ranges::dangling`](http://eel.is/c++draft/range.dangling), which algorithms use in their return types instead of potentially-dangling iterators when passed an rvalue range. So for example

```cpp
auto vec = get_vector();
auto it = std::ranges::find(vec, 99);
```

is fine -- `it` will be a `std::vector::iterator`. But if you were to write

```cpp
auto it = std::ranges::find(get_vector(), 99);
```

then the code will still compile, but `it` is now of type `std::ranges::dangling`. This type does not provide any operations, so if you try to use it like you would an iterator you'll get a handy error message involving the `dangling` type, [like this](https://godbolt.org/z/SRYkb-), alerting you to what's gone wrong.

(Note that this does prevent things like

```cpp
auto min_val = *std::ranges::min_element(get_vector());
// Error, no operator* in std::ranges::dangling
```

which is still safe provided the result of `get_vector()` is non-empty, as the iterator does not out-live the full-expression. In this case however, C++20 provides a new overload of `min()`, so you can say

```cpp
auto min_val = std::ranges::min(get_vector()); // Fine
```

instead.)

## Can I borrow your range? ##

Dangling iterators are a potential problem for most kinds of ranges -- we need to be careful to make sure that we do not use a `vector` iterator or an `unordered_map` iterator after the parent object has been destroyed, for example.

This is not the case for all ranges though. A `std::string_view`'s iterators for example actually point into some other character array -- so we can safely use a `string_view::iterator` after the `string_view` itself has been destroyed (provided the underlying array is still around, of course):

```cpp
std::string str = "Hello world";
// Weird, but okay:
auto iter = std::string_view{str}.begin();
*iter = 'h';
```

Types like `std::string_view` are called [*borrowed ranges*](http://eel.is/c++draft/range.range#itemdecl:2) in the new C++20 terminology. A borrowed range is one whose iterators can safely outlive their parent object. So an rvalue `std::vector` is *not* a borrowed range, but an rvalue `std::span` is.

Since we don't need to worry about dangling iterators when we use borrowed ranges, we don't need to apply the "dangling protection" described above when we pass them as rvalues to algorithms; we can just return iterators as normal:

```cpp
auto iter = std::ranges::min_element(get_span());
// iter is an iterator, *not* ranges::dangling
```

By default, all rvalue ranges are assumed to be non-borrowing. If we have a custom type whose iterators *can* safely "dangle" (say, a custom `StringRef` type) then we can opt in to allowing the ranges machinery to consider it *borrowed* by specialising the `enable_borrowed_range` trait:

```cpp
// Outside of any namespace
template <>
inline constexpr bool std::ranges::enable_borrowed_range<my::StringRef> = true;
```



## Can I borrow your view? ##

So now we know what a borrowed range is, and we know that if we pass an non-borrowed rvalue range into an algorithm, it will give us back a `std::ranges::dangling` object rather than a potentially dangerous dangling iterator.

There's one other piece of the puzzle left to discuss, namely **views**.

Informally, views are sometimes described as "ranges which do not own their elements". This makes them sound quite a lot like borrowed ranges -- but in actual fact they are distinct concepts.

A [`view`](http://eel.is/c++draft/range.view) is a range which is default constructible, has constant-time move and destruction operations and, if it is copyable, then constant-time copy operations (where "constant-time" here means "independent of the number of elements in the range"). So a `std::string_view` is a view, but a `std::vector` is not (because its copy and destruction operations are *O(N)*). These properties mean that we can pass views around by value without worrying too much about the cost of those operations.

Because the compiler can't readily check the algorithmic complexity of operations, we need to opt in to the `view` concept, just as with `borrowed_range`. There are two ways of doing this. Firstly, we can use another trait:

```cpp
template <>
inline constexpr bool std::ranges::enable_view<my::StringRef> = true
```

The second way is by making the view class inherit from the empty `std::ranges::view_base` base class -- but it's probably more useful to inherit from the [`std::ranges::view_interface`](http://eel.is/c++draft/view.interface) CRTP mixin instead (which in turn inherits from `view_base`) which also provides default implementations of some useful member functions for free.

## Putting it all together ###

So, views and borrowed ranges are independent concepts:

* **Borrowed**: a range whose iterators will not dangle -- either an lvalue range, or an rvalue with `enable_borrowed_range`
* **View**: a range with constant-time copy/move/destroy -- inherits from `view_base` or uses `enable_view`

These are tied together by the [`std::views::all()`](http://eel.is/c++draft/range.all) function, which takes a borrowed range and turns it into a view (by means of [`ref_view`](http://eel.is/c++draft/range.ref.view) or, in rare cases, [`subrange`](http://eel.is/c++draft/range.subrange)). If it is passed something that is already a view, then `views::all()` is a no-op.

Together, borrowed ranges and views are called **viewable ranges**, because they can be turned into views with `views::all()`. Why is this important? Because viewable ranges are the things that are allowed on the left-hand side of the ranges "pipe" operator:

```cpp
auto vec = get_vector();

auto v1 = vec | views::transform(func);
// Okay: vec is an lvalue
auto v2 = get_span() | views::transform(func);
// Okay: span is borrowed
auto v3 = subrange(vec.begin(), vec.end()) | views::transform(func)
// Okay: subrange is borrowed (and a view)
auto v4 = get_vector() | views::transform(func);
// ERROR: get_vector() returns an rvalue vector, which is neither
// a view nor a borrowed range
```

Another way of putting it is that `view`s expect to operate on other `view`s, and the ranges machinery helps you out by first using `views::all()` to turn a borrowed range into a `view` when necessary.

Conversely, not all views are borrowed ranges -- in fact, most of them are not. This means that if we pass an rvalue view into an algorithm, the "dangling protection" will kick in:

```cpp
auto vec = std::vector{-3, 2, -1, 0, 1, 2, 3};
auto sq = [](int i) { return i * i; };

auto m = std::ranges::min_element(std::views::transform(vec, sq));

std::cout << "Least square: " << *m << '\n';
// ERROR: no operator* in std::ranges::dangling
```
<small>[Compiler Explorer](https://godbolt.org/z/YwGd8K)</small>

The solution here is to save the view in a named variable and pass it to `min_element` as an lvalue instead. (In this specific case you could use `min()` instead as we did above, or be really fancy and [use a projection](https://godbolt.org/z/FVnunc) -- but that's a topic for another time...)

## TL;DR ##

In summary:

* A borrowed range is one whose iterator validity doesn't rely on the lifetime of the parent object
* Passing a non-borrowed range to an algorithm as an rvalue means that it will return a `ranges::dangling` rather than an iterator
* A borrowed range can be wrapped in a view object using `views::all()`
* Range adaptors only operate on lvalues, or borrowed range/view rvalues -- otherwise you get a compile error

## Can you put that in a table for me? ##

Sure!

| `T` |   `borrowed_range<T>`  | `view<T>` | `viewable_range<T>` |
| ---| ---| --- |---|
|`vector<int>` | ❌ | ❌|❌|
|`vector<int>&`|✅|❌|✅
|`string_view` | ✅| ✅ |✅|
|`span<int>`|✅|✅|✅|
|`span<int, 3>`|✅|❌|✅|
| `transform_view` |❌|✅|✅|
|`iota_view`|✅|✅|✅|

* In the `borrowed_range` column, a ✅ means that `std::ranges` algorithms will return an iterator rather than `ranges::dangling`
* In the `viewable_range` column, a ✅ means that the given type can be used on the left-hand side of the ranges "pipe" operator
* The `view` column is just to show that not all borrowed ranges are views :)


