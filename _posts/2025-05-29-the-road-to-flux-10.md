---
layout: post
title: The Road to Flux 1.0
author: Tristan Brindle
permalink: /posts/the-road-to-flux-10
comments: false
tags: C++
---

I've just posted a long update about my plans for Flux [on the Github discussions page](https://github.com/tcbrindle/flux/discussions/242), but I figured it might be worth posting here as well for a wider audience.

If you've used Flux before -- or you're just curious about alternatives to std::ranges -- then hopefully you'll find it interesting.

Just to reiterate what it says below -- I'm very keen to get feedback about the plans outlined here. If you have any comments or questions then please head on over to [the Github page](https://github.com/tcbrindle/flux/discussions/242).

Without further ado...

<br>
<br>

---

<br>
<br>


Flux has been around for a couple of years now, and the feedback I get is mostly that people seem to think it's pretty good. I think it's pretty good too! But I'm also aware that the current design of the library has some shortcomings. Over the last few months I've spent a lot of time thinking about ways in which things can be improved, and experimenting with various alternative approaches.

Having settled on what I believe is a promising design for the library going forward, I wanted to give an update on the planned changes as we move towards Flux 1.0. I'm very keen to get feedback from Flux users (and potential users!) -- so if you have any thoughts or comments on the new design, please let me know!

There's quite a lot here, so please make yourself comfortable, grab a `${PREFERRED_HOT_BEVERAGE}` and we'll dive in...

# Motivation #

I guess I should make clear that the goals for Flux haven't changed: as ever, I want to offer broadly the same feature set as std::ranges, but in a way that is:
 * Safer by default
 * At least as performant (and hopefully more so in many cases)
 * Easier to use, with fewer tricky corner cases
 * As compatible as possible with existing STL code

Indeed, the changes proposed here are about *furthering* these goals.

One piece of feedback I've received repeatedly is that while people generally like Flux when they try it out (especially its performance vs ranges), they tend to find that it feels "too different" to the standard library. Flux does many things its own way, making it feel like there's a lot to learn. This can make it a tough sell, especially when people have already invested the time in learning about ranges.

One of the goals for Flux 1.0 therefore is to reduce the "diff" from standard C++ -- that is, to try to make Flux look and feel a bit more familiar.

Another piece of feedback I've received several times is that people would love to see either internal iteration or the cursor model (or both) proposed for standardisation. While I (obviously) agree that these would be excellent, with my SG9 hat on it would be an *incredibly* tough sell to bring the current Flux `sequence` model to the committee. It is in a sense both too large and too similar to C++20 ranges.

With this in mind, the next version of Flux will replace the existing `sequence` concept with two separate (but complementary) interfaces: one for single-pass algorithms, using internal iteration (`iterable`) and a more powerful `collection` concept (a simplified version of today's `multipass_sequence`).

All begin well, I hope to propose `iterable` for standardisation in the C++29 cycle, for which Flux will serve as a reference implementation. With luck, the reduced scope (and performance and safety benefits) will help convince LEWG that this would be a useful addition to the standard.

# Planned changes #

## Pipes, not dots ##

Reluctantly, we'll move from using member functions for chaining to overloading `operator|`, as in ranges and the forthcoming stdexec. This is worse for readability, worse for discoverability, worse for error messages and worse for compile times than using member functions -- but it's what the standard library does, and so we're going to follow suit. This means that Flux pipelines will change from today's

```cpp
auto total = flux::ref(vec)
               .filter(flux::pred::even)
               .map([](int i) { return i * i; })
               .sum();
```

to

```cpp
auto total = std::cref(vec)
             | flux::filter(flux::pred::even)
             | flux::map([](int i) { return i * i; })
             | flux::sum();
```

(Yes, it would be possible to support both syntaxes, but I'd rather not do that -- it would mean more code that needed to be maintained and tested, and would lead to confusion about which is the "proper" way to write pipelines. Let's just do it the standard way, even if that's not ideal.)

Note that, unlike ranges and stdexec (but like Range v3's actions) I plan on making terminal algorithms like `sum()` pipeable, just because I think it reads quite nicely. I'd also quite like to add these pipeable function objects to a nested inline namespace (tentatively, `flux::operations`?) so that you could say:

```cpp
using namespace flux::operations;

auto total = std::cref(vec)
    | filter(flux::pred::even)
    | map([](int i) { return i * i; })
    | sum();
```

to avoid repeating the namespace name, but without bringing *everything* in namespace `flux` into scope (rather like `using namespace std::literals`).

One thing that **won't** be changing is the ownership model for adaptors. Flux adaptors will continue to act like sinks, taking ownership of what you pass to them -- rather than the confusing situation with range views today, where they sometimes transfer ownership and sometimes act as reference types. (We'll also continue to reject non-trivially-copyable lvalues, so you'll have to explicitly copy things like `vector`s into an adaptor.)

This means that, as ever, you'll need to be explicit if you want to pass a sequence reference into a Flux adaptor. The best way to do this will be with `std::ref` or `std::cref` -- the current `flux::ref` and `flux::mut_ref` functions will be retired in favour of using the standard functions.

## Goodbye `sequence`, hello `iterable` ##

### Background ###

STL input iterators have always been a bit weird. For forward ranges, an iterator acts like a pointer to a range element; but for single-pass ranges this abstraction falls apart. Trying to use iterators with single-pass ranges is the proverbial square peg in a round hole, and has always felt that way in practice.

The same is true of cursors in Flux today. Cursors simply aren't a good fit for single-pass iteration. I was aware of this when I first started working on the library, but I felt that it was better to try to come up with a single, generalised interface for *all* sequences, and that perfect was the enemy of good. With the benefit of hindsight, I've come to think that trying for a single, universal interface was probably the wrong decision.

A position-based interface (be it cursors or iterators) naturally means separating the "read" and "advance" operations. This is ideal for multipass sequences, where elements are readily accessible in memory or can be cheaply recreated. But single-pass sequences are isomorphic to streams, where in most cases "read and advance" is more naturally implemented as single operation. This mismatch is how we end up with things like the notoriously poor codegen for `views::filter()`, the infamous `transform | filter` "terrible problem", and the "take(5)" issue for stream wrappers.

Separately, when I started working on Flux, internal iteration was something of an afterthought -- I noticed it was possible to add it as a customisation point, and so I did so. It was only later that I realised how important this is for performance -- and that what most people like best about Flux happens because they're using code-paths where the internal iteration optimisation kicks in.

So, given that the current single-pass iteration scheme isn't very good, and internal iteration is great, we're going to start again, and introduce a new single-pass iteration model based around internal iteration.

### Iteration contexts ###

At the heart of the new model is what's called an `iteration_context`. If we had proper concepts in the language, the `iteration_context` interface would be defined like this:

```cpp
enum class iteration_result : bool {
    incomplete, complete
};

template <typename T>
explicit concept iteration_context {
    typename element_type;

    template <typename Pred>
        requires callable<Pred&, bool(element_type)>
    auto run_while(Pred&& pred) -> iteration_result;
};
```

That is, an iteration context needs to provide an `element_type` typedef, and a `run_while` member function which takes a unary predicate and returns an `iteration_result`.

When `run_while` is called, the context invokes the user's predicate with each element of the sequence in succession; if the predicate returns false, then `run_while` stops and returns `iteration_result::incomplete`. Otherwise, when the context reaches the end of the sequence, it returns `iteration_result::complete`. If `run_while` returns `incomplete`, it is expected that we can immediately call `run_while` again to restart iteration from the *next* element.

Iteration contexts will typically hold pointers or references to their parent sequences, and are expected to be **non-copyable** and **non-movable** -- this both discourages people from using them as anything except short-lived local variables, and saves implementors from having to guarantee validity after a move (which might be tricky in some cases).

Because iteration contexts hold references, they should be considered as "borrowing" from their parent sequence, to use Rust terminology. This means that, for safety, we should adhere to the so-called "law of exclusivity": we must not call a mutating function on the sequence while an iteration context is "live", and if we have a mutating iteration context then we should avoid *any* use of the parent sequence. One way of thinking about this is as if the sequence gets moved into the context when we create it, and restored in the context destructor.

The simplest possible iteration context might be something like so:

```cpp
struct fours {
    using element_type = int;

    iteration_result run_while(auto&& pred) {
        while (true) {
            if (!pred(4)) {
                return iteration_result::incomplete;
            }
        }
    }
};
```

This infinitely generates the number 4 until the user is fed up enough to return `false` from their predicate.

A more useful example is the following:

```cpp
template <std::ranges::forward_range R>
struct range_iteration_context {
    using Iter = std::ranges::iterator_t<R>;
    using Sent = std::ranges::sentinel_t<R>;

    Iter iter;
    Sent sent;

    using element_type = std::iter_reference_t<Iter>;

    iteration_result run_while(auto&& pred)
    {
        while (iter != sent) {
            if (!pred(*iter++)) {
                return iteration_result::incomplete;
            }
        }
        return iteration_result::complete;
    }
};
```

This is a generic iteration context for any `forward_range`. A generic context for any `input_range` is only very slightly more complicated, since we don't have postfix `operator++` on iterators in that case.

Algorithms on iteration contexts are as follows:

* `run_while`

  ```cpp
  template <iteration_context Ctx, typename Pred>
      requires callable<Pred&, bool(Ctx::element_type)>
  auto run_while(Ctx& ctx, Pred&& pred) -> iteration_result
  ```

  Just returns `ctx.run_while(pred)`

* `step`

  ```cpp
  template <iteration_context Ctx, typename Func>
      requires callable<Func&, void(Ctx::element_type)>
  auto step(Ctx& ctx, Func func) /* -> see below */;
  ```

  Performs a single iteration step and calls `func` with the next sequence element, wrapping the return value of `func` in an optional. For example, given our `fours` context above, we could write

  ```cpp
  auto opt = step(fours, [](int i) { return i * i; });
  assert(opt.value() == 16);
  ```

  Formally, let `R` be `invoke_result_t<Func&, Ctx::element_type>`. If `is_void_v<R>` is true, `step` is equivalent to

  ```cpp
  bool called = false;
  run_while(ctx, [&](auto&& elem) {
      std::invoke(func, FWD(elem));
      called = true;
      return false;
  });
  return called;
  ```

  Otherwise, let `S` be `std::decay_t<R>` if `R` is an rvalue reference type, or `R` otherwise. Then `step` is equivalent to:

  ```cpp
  optional<S> out = nullopt;
  run_while(ctx, [&](auto&& elem) {
       out.emplace(std::invoke(func, FWD(elem)));
       return false;
  });
  return out;
  ```

 * `next_element`

   ```cpp
   template <iteration_context Ctx>
   auto next_element(Ctx& ctx) -> optional<Ctx::element_type>;
   ```

    Equivalent to `step(ctx, std::identity{})` -- that is, it returns an optional containing the next sequence element, or a `nullopt` if there were no more elements.

    This function allows single-step iteration through an iterable sequence, equivalent to the Rust iteration model.

    If the element_type is a reference then care must be taken with this function, as the reference may be invalidated by a subsequent call to `run_while`, `step` or `next_element`.


### Iterables ###

While iteration contexts actually perform the low-level operations, one of the design goals is that they should mostly be considered an implementation detail. **Most users won't interact with iteration contexts directly**. Rather, the intention is that end users will compose pipelines out of higher-level algorithms on **iterables** -- that is, things we can iterate over.

The `iterable` concept is defined as:

```cpp
template <typename T>
concept iterable = requires (T& t) {
    { iterate(t) } -> iteration_context;
};
```

i.e. it is satisfied when `flux::iterate(it)` is valid and returns an `iteration_context`. In turn,`flux::iterate(it)` returns

 * `flux::iterable_traits<R>::iterate(it)`, where `R` is `std::remove_cvref_t<decltype(it)>`, if `iterable_traits<R>` is not generated from the primary template
 * Otherwise, `it.iterate()` if that is a valid function call
 * Otherwise, a specialisation of `range_iteration_context` if `decltype(it)` satisfies `input_range`

If none of these apply then the call to `iterate()` is ill-formed.

Translated into english, this means that we're `iterable` if we have a member function `iterate()` which returns an `iteration_context`, or if we provide a specialisation of `flux::iterable_traits` along the lines of

```cpp
template <>
struct flux::iterable_traits<MyIterable> {
    static auto iterate(MyIterable& it) -> MyIterContext
    {
        ...
    }
};
```

It's worth noting here that many iterables will provide both const and non-const overloads of `iterate()`, returning different context types. Unlike STL iterators however, there is no requirement for convertibility between non-const and const iteration contexts.

The `iterable` concept is equivalent to today's single-pass `sequence`. The plan for Flux is that all of the current single-pass algorithms and adaptors will be modified to operate on `iterable`s instead.

For example, `for_each` can be implemented on iterables as:

```cpp
template <iterable It, typename Func>
    requires callable<Func&, void(iterable_element_t<It>)>
auto for_each(It&& it, Func func) -> Func
{
    auto ctx = iterate(it);
    run_while(ctx, [&](auto&& elem) {
        (void) std::invoke(func, FWD(elem));
        return false;
    });
    return func;
}
```

while `fold()` can be written as roughly:

```cpp
template <iterable It, typename Func, typename Init = iterable_value_t<It>>
    requires foldable<...>
auto fold(It&& it, Func func, Init init = {}) -> Init
{
    auto ctx = iterate(it);
    run_while(ctx, [&](auto&& elem) {
        init = std::invoke(func, move(init), FWD(elem));
        return true;
    });
    return init;
}
```

Algorithms which operate on multiple sequences in lock-step generally can't be written directly in terms of `run_while`; instead, we can use `step` or `next_element` to advance each sequence by a single element at a time. For example, here is how we could write `equal`:

```cpp
template <iterable It1, iterable It2>
   requires std::equality_comparable_with<
       iterable_element_t<It1>,
       iterable_element_t<It2>>
auto equal(It1&& it1, It2&& it2) -> bool
{
    iteration_context auto ctx1 = iterate(it1);
    iteration_context auto ctx2 = iterate(it2);

    while (true) {
        auto opt1 = next_element(ctx1);
        auto opt2 = next_element(ctx2);

        if (opt1 && opt2) {
            if (*opt1 != *opt2) {
                return false;
            }
        } else if (opt1 || opt2) {
            return false;
        } else {
            return true;
        }
    }
}
```

As ever, writing iterable *adaptors* involves a bit more work than terminal algorithms, but generally less so than today's `sequence` adaptors (and much less so than most C++20 range adaptors). Typically, an adaptor will provide an iteration context which wraps the context of its base iterable. For example, the `map` adaptor (`transform_view` in C++20) would look something like this:

```cpp
template <iterable Base, typename MapFn>
struct map_adaptor {
    Base base;
    MapFn map_fn;

    template <typename BaseCtx, typename Fn>
    struct context_type {
        BaseCtx base_ctx;
        Fn fn;

        using element_type =
            std::invoke_result_t<Fn&, typename BaseCtx::element_type>;

        auto run_while(auto&& pred) -> iteration_result
        {
            return base_ctx.run_while([&](auto&& elem) {
                return std::invoke(pred, std::invoke(fn, FWD(elem)));
            });
        }
    };

    auto iterate()
    {
        return context_type{.base_ctx = flux::iterate(base),
                            .fn = std::ref(map_fn)};
    }

    auto iterate() const requires iterable<Base const>
    {
        return context_type{.base_ctx = flux::iterate(base),
                            .fn = std::ref(map_fn)};
    }
};
```

### Reverse iteration ###

Reverse iteration in Flux today works in almost exactly the same was as it does in the STL. We have a `bidirectional_sequence` concept which "inherits from" `multipass_sequence` and allows the user to decrement cursors. We then have a `reverse()` adaptor which swaps over the increment and decrement operations on cursors, with some fiddly off-by-one messing about in the read operation (again, just like reverse iterators).

Happily, the new iterable model allows us to make a dramatic simplification to this arrangement.

We can define a `reverse_iterable` concept as

```cpp
template <typename T>
concept reverse_iterable = iterable<T> &&
    requires (T& t) {
        { reverse_iterate(t) } -> iteration_context;
    };
```

where `flux::reverse_iterate(it)` is defined similarly to `iterate()`. In other words, we require that `reverse_iterate()` returns an iteration context -- one that just happens to generate the elements from back to front.

With this concept in place, we can provide a generic reverse-iterating context for all bidirectional, common ranges:

```cpp
template <std::ranges::bidirectional_range R>
    requires std::ranges::common_range<R>
struct range_reverse_iteration_context {
    using Iter = std::ranges::iterator_t<R>;

    Iter const start;
    Iter iter;

    using element_type = std::iter_reference_t<Iter>;

    auto run_while(auto&& pred) -> iteration_result
    {
        while (iter != start) {
            if (!pred(*--iter)) {
                return iteration_result::incomplete;
            }
        }
        return iteration_result::complete;
    }
};
```

It's worth noting that unlike `std::reverse_iterator`, this formulation doesn't need to perform an extra decrement before each dereference, reducing the number of per-element operations and likely allowing more optimisation opportunities. Also, importantly, it means that iterables can be reversible *without* being multipass, unlike the STL and current Flux models.

Most adaptors can support reverse iteration without too much trouble. For example, our `map_adaptor` above requires very little modification:

```cpp
template <iterable It, typename MapFn>
struct map_adaptor
{
    // As before...

    auto reverse_iterate() requires reverse_iterable<Base>
    {
        return context_type{.base_ctx = flux::reverse_iterate(base),
                            .fn = std::ref(map_fn)};
    }

    auto reverse_iterate() const requires reverse_iterable<Base const>
    {
        return context_type{.base_ctx = flux::reverse_iterate(base),
                            .fn = std::ref(map_fn)};
    }
};
```

Providing a `reverse_adaptor` (like `std::views::reverse`) is also very simple -- we just switch over the calls to `iterate` and `reverse_iterate`:

```cpp
template <reverse_iterable Base>
struct reverse_adaptor {
    Base base;

    auto iterate() { return flux::reverse_iterate(base); }

    auto iterate() const requires reverse_iterable<Base const>
    {
        return flux::reverse_iterate(base);
    }

    auto reverse_iterate() { return flux::iterate(base); }

    auto reverse_iterate() const requires iterable<Base const>
    {
        return flux::iterate(base);
    }
};
```

### Iterables and ranges ###

As we've seen, under the new model, *every* C++20 `input_range` is automatically a `flux::iterable`, and every bidirectional common range is `reverse_iterable`. This is a huge improvement on the status quo, where only sized, contiguous ranges are automatically Flux `sequence`s.

This means that going from the ranges world to the Flux world couldn't be easier. You can pass *any* C++20 range into a Flux algorithm, exactly as you would for a `std::ranges` algorithm. You can also pass any range into a compatible Flux adaptor, without needing to do anything special. With the switch to pipes as well, this means that Flux will (hopefully) become a near drop-in replacement for view pipelines, but with improved performance in many common cases.

Going in the opposite direction, from the Flux iterable world to ranges, is a little bit trickier, for a couple of reasons:
 * Firstly, in C++20, iterators are required to be (at least) movable, but Flux iteration contexts are not -- so we cannot store an iteration context inside an iterator, even if we wanted to, without unappealing workarounds like allocating.
 * Secondly, with iterators, repeated calls to read (without an intervening advance) are valid -- but iteration contexts fuse together the read and advance operations, so we can't do one without the other. This means that we will need to cache an element somewhere -- but again, we cannot necessarily do this in an iterator, because there is no requirement that sequence elements be moveable.

What this means is that we need to use an adaptor class -- a range object -- to hold both the iteration context and the cached element. This range can then hand out input-only iterators, like so:

```cpp
template <iterable It>
struct as_range_adaptor
{
private:
    It iterable;
    iteration_context_t<It> ctx = iterate(iterable);
    next_element_t<It> elem = next_element(ctx);

    struct iterator : move_only {
        as_range_adaptor* parent;

        using value_type = iterable_value_t<It>;
        using difference_type = std::ptrdiff_t;

        auto operator*() const -> decltype(auto)
        {
            return *parent->elem;
        }

        auto operator++() -> iterator&
        {
            parent->elem = next_element(parent->ctx);
            return *this;
        }

        void operator++(int) { ++*this; }

        auto operator==(std::default_sentinel_t) const -> bool
        {
            return !parent->elem.has_value();
        }
    };

public:
    auto begin() -> iterator { return iterator{.parent = this}; }

    static auto end() -> std::default_sentinel_t { return {}; }
};
```

Users can then move from Flux to ranges by piping into `as_range`, which returns an `as_range_adaptor`. For example:

```cpp
auto rng = flux::iota(0, 10)
         | flux::filter(flux::pred::even)
         | flux::map([](int i) { return i * i; })
         | flux::reverse
         | flux::as_range;

my_range_algorithm(rng.begin(), rng.end());
```

One thing that's worth noting is that while `rng` here is a range whose elements are lazily generated, it is *not* a `view`, because views are required to be at least moveable. This is turn means that we can't pass an rvalue "as_range" directly into a C++20 view function -- we need to save it as a named variable first, and then pass it as an lvalue:

```cpp
/* Error: as_range returns a non-movable range, which we cannot pass
   into the transform_view */
auto view = flux::iota(0, 10)
          | flux::filter(flux::pred::even)
          | flux::as_range
          | std::views::transform([](int i) { return i * i; }); // ERROR!

/* Okay: pass the range as an lvalue instead */
auto rng = flux::iota(0, 10)
         | flux::filter(flux::pred::even)
         | flux::as_range;

auto view = rng | std::views::transform([](int i) { return i * i; });
```

While this is somewhat unfortunate, the fact that this only applies to single-pass iterables, there is an easy workaround, and Flux provides a wealth of adaptors of its own, will hopefully mean that it doesn't come up too often in practice.

#### Special consideration: the range-based for loop ####

Probably the most common reason that people will want to convert iterables to ranges is so that they can use a range-based for loop. While they can do so with `as_range`, it's a bit annoying for users to need to remember to add that to the end of their Flux pipelines in order to use the convenient built-in syntax.

Fortunately there is something we can do. It turns out that if you look at the way the range-based for loop desugars, the requirements on the "iterator" returned by `begin()` are much less strict than the C++20 `std::ranges` concepts. In particular, there is no requirement that such "iterators" be moveable or copyable.

This means that we could, in theory, add `begin()` and `end()` methods to Flux iterable adaptors which return a non-moveable "pseudo-iterator", specifically for use with range-for loops:

```cpp
template <iterable It>
struct pseudoiterator {
private:
    iteration_context_t<It> ctx;
    next_element_t<It> elem;

public:
    explicit pseudoiterator(It& it)
        : ctx(iterate(it)),
          elem(next_element(ctx))
    {}

    auto operator*() const -> decltype(auto) { return *elem; }

    auto operator++() -> pseudoiterator
    {
        elem = next_element(ctx);
        return *this;
    }

    auto operator==(std::default_sentinel_t) const -> bool
    {
        return !elem;
    }
};
```

This would mean that we could use range-based for loops "naturally":

```cpp
for (int i : flux::iota(0, 10) | flux::filter(is_even)) // works
    do_something(i);
```

The downside of this approach is that it might be confusing for users if `.begin()` is valid to call, and returns something that looks a lot like an iterator, but won't work with C++20 (or many C++17) algorithms without using `as_range`.

(Of course, the ideal solution to this problem would be to teach the language for loop about the `iterable` concept. Maybe one day...)

## Collections ##

As we've mentioned, the `iterable` concept is all about single-pass iteration, and it turns out that you can do an awful lot just with this. But some algorithms require that we are able to visit the same location in a sequence more than once -- that is, multi-pass iteration.

The current version of Flux uses the `multipass_sequence` concept for this purpose. In the next version, this will be replaced by the much simplified `collection` interface instead.

### Positions ###

In Flux today, a `cursor` for a multipass sequence represents a position in that sequence, rather like a generalisation of an array index. (This is in contrast to iterators, which are a generalisation of array *pointers*.)

In moving to `collection`, what a cursor represents won't change -- but the name will. Freed from the need to support single-pass sequences (where there is no meaningful notion of "place"), we can switch to calling cursors what they really are: **positions**.

As with multipass cursors in today's Flux, collection positions are required to model `std::regular` -- that is, they must be default constructible, copyable, and equality comparable. We are also adding an addition requirement on positions: they must have a strict total order.

In other words, given two positions `p1` and `p2`, as well as being able to ask (as ever) whether `p1 == p2`, we also want to be able to ask whether `p1 < p2`, that is, if `p1` represents an earlier position in a collection than `p2`.

The goal for this is safety. If positions are always ordered, then we can easily perform bounds checking for all collections. In particular, it means that we can also perform bounds checking for arbitrary *slices* of collections. This is particularly important if different mutable slices get passed to different threads, where it is vital that we ensure that each thread does not write beyond its assigned bounds.

Formally, the `position` concept looks like this:

```cpp
template <typename P>
concept position =
    std::regular<P> &&
    std::three_way_comparable<P, std::strong_ordering>;
```

### Collection concept ###

In order to define a type as a `collection`, users need to provide implementations of four functions. As with the `iterable` concept, these can either be member functions of the type itself, or they can be static members of a user-defined specialisation of `flux::collection_traits<T>`.

The required functions are:

 * `begin_pos`

   ```cpp
   template <typename C>
   auto begin_pos(C const&) -> position auto;
   ```

   As you might expect, returns the start `position` of the collection.

 * `end_pos`

   ```cpp
   template <typename C>
   auto end_pos(C const&) -> position_t<C>;
   ```

   This returns the end position of the collection, which must be the same type as `begin_pos`

 * `inc_pos`

   ```cpp
   template <typename C>
   void inc_pos(C const&, position_t<C>&);
   ```

   Increments the given position so that it points to the next element of the collection.

 * `read_at_unchecked`

   ```cpp
   template <typename C>
   decltype(auto) read_at_unchecked(C&, position_t<C> const&);
   ```

   Returns the element at the given position. The "unchecked" part means that the implementor may assume that `is_readable_position` (see below) is `true` on entry to this function -- that is, someone else has already taken care of bounds checking. This allows us to implement things like `try_read_at()` without double bounds checks.

One thing to note is that the first three of these functions take the collection by const reference. This is a change from the existing `multipass_sequence` requirements, with the goal of making life easier for people implementing the collection protocol, as well as making things conceptually simpler overall. As always, it is possible (and common) for collections to provide two `read` overloads for const- and non-const element access.

Another thing to note is that all collections are required to be able to provide an end position. This means that, unlike iterables, collections are always finite. (I don't believe it's possible to write an infinite collection with ordered positions without some sort of `BigInt` type anyway, but if I'm wrong about that please let me know.) The current Flux `bounded_sequence` concept is no more, as all collections are now "bounded".

We can now define some further collection algorithms, using the above functions:

 * `next_pos`

   ```cpp
   template <collection C>
   auto next_pos(C const& col, position_t<C> pos) -> position_t<C>;
   ```

   Returns the next position after `pos` as a functional update. Equivalent to

   ```cpp
   inc_pos(col, pos);
   return pos;
   ```

 * `is_valid_position`

   ```cpp
   template <collection C>
   auto is_valid_position(C const& col, position_t<C> const& pos) -> bool
   ```
   Equivalent to `pos >= begin_pos(col) && pos <= end_pos(col)`

 * `is_readable_position`

   ```cpp
   template <collection C>
   auto is_readable_position(C const& col, position_t<C> const& pos) -> bool
   ```

   Equivalent to `pos >= begin_pos(col) && pos < end_pos(col)`

   (Note that `end_pos(c)` is a *valid* but not *readable* position.)

 * `read_at`

   ```cpp
   template <collection C>
   decltype(auto) read_at(C& col, position_t<C> const& pos)
   ```

   Performs bounds-checked read access. This should be the default read operation for collections.

   Equivalent to

   ```cpp
   if (!is_readable_position(col, pos)) {
       runtime_error(); // fail-fast
   }
   return read_at_unchecked(col, pos)
   ```

   except that bounds checking may be performed at compile time if possible (see [this blog post](https://tristanbrindle.com/posts/compile-time-bounds-checking-in-flux) for details).

 * `try_read_at`

   ```cpp
   template <collection C>
   auto try_read_at(C& col, position_t<C> const& pos)
       -> optional<collection_element_t<C>>
   ```

   If `is_readable_pos(col, pos)` is true, returns an optional containing the result of `read_at_unchecked(col, pos)`. Otherwise, returns a disengaged optional.

We can also implement a generic iteration context for collections:

```cpp
template <collection C>
struct collection_iteration_context : immovable {
    C* ptr;
    position_t<C> pos;
    position_t<C> const end;

    using element_type = collection_element_t<C>;

    auto run_while(auto&& pred) -> iteration_result
    {
        while (pos < end) {
            if (!pred(read_at_unchecked(*ptr, std::exchange(pos, next_pos(*ptr, pos))))) {
                return iteration_result::incomplete;
            }
        }
        return iteration_result::complete;
    }
};
```

Thus every collection can be iterable, and indeed the `collection` concept subsumes `iterable`. Of course, the above context type is just used as the fallback default; individual `collection` implementations may chose to provide a custom `iterable` implementation instead.

### Collection refinements ###

#### Permutable collection ####

A permutable collection additionally implements an extra customisation point, `swap_at_unchecked`:

```cpp
template <collection C>
void swap_at_unchecked(C& col, position_t<C> const& p1, position_t<C> const& p2);
```

As you might imagine, this swaps the elements at the two positions, with the assumption that both positions are readable. A default implementation of this function is provided when `collection_element_t<C>` is a non-const lvalue reference.

Of course, we also provide a bounds-checking wrapper, which should be preferred:

```cpp
template <collection C>
void swap_at(C& col, position_t<C> const& p1, position_t<C> const& p2)
{
    if (!(is_readable_pos(col, p1) && is_readable_pos(col, p2)) {
        runtime_error();
    }
    swap_at_unchecked(col, p1, p2);
}
```

The purpose of this function is so that we can implement in-place algorithms like `rotate()` and `swap()` on collections which return proxy references, like `zip`.

(It will also be necessary if we ever get a borrow checker for C++...)


#### Bidirectional collection ####

A collection can be made bidirectional by providing a `dec_pos` implementation which, unsurprisingly, decrements a position. We also provide a `prev_pos` function as a counterpart to `next_pos`. Obviously, all bidirectional collections are also `reverse_iterable`.

[Indeed, now that we have `reverse_iterable` it might be worth looking at whether `bidirectional` "pulls its weight" in the concept hierarchy.]

#### Random-access collection ####

A random-access collection additionally provides implementations of two more customisation points:

 * `distance`

   ```cpp
   template <collection C>
   auto distance(C const& col,
                 position_t<C> const& from,
                 position_t<C> const& to) -> int_t;
   ```

   Returns the number of elements between `from` and `to`, which may be negative if `to < from`, in constant time. The size of a random-access collection can be found with `distance(c, begin_pos(c), end_pos(c))`

 * `offset_pos`

   ```cpp
   template <collection C>
   void offset_pos(C const& col, position_t<C>& pos, int_t offset);
   ```

   Adjusts `pos` by `offset` places (which may be negative) in constant time.

   If `offset > distance(col, pos, end_pos(col))` or `offset < distance(col, pos, begin_pos(col))` then the resulting position will be invalid. It is unspecified whether this function raises a runtime error in this case.

   [Note: some implementations may need to catch an out-of-bounds jump immediately, to prevent undefined behaviour. Other implementations may choose to defer the bounds check until the next attempted read.]

#### Contiguous collection ####

A contiguous collection is a random-access collection whose elements are stored in a contiguous array in memory.

The `contiguous_collection` concept requires that the collection's element type is an lvalue reference, and that the collection implements the `as_span` customisation point:

 * `as_span`

   ```cpp
   template <random_access_collection C>
   auto as_span(C& col) ->
       std::span<std::remove_reference_t<collection_element_t<C>>>;
   ```

   Returns a `span` over the elements of `col`.

   [Note: `contiguous_range` and the current Flux `contiguous_sequence` instead require a `data()` function which returns a raw pointer to the start of the underlying storage. Returning a `span` instead is moderately safer, as the element count has less chance of getting lost.]

### Collections and ranges ###

For safety, collections need to be able to perform bounds checking. We can do this  cheaply for sized, contiguous ranges, by using an integer index as the position type and doing `ranges::data()[idx]` in the read operation -- in other words, exactly what we do in Flux today. We can also extend this to all sized random-access ranges by using `ranges::begin()[idx]` in the `read` implementation -- which may be slightly more expensive than using a raw pointer, but is still guaranteed to be O(1).

So, going from the ranges world to the Flux world can be summarised as follows:
 * If `R` is a sized, random-access range, then it is also automatically Flux `random_access_collection`. If `R` is additionally a `contiguous_range` then it is also a `contiguous_collection`. This applies to things like raw arrays, `std::array`, `vector`, `string`, `string_view`, `span` etc, as well as various adapted views.
 * Otherwise, `R` is a Flux `iterable` only

For node-based data structures like `std::list` or `std::unordered_map`, there is no simple way for us to enumerate elements or perform bounds checking, so they will be `iterable` only. (But it's worth remembering that Flux's `iterable` is isomorphic to Rust's `iterator`, which people seem broadly satisfied with -- there is still a lot of functionality available even without `collection` conformance.)

Going in the other direction, things are much simpler for `collection`s than for `iterable`s: every Flux `collection` can be (at least) a `forward_range`, without any need for `as_range` or pseudoiterators or anything like that. Just as we do today for `multipass_sequence`s, we can define a `collection_iterator`:

```cpp
template <collection C>
struct collection_iterator {
    C* ptr;
    position_t<C> pos;

    using reference = collection_element_t<C>;
    using value_type = collection_value_t<C>;
    using difference_type = std::ptrdiff_t;
    using iterator_category = std::forward_iterator_tag;

    auto operator*() const -> reference
    {
        return read_at(*ptr, pos);
    }

    auto operator++() -> iterator&
    {
        inc_pos(*ptr, pos);
        return *this;
    }

    auto operator++(int) -> iterator
    {
        auto tmp = *this;
        ++*this;
        return tmp;
    }

    auto operator==(iterator const& rhs) const -> bool = default;
};
```

...along with about a billion more conditional operator overloads for bidir and random-access iterator support. We can then provide every `collection` implementation with `begin()` and `end()` members returning `collection_iterator` (just as we do today with the almost-identical `sequence_iterator`).

## A special note about `filter` ##

When we use a filtering adaptor -- in Flux, ranges, or any similar library -- the result of the filter predicate determines which elements are part of the output sequence.

That sounds obvious, but it has an important consequence. It means that mutating elements in a way that changes the result of filter predicate is equivalent to adding or removing elements from the resulting sequence. Doing so *while we're iterating over the sequence* is really asking for trouble, in the same way as adding or removing elements of a vector while iterating over it.

Now as it turns out, if we're only doing a single pass over the data -- or more precisely, if we only evaluate the predicate once per element -- then we can get away with arbitrary mutation, because it never actually gets observed. But if we're doing multiple passes, then changing membership of the filtered sequence "on the fly" is very very likely to cause problems. Again, this is true whether we're talking about C++ ranges, Flux, D ranges, or any other library (yes, even Rust).

This would be fine, except that mutating elements while filtering is something that people like to do fairly often. With Flux, fortunately, the worst that will happen is usually a failed bounds check at run time. But it would still be better to prevent people from getting into difficulty in the first place, if we can.

For this reason, I'm proposing that the new `flux::filter` will **not** be a `collection`: that is, it will be single-pass only. Thanks to our new iteration model, filter can still be `reverse_iterable`, so I'm hopeful that most of what people want to do can still be achieved even without multi-pass.

For the remaining cases, I propose to add a second adaptor, tentatively called `flux::filter_collection`. As the name suggests, this will model `collection` (or `bidirectional_collection` if the base allows), but importantly will yield "constified" elements. This doesn't completely prevent people from mutating elements while iterating -- there's always `const_cast`! -- but it will hopefully serve as a deterrent.

Finally, let's talk about caching. Ranges has the requirement that `begin()` and `end()` are constant-time, and Flux will have the same requirement for `begin_pos()` and `end_pos()` on collections. In order to satisfy this requirement, ranges' `filter_view` caches the first element which matches the filter, so that the first call to `begin()` is O(N), but subsequent calls are O(1). Because of this, `filter_view` does not support const iteration.

Given the definition of our new `collection` protocol above, we're in a tricky situation when it comes to implementing `filter_collection`. We require that `begin_pos()` is callable on a **const** instance of a collection, while also requiring that it returns an answer in constant time. If we wanted to use caching in order to satisfy the second requirement (at least at the second time of asking), then the only solution would be a `mutable` member variable, which would in turn necessitate a mutex (or similar mechanism) to achieve thread safety -- definitely not something you want in what is intended to be a high-performance library.

My proposed solution to this conundrum is to simply not do any caching at all in `filter_collection`, and instead slap a big fat warning in the documentation that it might not meet the complexity requirements -- [which is exactly what Swift does](https://developer.apple.com/documentation/swift/lazyfiltercollection#discussion), in fact.

I feel fairly confident that this is the right trade off, in part because the current `flux::filter` doesn't do any caching either, *and I'm pretty sure no-one has ever noticed*. In the new Flux, using `filter_collection` will probably be quite rare, and using in specific ways that would trigger quadratic behaviour should hopefully be very, very rare indeed.

# In conclusion #

This has been quite a mammoth update, so thank you for making it to the end!

I'm pretty excited about the upcoming changes, and I believe Flux will become even more awesome than it already is once everything is finished. But I'm also very keen to gather feedback from the community. Does all this sound good? Are there improvements I should incorporate? Am I barking completely up the wrong tree? Please let me know below.

Meanwhile, I've started implementing these changes on the [devel branch on Github](https://github.com/tcbrindle/flux/tree/devel). Right now there isn't an awful lot to see but if you're interested in following the latest developments, that's the place to do it.

-- Tristan

