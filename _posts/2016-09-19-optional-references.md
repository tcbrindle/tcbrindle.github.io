---
layout: post
title: The case for optional references
author: Tristan Brindle
permalink: /posts/optional-references
comments: true
tags: C++
---

I have a confession to make. Whenever I've come across code that looks like this:

```cpp
struct example {
    example() = default;

    example(std::string& s) : str_{s} {}

private:
    boost::optional<std::string&> str_{};
};
```

there is a little voice inside my head that whispers *"why didn't you just use a pointer?"*. Like so, for instance:

```cpp
struct example {
    example() = default;

    example(std::string& s) : str_{&s} {}

private:
    std::string* str_ = nullptr;
};
```

This is equivalent to the first example, except that it's slightly less typing, it doesn't have any dependencies, and feels in some sense "cleaner". I personally have always preferred it.

Except, I was wrong. After attending Bjarne Stroustrup's keynote and [this excellent talk](https://cppcon2016.sched.org/event/7nDJ/using-types-effectively) at Cppcon this morning, I'm persuaded that optional references are a good thing. In this post I hope to be able to convince you of the same.

# std::optional<T&>? #

It seems I'm not the only one who preferred the pointer form. Whereas `boost::optional` allows reference types, the standardised version `std::optional` (forthcoming in C++17, and available now in libstdc++ and libc++ as `std::experimental::optional`) does not. [N3527](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2013/n3527.html#rationale.refs), revision 3 of the paper proposing `std::optional`, says:

> The intention is that the Committee should have an option to accept optional values without optional references, if it finds the latter concept inacceptable [sic]

The paper also goes on to talk about one potential pitfall of `optional<T&>`, namely assignment, that we'll talk about later. But first, the pros.

# Expressing intent #

Pointers represent memory addresses. When we say

```cpp
int a = 0;
int* b = &a;
```

we are assigning the *address* of `a` to the variable `b`. But if we're using a pointer as an optional reference then this is misleading: we don't care about  memory addresses at all. We only care if it's `nullptr` or not. Compare the above with

```cpp
int a = 0;
optional<int&> b{a};
```

Here, we are explicitly saying that `b` is either empty, or a reference to an `int`. We are **expressing our intent** much more clearly.

Bjarne made the point in his keynote (as he has done several times before) that he would like C++ to be more friendly to novice programmers. By introducing `std::array<T>`, the committee removed one of the major uses of raw pointers inherited from C; permitting `optional<T&>` would remove another. It would allow us to say "only use raw pointers in low-level code", and would make the learning and teaching of C++ that little bit easier.

# Better syntax #

I don't think I'm alone in finding the syntax for declaring const pointers and pointers-to-const to be anything but intuitive. Any introduction to pointers contains an example such as the following:

```cpp
int i = 0;
int* a = &i;              // mutable pointer to mutable int
const int* a = &i;        // mutable pointer to const int
int const* b = &i;        // as above, pointer to const int
int* const c = &i;        // const pointer to mutable int
int const * const d = &i; // const pointer to const int;
```

(It's telling that even after many years of writing C and C++, I still had to double-check that to make sure I got it right.)

Compare this to the equivalent with optional references:

```cpp
int i = 0;
auto a = optional<int&>{i};
auto b = optional<const int&>{i};
const auto c = optional<int&>{i};
const auto d = optional<const int&>{i};
```

The use of the template syntax (and AAA style) immediately makes it clear whether it is the wrapper or the contained referee that is `const`.  The same cannot be said for `int * const`, especially for novices.

Again, there is an analogy with `std::array` vs C arrays: have you ever tried to pass a C array by reference to a function? The syntax is odd to say the least:

```cpp
void f(const std::string (&arg)[4]); // huh? are we taking the address of arg?
```

whereas with `std::array`, it's obvious what's going on:

```cpp
void f(const std::array<std::string, 4>& arg); // oh, I see
```

As above, clear, consistent, familiar syntax allows us to express our intent more clearly.

# No pointer arithmetic #

Optional references do not allow pointer arithmetic. This removes a potential source of errors:

```cpp
int* maybe_increment(int* pi)
{
    if (pi) {
        pi += 1;
    }
    return pi;
}
```

Did you spot the bug? The above is legal C++ and compiles without warning, but it's very unlikely that it does what the author intended. Compare this to the optional reference case:

```cpp
optional<int&> maybe_increment(optional<int&> oi)
{
    if (oi) {
        // oi += 1; // Error, no match for operator+=
        *oi += 1; // That's better
    }
    return oi;
}
```

# Consistency with `std::variant` #

Another new addition to C++17 is `std::variant`. One of the things that `variant` allows is to specify that an empty state is permitted by using `std::monostate`. For example:

```cpp
auto v = variant<monostate, int>{}; // v is either empty, or contains an int
```

Such a variant is conceptually the same as an `optional<int>`, except that the latter has a more specialised, friendlier API. This is in much the same way that `pair<int, float>` is conceptually the same as `tuple<int, float>`, except that it provides a more specialised API to directly access the two members.

But here's the thing: `variant<monostate, int&>` [is permitted](http://en.cppreference.com/w/cpp/utility/variant), behaving as if it contained a `std::reference_wrapper<int>`. So why not `optional<int&>`? Such an inconsistency is needless and confusing.

# Generic programming #

The lack of optional references means that generic code which deals with optionals is unnecessarily complicated.

N3527, referenced above, acknowledges this, saying:

> Users that in generic contexts require to also store optional lvalue references can achieve this effect, even without direct support for optional references, with a bit of meta-programming.

and providing the following workaround:

```cpp
template <class T>
struct generic
{
  typedef T type;
};

template <class U>
struct generic<U&>
{
  typedef std::reference_wrapper<U> type;
};

template <class T>
using Generic = typename generic<T>::type;

template <class X>
void generic_fun()
{
  std::optional<Generic<X>> op;
  // ...
}
```

Urgh. This should not be necessary.

# Zero cost #

Like `std::unique_ptr`, optional references can be implemented as a zero-overhead abstraction around a raw pointer. In fact, `boost::optional` does just this.

There is no reason `std::optional<T&>` could not do the same.

# The assignment problem (and what to do about it) #

Having given the plus points of allowing optional references, it's only fair to give some time to the potential downsides as well. In particular, deciding what to do about assignment appears to be a bit tricky.

Version 2 of the paper proposing `std::optional` (N3406) [goes into a bit more detail about the assignment problem](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2012/n3406#rationale.refs). In short, when assigning one optional reference to another, should it behave like a pointer, or like a built-in language reference?

```cpp
int i = 0;
int j = 1;

int* p1 = &i;
int* p2 = &j;
p2 = p1; // Pointer is rebound, p2 now points to i

int& r1 = i;
int& r2 = j;
r2 = r1; // No reference rebinding, j now equals 0

auto o1 = optional<int&>{i};
auto o2 = optional<int&>{j};
o2 = o1; // Should we rebind o2, or copy the value?
```

From what I can gather, this was one of the major points of contention regarding optional references. As the paper makes clear, there are arguments to be made both ways about how it should behave, and it was not obvious as the time the paper was proposed which way `optional` should go.

However, I believe that we should now firmly come down on the side of rebinding. Why? Recall from above that `optional<T>` can be regarded as a special case of `variant<monostate, T>`. So what does `variant<monostate, T&>` do when we assign to it? I don't have an implementation of `std::variant` to hand, but since it permits implementations to store references in a `reference_wrapper`, I believe it must rebind:

```cpp
int i = 0;
int j = 1;

auto v1 = variant<monostate, int&>{i};
auto v2 = variant<monostate, int&>{j};
v2 = v1; // v2 now contains a reference to j
```

So, whether the committee intended it or not, it has made a decision; `variant` rebinds references, and so following the principle of least surprise, `optional` should too. (For what it's worth, this is the [same as `boost::optional` does](http://www.boost.org/doc/libs/1_53_0/libs/optional/doc/html/boost_optional/rebinding_semantics_for_assignment_of_optional_references.html).)

# In summary #

By allowing optional references, the committee would add a zero-overhead abstraction which would allow C++ programmers to make their intent clearer, to make code easier to read, to reduce the possibility for errors, and make teaching the language easier. It would also remove an unnecesary incompatibility between `optional` and `variant`.

I don't know if there's still time to get this in to C++17, but if anybody who can attend committee meetings feels like proposing it, I'm happy to work on a paper with you...
