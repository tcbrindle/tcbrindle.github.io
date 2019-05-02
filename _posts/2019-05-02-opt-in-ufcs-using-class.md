---
layout: post
title: Opt-in UFCS with "using class"
author: Tristan Brindle
permalink: /posts/opt-in-ufcs
comments: false
tags: C++
---

Sadly, I've never had a chance to read Bjarne Stoustrup's *"Design and Evolution of C++"*. I suspect I'd find it fascinating, and that it would probably answer many questions I have in my mind about why things in C++ are the way they are today. Unfortunately the book seems to be in short supply (I assume it's out of print), and the prices for new copies on Amazon are rather on the steep side.

Nonetheless, many of the papers from the early days of C++ standardisation are still available online, and occasionally one turns up which lets us look back in time at how some of the C++ sausage was made. So it was last week, when  Arthur O'Dwyer, the Unstoppable Blogging Machine, wrote about [argument-dependent lookup (ADL)](https://quuxplusone.github.io/blog/2019/04/26/what-is-adl/). As usual with Arthur's prodigious output, the post was very good, but even more interesting to me was a link it contained to a gem of a paper from 1993.

The paper in question is Stroustrup's [N0262](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/1993/N0262.pdf), the original proposal for adding namespaces to C++. If you're interested in delving in to the history of C++ standardisation, I'd strongly recommend giving it a read.

What we ended up getting in C++98 was very close to what Stroustrup originally proposed. But much of the discussion is quite eye-opening, at least to me. For example:

 1. Even in 1993 -- five years before standard C++ would even exist! -- there were already concerns about backwards compatibility and migrating "legacy" code

 2. Stroustrup originally proposed an extended form of *using-declaration* which would have allowed you to add multiple names to the current scope in one go:

    ```cpp
    using ns::(foo, bar, baz);
     ```

    I'm curious as to why this was rejected.

 3. Stroustrup devotes Appendix B to discussing a hypothetical "what if?" design of making a namespace a special kind of class. He calls such a thing a "module". That's right, we nearly had C++ modules in 1993! (Well, kind of.)

## Classes and namespaces ##

One of the most interesting things about the paper is that (contrary to the hyptothetical "module" discussion above) Dr S. points out that in his design we can consider a class as a special kind of namespace:

> It has been suggested that a namespace should be a kind of class, but I don’t think that is a good idea (see Appendix B). The opposite, that a class is a kind of namespace, seems almost obviously true. A class is a namespace in the sense that all operations supported for namespaces can be applied with the same meaning to a class unless the operation is explicitly prohibited for classes.

 I'd never really looked at things this way before, but it suddenly makes the use of the `using` keyword to un-hide base class members look very natural -- it has exactly the same meaning as a *using-declaration* for namespace members. C++ is not a language for which the word "elegant" normally springs to mind, but in this case it almost feels appropriate.

Stroustrup actually takes this class/namespace equivalence to its logical conclusion, and proposes that we should be allowed to use a class name in a *using-directive*, i.e.

```cpp
struct X {
    static void f();
    void g();
};

void h()
{
    using namespace X; // a class is-a namespace!
    f(); // Ok, calls X::f()
    g(); // error: object missing for non-static member function
}
```

The rationale for this is worth quoting in its entirety:

> [S]hould it be allowed to specify a using-declaration or a using-directive for a class? My suggested answer is ‘‘yes.’’ My (weak) argument for ‘‘yes’’ is ‘‘why not?’’

(As an aside, one wonders how many features ended up in C++ simply because Stroustrup thought "why not?")

## Using class? ##

The above part of the proposal didn't make it into C++98, and to my knowledge has not been re-proposed since: we cannot currently say `using namespace X`, where `X` is a class name. But is it an entirely bad idea?

The rest of this post is a bit of an investigation into what we might be able to achieve by allowing it. All of the following is really a thought experiment: although I've used the word "proposal" in a few places, I'm definitely not (yet!) proposing that we add this to C++.

So let's imagine that we added this functionality today. Rather than spell it `using namespace X`, it would probably be less confusing to say `using class X` (or equivalently, `using struct X`), so let's go with that.

(Curiously, at least if my reading of the standard grammar is correct, this syntax is currently unused. We can say `using N::X` as a *using-delaration* for a class `X` in namespace `N`, but we cannot spell it `using class N::X`, even though an *elaborated-type-specifier* like this is valid in many other places. It appears this particular syntax gap is also exploited by the `using enum` proposal discussed later.)

Under our proposal, a `using class` directive would do the same thing as a `using namespace` directive: make all the names from that "namespace" available in the scope in which the directive appears. So for example:

```cpp
struct X {
    static void f(int);
    void g();
};

void f(double);

void h()
{
    f(3); // calls ::f(double)

    using struct X;
    f(3); // ::f and X::f considered, X::f is a better match

    g(); // ERROR: no object
}
```

So far, so... kind of pointless? This doesn't really seem to add any useful new functionality -- we can already make "static" member functions available in the enclosing namespace by making them `friend`s instead, and the fact that this introduces non-static member functions into the current scope -- but makes it impossible to call them -- seems to be asking for trouble.

### What if... we *could* call them? ###

But hang on a sec. What if, when made accessible in a scope via a `using class` directive, we *could* call a non-static member function, by explicitly passing the "implicit object" as the first argument to the call?

That is, what if we could do the following:

```cpp
struct X {
    void g(int);
};

void h()
{
    X x;
    using class X;
    g(x, 3); // okay, calls x.g(3)!
}
```

Specifically, for each non-static member function `f(params...) [cv]` in `X`, a synthesised non-member function `f(X [cv]&, params...)` is considered for the purposes of overload resolution (or `f(X [cv]&&, params...)` for an rref-qualified member). If the synthesised function is selected, then it is rewritten as a member function call. (The intention is that this matches the [current rules](http://eel.is/c++draft/over.match.funcs) for how the implicit object parameter is treated during overload resolution, and the way member and non-member operator overloads are considered in the same overload set.)

Again, when you look at just a simple example then it seems kind of pointless -- `g(x, 3)` is arguably less readable than `x.g(3)`. However, the magic happens when we bring templates into the mix. Let's consider the implementation of an imaginary STL-like algorithm:

```cpp
template <typename R, typename T>
bool contains(const R& range const T& value)
{
    using class R;

    auto first = begin(range);
    const auto last = end(range);

    while (first != last) {
        if (*first == value) {
            return true;
        }
        ++first;
    }

    return false;
}
```

Because of `using class R`, all of the members of `R` are made available in the current scope. This means that if `R` is, say, a specialisation of `std::vector`, then its `begin() const` and `end() const` member functions will be considered (and in all likelihood selected as an exact match) to initialise `first` and `last`.

With `using class`, it doesn't matter to the author of `contains()` how `begin` and `end` are implemented: whether they're members, static members, free functions or whatever. Any of the following could be passed to `contains()`:

```cpp
struct A {
    struct iterator;
    iterator begin() const;
    iterator end() const;
};

struct B {
    struct iterator;
    static iterator begin(const B&);
    static iterator end(const B&);
};

struct C {
    struct iterator;
    friend iterator begin(const B&);
    friend iterator end(const B&);
};

struct D;
struct D_iter;

D_iter begin(const D&);
D_iter end(const D&);
```

Today, C++20 ranges implementations have to go to [quite some lengths](https://github.com/tcbrindle/NanoRange/blob/master/include/nanorange/detail/ranges/begin_end.hpp#L21) to handle the different ways that `begin()` and `end()` could be written (not to mention `size()`, `data()` and a host of others). With this form of opt-in [*uniform function call syntax*](https://en.wikipedia.org/wiki/Uniform_Function_Call_Syntax) provided by `using class`, almost all of that becomes unnecessary, and is handled by the language itself.


## Hang on, wasn't UFCS rejected by the committee? ##

UFCS for C++ has been proposed in a few forms in the last few years, including by Stroustrup himself. None of the proposals so far have been accepted -- Barry Revzin has a [nice summary](https://brevzin.github.io/c++/2019/04/13/ufcs-history/) of the situation on his blog. To be clear, I should state that here I am suggesting what Revzin calls "OR2" -- that is, all candidates (member and non-member, including those found by ADL) are considered according to the existing overload resolution rules -- but **only** when using non-member syntax ("CS1"), and **only** when a class's members have been made available in the current scope via a `using class` directive.

It's this last point that differentiates this idea from earlier proposals. As I understand it, the existing UFCS proposals have come unstuck due to the fear of unstable interfaces and "spooky action at a distance". Depending on which proposal we're talking about, either adding a new non-member overload (perhaps somewhere in a far-away header), or adding a new member function (perhaps somewhere in a far-away base class) could result in a different function begin called when innocent code is recompiled.

With this (proto-)proposal, we can use `using class` to explicitly say "yes, I know that might happen, *and I'm okay with it*". If you're not okay with it, don't use `using class`: you'll get the same results you do today.

## Addendum: using enum ##

At this point, it's worth mentioning [P1099](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2019/p1099r4.html) "Using enum" by Gašper Ažman and Jonathan Müller, which is looking likely to be part of C++20. Under their proposal, you'll be able to use a `using enum` directive to bring the names of enumeration members into the current scope. To take the motivating example from their paper:

```cpp
enum class rgba_color_channel { red, green, blue, alpha };

std::string_view to_string(rgba_color_channel channel) {
  switch (my_channel) {
    using enum rgba_color_channel;
    case red:   return "red";
    case green: return "green";
    case blue:  return "blue";
    case alpha: return "alpha";
  }
}
```

The "proposal" presented here is entirely compatible with `using enum`, and in fact matches its semantics very nicely. Just as `using enum` makes members of an enumeration available in the current scope, so `using class` makes members of a class available in the current scope.

So, `using class` would give us better consistency with namespaces (as envisioned by Stroustrup a quarter of a century ago), would give consistency with `using enum` in C++20, *and* solve the UFCS problem. Take that, Vasa!

## Quick questions ##

Here are some off-the-top-of-my-head answers to some off-the-top-of-my-head questions about how a hypothetical `using class` might work:

 * What's the difference between `using class` and `using struct`?

   Nothing whatsoever. Both should be valid, and they should do exactly the same thing, regardless of which *class-key* was used when defining the class. Yes, it's kind of pointless to allow both, but picking one or the other seems like it would be more inconsistent, given they're almost interchangeable everywhere else in the language.

 * What about `using union`?

   A union is a kind of class in C++, so it makes sense that you should be able to say `using class U` where `U` is a union (subject to the normal UB rules regarding reading from an inactive member).

   Should we additionally allow `using union U` if and only if `U` is a union? I can't see that it would be all that useful, but then again, "why not?"

 * Can I use `using class` with a forward declaration?

   No. The type must be complete at the point the directive is seen (or at instantiation time for a dependent type in a template).

 * What about private/protected members?

   This is a tricky one. My initial feeling was that only accessible members should be made available via `using class`: otherwise, we risk having a viable, intended non-member function out-matched by an inaccessible private member, which seems like it would be a pain. It arguably also leaks implementation details of the class into the current scope.

   On the other hand, accessibility testing happens after overload resolution everywhere else in the language (at least as far as I'm aware), so it would probably make more sense not to change that.

 * You've only mentioned member functions. What about data members?

   My view is that yes, data members should be made available in the current scope as well, so that we can invoke data members that are callable (e.g. `std::function`). We could possibly even go further and say that `memvar(x)` is re-written as `x.memvar`, which is (roughly) what `std::invoke` does with PMDs.

   Remember that a local variable in a scope will hide a name that's merely made available via a *using-directive*, so this wouldn't prevent you from naming your local variables whatever you like.

 * What about member types?

   Yep, those too, for consistency with `using namespace`.

 * What if I say `using class X; using class Y;` in the same scope?

   Then names from both classes are available in the current scope, just like if you said `using namespace N1; using namespace N2;`. If the two classes have members with the same name, then overload resolution is performed (in the case of a function call), or it's ambiguous, and you have to explicitly say which one you mean.

 * Can a `using class` directive appear at class scope?

   If allowed, this would have the effect of making all of the (public?) names in class `B` available in class `D`. I think C++ already has a way of doing this, and adding a second one seems like it would be problematic.

 * Could `using class` be used to do "full UFCS" (member syntax calling non-member functions too)?

   There is a strong desire in some quarters for a UFCS solution that allows calling non-member functions via the member syntax ("CS2" in Revzin's terminology). This is akin to what D does, and would allow something similar (at least syntactically) to Rust traits, C# extension methods and Swift extensions. There's no doubt that the member syntax can often be much more readable, particularly when nesting/chaining many calls.

   So, could `using class` be extended to allow this? Perhaps, but I haven't thought about it too deeply, and I'd be hesitant to suggest it.

   In C++, it's already accepted that an unqualified non-member call can have a open set of candidates: all `using class` does is offer an opt-in way to extend that set yet further, within a particular scope. By contrast, the set of candidates for a member function call is strictly limited. I think it might be a hard sell to change that.

   That said, if this idea proves popular, and someone comes up with a "full UFCS" implementation, and shows that it doesn't break any code and allows some nice new patterns, then you never know. Stranger things have happened.

 * `using namespace` is the devil's work, and you're suggesting making things worse!

   It's true that `using namespace` -- and in particular `using namespace std` -- is often seen an an anti-pattern. But remember that (a) most classes have fewer members than most namespaces, and (b) classes are closed, whereas namespaces are open. This means that (as I see it) there is less risk of unintended consquences with `using class` than `using namespace`. I also think that the recommendation would be that `using class` should normally only be used within function (template) implementations (although banning it outright at namespace scope feels a bit heavy-handed).


## In summary ##

In summary, I read a very old paper, saw something I liked the idea of and had a thought for how it could be used to solve a real problem in C++ today.

This post really contains two proposals in one. Firstly, we re-introduce the idea of allowing a class name in a *using-directive*, as first suggested by Stroustrup 26 years ago (albeit with a slightly different syntax). Secondly, we allow non-static member function names to participate in overload resolution, if made available via `using class`.

At this point, this is all very much just an idea, and perhaps a stupid one. It touches on both the third and fourth rails of the C++ standard -- name lookup and overload resolution -- which are both fearsomely complicated as it is, and which no sane person wants to mess with. I have no doubt there are tricky (and perhaps even obvious) problems I'm overlooking.

Nonetheless, I do like the idea and I think it (perhaps) has merit, and so I thought it was worth writing up as a blog post to get some feedback. Let me know if you think I might be on to something, or if I've completely lost the plot.

And who knows, if enough people like it then maybe I'll write this up more formally...

