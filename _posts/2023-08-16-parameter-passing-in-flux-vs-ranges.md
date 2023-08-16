---
layout: post
title: Parameter Passing in Flux versus Ranges
author: Tristan Brindle
permalink: /posts/parameter-passing-in-flux-vs-ranges
comments: false
tags: C++
---

Regular readers of this irregular blog will know that I'm a huge fan of C++20 Ranges and the 'collection-orientated' style of code that they allow. For the last few months, I've been working on a new C++20 library called [Flux](https://github.com/tcbrindle/flux), which aims to provide many of the same facilities as Ranges as well as offering improved safety, ease-of-use, and in some cases better runtime efficiency.

In the first of what might turn out to be a series comparing Flux and Ranges, I want to look at how the two libraries differ in their handling of arguments to sequence adaptors.

## The similarities ##

Let's start with the similarities. Both Flux and Ranges differentiate between eagerly evaluated *algorithms* and lazy *adaptors*, the latter of which are often called 'views' in the Ranges world.

The eager algorithms are functions that take one or more sequences and immediately iterate over them, either returning the result of a calculation or modifying a sequence in-place. Flux and Ranges use the same approach here: algorithm implementations take their arguments as forwarding references, meaning they can accept lvalues, rvalues, const and non-const arguments alike. Because algorithms immediately operate their arguments and return you the answer, there are no major lifetime issues to worry about: if you pass in an rvalue temporary and it disappears after the call is finished, everything is okay. The algorithm has done its job while the object was alive and we (usually) no longer need to worry about it.

Adaptors on the other hand are a little different. In both Flux and Ranges, adaptors don't do any actual work when you first construct them. Instead, they return new sequences which only perform their operations as you later iterate over them. We call this kind of 'just in time' work *lazy evaluation*.

Both libraries implement adaptors in more-or-less the same way, as classes templated on some underlying sequence type, with a data member representing the underlying sequence:

```cpp
template <typename Base>
class some_adaptor {
    Base base_;

    /* ...other stuff... */
};
```

If `Base` owns the elements it points to, then again, there are no lifetime issues here: `Base` owns the elements, and `some_adaptor` owns the `Base`, and so everything is fine. This works recursively as you build up an adaptor pipeline, with each adaptor templated on its 'parent' in the adaptor chain.

The problem comes in the case where `Base` is non-owning, and refers to elements with distinct lifetime -- think of something like a `span` or `string_view`. In this case, it's the programmer's responsibility to ensure that the original elements outlive the adaptor object that refers to them, otherwise you're going to have a bad time.

All of this is true for both Flux and Ranges -- in C++ we don't have a garbage collector to clean up after us, or a borrow checker to make sure our references' lifetimes are in order. The difference between the libraries comes in how they pass arguments into their adaptor classes, and in particular how Flux tries to *make potentially long-lived references explicit*.

## Reviewing Ranges ##

Before we get on to talking about Flux, let's review how things are done in the Ranges world. Let's say we have a line of code like this:

```cpp
auto filtered = std::views::filter(get_range(), pred);
```

and have a look at what happens depending on what `get_range()` returns.

Although certain `views::xxx()` functions [can do more sophisticated things](https://brevzin.github.io/c++/2023/03/14/prefer-views-meow/) depending on the types you provide, `views::filter()` is a relatively simple one: it always constructs a new `ranges::filter_view`.

But it's still not all *that* simple. There are actually five possible different behaviours here, depending on whether `get_range()` returns an lvalue or an rvalue, whether the returned range satisfies the [`std::ranges::view`](https://en.cppreference.com/w/cpp/ranges/view) concept or not, and in the latter case, whether the `view` is copyable.

We'll start off with the easiest case. If `get_range()` returns an rvalue `view`, it's simply moved into a newly constructed `filter_view` object as we'd probably expect. Nothing surprising about that.

If `get_range()` returns an *lvalue* `view` on the other hand, it will be *copied* into the new `filter_view`. Ranges requires that copying a view, if it works, is a constant-time operation, so this isn't going to be too expensive. If the view is non-copyable, we'll get a compile error.

What about if `get_range()` returns a non-`view`? Well, that's where things get interesting. For an rvalue non-`view`, the object returned from `get_range()` will be wrapped in a [`ranges::owning_view`](https://en.cppreference.com/w/cpp/ranges/owning_view) object, which is basically a move-only wrapper around the returned range. (We can't know whether some arbitrary range is `O(1)` copyable, so Ranges disallows copying in order to maintain the `view` semantic requirements.) As the name suggests, in this case the `filter_view` will end up owning the returned range.

> This behaviour is a relatively recent change to Ranges -- previously there was no `owning_view` and attempting to pass an rvalue non-`viewable_range` into an adaptor was a compile error. This was changed in [P2415](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2021/p2415r2.html) in 2021 and retroactively applied to C++20.

There is one last case to consider: when `get_range()` returns an lvalue non-`view`. This time, the returned reference is wrapped in a [`ranges::ref_view`](https://en.cppreference.com/w/cpp/ranges/ref_view), which stores a pointer to the returned range and passes that into the `filter_view`. Because the `filter_view` now holds a pointer to the underlying range, it means that we need to be careful to make sure that we don't allow the underlying range to be destroyed prematurely. In fact, we need to be pretty careful about *any* changes we make to the underlying range while we're using the `filter_view`, as we might [accidentally end up invalidating it](https://brevzin.github.io/c++/2023/04/25/mutating-filter/).

Let's sum that up again: what happens when we pass a range argument `r` into an adaptor?

 * rvalue + view: `r` is moved into the adaptor
 * lvalue + view: `r` is copied into the adaptor (in constant time), or a compile error if `r` is non-copyable.
 * rvalue + non-view: `r` is moved into the adaptor via an `owning_view`, meaning the adaptor will be non-copyable
 * lvalue + non-view: the address of `r` is stored in a `ref_view`, which is in turn stored in the adaptor

It's the *implicit referencing* in the last case that is potentially problematic, because it's not particularly obvious when forming a range adaptor whether there might be lifetime issues. If you use Ranges a lot then the rules become etched in your memory, but not every C++ programmer uses Ranges (or even C++) every day. It also means that if we're not careful, innocent-looking code can end up causing *dangling views*:

```cpp
auto filter_even(const std::vector<int>& vec)
{
    return std::views::filter(vec, [](int i) { return i % 2 == 0; });
}

auto evens = filter_even({1, 2, 3, 4, 5}); // Oh no!!
```

Here, `vec` is initialised from an rvalue temporary and its address is then stored in the returned object, which becomes dangling when the temporary is destroyed at the end of the statement. Any later use of `evens` is going to end up attempting to read from deleted memory -- but that's not necessarily obvious unless you're well-versed in how range adaptors work.

### An aside: causing const confusion ###

There is another potential surprise with the way Ranges does things, which is when it comes to *const iteration*. This doesn't affect `views::filter` because that's never const-iterable, but it would be noticeable with something like `views::take`:

```cpp
std::vector vec{1, 2, 3, 4, 5};

const auto taken = std::move(vec) | std::views::take(3);

taken.front() = 10; // Error, assigning to const reference
```

Here we're moving `vec` into the `take_view`, meaning that the view owns the elements -- and since we declared `taken` to be `const`, we can't change those elements later. Specifically, `ranges::range_reference_t<decltype(taken)>` is `const int&`, meaning that the last line is a compile error.

On the other hand, if we were to do something like this:

```cpp
std::vector vec{1, 2, 3, 4, 5};

const auto taken = vec | std::views::take(3);

taken.front() = 10; // Fine, vec is now [10, 2, 3, 4, 5]
```

Now `taken` holds a `ref_view`, which holds a `std::vector<int>*`, and the top-level const no longer has any effect on element access: `range_reference_t<decltype(taken)>` is non-const `int&`, meaning the assignment works just fine.

Again, if you're comfortable enough with C++ to know the difference between `T const*` and `T* const`, and if you've used ranges enough to know when adaptors take implicit references, then this behaviour is exactly what you'd expect. For everyone else, the apparent lack of const-correctness in this case might come as a bit of a surprise.

## Getting that sinking feeling ##

Although Flux treats arguments to sequence *algorithms* in much the same way as Ranges does, things are rather different when it comes to sequence *adaptors*.

In Flux, an adaptor is considered a *sink function*, meaning it **always takes ownership** of the object you pass to it. In other words, adaptors in Flux always behave as if they take their sequence arguments **by value**. Let's have a look at an example:

```cpp
auto is_even = [](int i) { return i % 2 == 0; };

auto evens = flux::filter(std::array{1, 2, 3, 4, 5}, is_even);
```

Here we're passing an rvalue `std::array` into the `filter` adaptor function. The adaptor will take ownership of the array, which is moved into the returned object -- similar to what would happen with the equivalent Ranges code since P2415, but without wrapping in an `owning_view`.

The same thing happens if we explicitly `move()` into the adaptor:

```cpp
std::array array{1, 2, 3, 4, 5};

auto evens = flux::filter(std::move(array), is_even);
```

Again, we're passing an rvalue argument that is moved into the resulting adaptor object.

Where things differ is when we pass an lvalue argument, like so:

```cpp
std::array array{1, 2, 3, 4, 5};

auto evens = flux::filter(array, is_even);
```

As with all Flux adaptors, `filter()` behaves as if it takes its arguments by value, so `evens` will be initialised with a *copy* of the array. Value semantics means there are no possible lifetime issues in this case -- the lifetime of `evens` is distinct from that of `array` -- and modifying `array` cannot put the filter adaptor into an unexpected state.

But wait, isn't there an obvious downside to this approach? Copying five `int`s is one thing, but I definitely don't want to accidentally pass a multi-megabyte `std::vector` into a function by value!

Fortunately, Flux has you covered. Flux adaptors behave *as if* they take their arguments by value, but the 'as if' part is important. What actually happens is that an implicit copy is performed *only for types for which `std::is_trivially_copyable_v` is true*. Passing a non-trivially-copyable lvalue is a compile error:

```cpp
std::vector<int> vec = get_huge_vector();

auto evens = flux::filter(vec, is_even); // ERROR
                                         // vec is not trivially copyable
```

Non-trivial types like `vector` must be passed as rvalues, meaning you need to create an explicit copy if copying is the behaviour you want. Flux provides a `copy()` function for this purpose, or you can use `auto()` in C++23:

```cpp
std::vector<int> vec = get_huge_vector();

auto evens = flux::filter(flux::copy(vec), is_even); // Explicit copy
// or
auto evens = flux::filter(auto(vec), is_even); // Explicit copy (C++23)
// or
auto evens = flux::filter(std::move(vec), is_even); // Explicit move
```

In other words, if copying is (likely to be) 'cheap' then Flux will do the copy for you, otherwise you need to explicitly request it. Not only does this prevent accidental expensive copies, but it also makes it obvious in your source code when a potentially expensive copy is happpening. As an added bonus, the 'as if' behaviour enables us to avoid an extra move that might sometimes be needed with 'true' pass-by-value.

> Note that `is_trivially_copyable` isn't a perfect proxy for 'is cheap to copy', because trivially copyable types can still be arbitrarily large -- something like `std::array<int, 1'000'000>` for example. I've toyed with the idea of adding a size cutoff for implicit copies, but it's hard to find a good upper limit and I'm reluctant to have some arbitrary `N` for which `array<T, N>` works but `array<T, N+1>` doesn't, as that would be very surprising and bad for generic code. In any case, huge stack-allocated trivially copyable classes are probably rare in practice, and where they do exist you're much more likely to want to pass them by reference as we'll see below.

## References really required? ##

Value semantics are great, but there are times when we actually *do* want to pass references around.

For such cases, Flux provides **explicit pass-by-reference** into adaptors using the `flux::ref()` function. For example:

```cpp
std::vector vec{1, 2, 3, 4, 5};

auto evens = flux::filter(flux::ref(vec), is_even);
```

The `ref()` function accepts an lvalue sequence and returns a specialisation of the internal `ref_adaptor` type, which works very much like Ranges' `ref_view` in that it stores a pointer to the passed-in sequence.

As such, we do now need to be careful with lifetimes to make sure that the referred-to object outlives the object holding the reference. The key difference is that with Flux, *taking a reference is clearly visible at the call-site*.

Just as with Ranges, since `vec` and `evens` are now referring to the same data we again need to be careful not to invalidate the filter state by modifying `vec` "behind its back". But since it's now more obvious that the same data is being referred to twice, this situation is easier to avoid.

Similarly, consider the Flux equivalent of the 'dangling view' Ranges code we saw earlier. In Flux, trying to call `filter(vec, pred)` with an lvalue vector is not going to work -- it will be a compile error, as we saw above. Instead,  we would need to *explicitly opt in* to doing the wrong thing, and if we do so then it becomes immediately apparent that something strange is going on:

```cpp
auto filter_even(const std::vector<int>& vec)
{
    // This now looks very fishy
    return flux::filter(flux::ref(vec), [](int) { return i % 2 == 0; });
}

auto evens = filter_even({1, 2, 3, 4, 5});
```

With the explicit use of `flux::ref()` it's now much more obvious that we're returning a reference to `vec` from our function, with the potential lifetime problems that that entails.

## Making mutability more marked ##

Mutable references are bad.

Memory-safe languages like Rust, Swift and ~~Val~~ Hylo provide their safety guarantees in part by strictly limiting how and when mutable references can be used. Specifically, in these languages we can have **either** a single mutable reference to an object, **or** any number of immutable references, but not both at the same time -- the so-called "law of exclusivity". (Swift and Hylo don't directly expose references to the user, but that's what's going on behind the scenes.)

If we could somehow follow this same rule in C++ then in theory we would be able to achieve a similar level of memory safety. While this is very difficult to do in general without compiler or language support, there are a couple of ways in which we can try to make it easier -- for example by making mutable references more obvious in our code.

To this end, `flux::ref(x)` actually takes a **const reference** `x` and internally stores a `T const*`. That is, it behaves pretty much the same as `std::cref()`, with the added constraint that its argument must be a const-iterable `flux::sequence`. The returned object is freely copyable (indeed, trivially copyable) so you can pass it multiple times to functions if you wish:

```cpp
std::vector<int> vec = get_vector();

auto ref = flux::ref(vec); // stores std::vector<int> const*

auto zipped = flux::zip(ref, flux::drop(ref, 1)); // Okay, copies ref twice

for (auto [a, b] : zipped) {
    // ...
}
```

If you actually want to pass a mutable reference into a Flux adaptor, you need to say `flux::mut_ref(x)`. This is slightly more to type than just `ref()`, helping to emphasise that you should reach for a const reference first; it also means that it's more obvious in our code where we are passing a mutable reference so we can try to avoid having more than one of them at any time. Unlike with `ref()`, the object returned from `mut_ref()` is move-only, to try to help prevent accidentally creating multiple mutable references.

This means that if we tried to write the same zip code again but with a mutable reference, we're going to get a compile error:

```cpp
std::vector<int> vec = get_vector();

auto mref = flux::mut_ref(vec); // stores std::vector<int>*

auto zipped = flux::zip(mref, flux::drop(mref, 1)); // ERROR
    // mref is not copyable
```

We could try to avoid this compile error in a couple of ways, either by moving `mref` twice:

```cpp
std::vector<int> vec = get_vector();

auto mref = flux::mut_ref(vec);

auto zipped = flux::zip(std::move(mref), flux::drop(std::move(mref), 1));
// Compiles, but looks very wrong
```

Or by calling `mut_ref()` on the same argument twice:

```cpp
std::vector<int> vec = get_vector();

auto zipped = flux::zip(flux::mut_ref(vec),
                        flux::drop(flux::mut_ref(vec), 1));
// Ditto
```

But in either case the result is something that *just looks wrong*.

The equivalent Ranges code for the last example is more consise:

```cpp
std::vector<int> vec = get_vector();

auto zipped = std::views::zip(vec, std::views::drop(vec, 1));
```

but in this case the fact that we're storing multiple mutable references to `vec` is not at all obvious.

> The Flux iteration model also helps to avoid multiple referencing, because Flux cursors do not in general constitute a "borrow" in Rust terminology, unlike STL iterators -- but that's a topic for another post.

### An aside: const confusion clarified? ###

We saw earlier that for Ranges, the const-accessibility of elements of an adapted range depends on whether the source range was an lvalue or an rvalue, and whether it models the `view` concept:

```cpp
std::vector vec{1, 2, 3, 4, 5};

auto view1 = std::views::take(vec, 3);
view1.front() = 10; // Okay

const auto view2 = std::views::take(vec, 3);
view2.front() = 10; // Still okay

auto view3 = std::views::take(auto(vec), 3); // copy vec
view3.front() = 10; // Okay

const auto view4 = std::views::take(auto(vec), 3);
view4.front() = 10; // Error, assigning to const reference
```

With Flux on the other hand, sequence adaptors always take their arguments as-if by value, and we need to explicitly opt-in to operating on references with `flux::ref()` or `mut_ref()`. This makes things somewhat less surprising: if we request const- or mutable reference access via `ref()` or `mut_ref()` respectively, then that's what we get, regardless of the const-ness of sequence object that holds the `ref_adaptor`. Otherwise, if we transfer ownership of the elements into the adaptor, then the const-ness of the elements follows the const-ness of the sequence object as we would expect.

The following example demonstrates the various cases:

```cpp
std::vector vec{1, 2, 3, 4, 5};

auto seq1 = flux::take(vec, 3); // Error, vec is not trivially_copyable

auto seq2 = flux::take(flux::ref(vec), 3); // Want read-only access to elements of vec
seq2.front().value() = 10; // Error, elements are immutable, as requested

auto const seq3 = flux::take(flux::mut_ref(vec), 3); // Want read-write access to elements of vec
seq3.front().value() = 10; // Okay, elements are mutable, as requested

auto seq4 = flux::take(auto(vec), 3); // No referencing, we own a copy of vec
seq4.front().value() = 10; // Okay, seq4 is non-const

const auto seq5 = flux::take(auto(vec), 3); // Declared const this time
seq5.front().value() = 10; // Error, seq5 is const
```

> Note that unlike the standard library, Flux's `front()` operation returns an `optional` to avoid undefined behaviour in the case where the sequence is empty, so we need to add `.value()` to access the element itself. And yes, `flux::optional<T>` allows `T` to be a reference.

## In summary ##

Like C++20 Ranges, Flux provides a [variety of sequence adaptors](https://tristanbrindle.com/flux/reference/adaptors.html) which you can use to build up a pipeline of operations. Ranges implicitly stores the address of lvalue non-`view`s passed to range adaptors, which can hide lifetime dependencies.

Flux on the other hand semantically always takes adaptor arguments by value, except that you need to make an explicit copy if copying might be expensive. To pass a reference to a sequence, you need to explicitly form one with `flux::ref` or `mut_ref`. By giving the latter a longer name and making it move-only, Flux tries to encourage you to use const references whenever possible and avoid having multiple mutable references to a sequence. Flux's explicit referencing also tries to make the const-complexity story a little less mysterious than with Ranges.

If you found this interesting and want to know more then you can find me [on whatever Twitter is called now](https://twitter.com/tristanbrindle), or [checkout Flux on Github](https://github.com/tcbrindle/flux).
