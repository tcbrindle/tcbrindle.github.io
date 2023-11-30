---
layout: post
title: Compile-time Bounds Checking in Flux
author: Tristan Brindle
permalink: /posts/compile-time-bounds-checking-in-flux
comments: false
tags: C++
---

When using bounds checked interfaces, there are often situations during optimisation where the compiler is able to statically prove that a bounds check will always pass, allowing it to omit the checking code entirely. The simplest example would be something like so:

```cpp
int get_first_elem(const std::array<int, 5>& arr) {
    return arr.at(0);
}
```

<small>[Compiler Explorer](https://godbolt.org/z/qfzadq51o)</small>

When optimising this function, the compiler knows that the array `arr` always has five elements, and it knows that we're asking for the element at index zero. Therefore, even though we're using the bounds-checked `at()` function, the optimiser has enough information to remove the check and unconditionally return the first element of the array.

This is a great optimisation, particularly for libraries like [Flux](https://github.com/tcbrindle/flux) which perform bounds checking universally by default. But if the compiler is able to detect when a bounds check will always *pass*, can it also do the converse -- that is, detect when a bounds check will always *fail*?

The answer is that yes, of course it can! Let's say we change the above function to `get_last_elem()`, but we make an off-by-one error in the implementation and ask for index 5 by mistake:

```cpp
int get_last_elem(const std::array<int, 5>& arr) {
    return arr.at(5); // oops, off-by-one
}
```

<small>[Compiler Explorer](https://godbolt.org/z/4xY3Pvs35)</small>

If we look at the generated code, we can see that the compiler is now unconditionally throwing an exception from this function, as it knows that the bounds check is certain to fail.

Generating code that fails efficiently is all very well, but what would be even better is if we could somehow **turn this into a compile-time error** instead: that is, to have the compiler report an error to the user if it detects a bounds violation during compilation.

The great news is that **Flux can now do exactly this**! Let's change our erroneous `get_last_elem()` to use the `flux::read_at()` function instead of `std::array::at()`:

```cpp
int get_last_elem(const std::array<int, 5>& arr) {
    return flux::read_at(arr, 5); // oops, off-by-one
}
```

Now, with GCC or Clang at `-O1` or higher, we'll get a compiler error message instead:

```
error: call to 'flux::static_bounds_check_failed' declared with attribute error: out-of-bounds sequence access detected
  304 |                     static_bounds_check_failed();
```

<small>[Compiler Explorer](https://flux.godbolt.org/z/P7TfnMYsv)</small>

Of course, this is just a simple example, but GCC (in particular) is able to perform the equivalent "lifting" of errors in some surprisingly complicated sitations. The higher your optimisation settings, the more likely the compiler is going to be able to turn your run-time bounds check into a compile-time one.

## How does it work? ##

Note that this is nothing whatsoever to do with `constexpr` evaluation: in that case we'll always get a compile error if we attempt to read outside the bounds of an array, by the rules of the abstract machine. Rather, this is for code that is (notionally) run time, but where the optimiser has enough information to "see through" it using a combination of function inlining and constant propagation.

This trick works by making use of the GCC compiler extension [`__builtin_constant_p`](https://gcc.gnu.org/onlinedocs/gcc/Other-Builtins.html#index-_005f_005fbuiltin_005fconstant_005fp) (also available in Clang). Given an expression, this function returns whether the compiler knows the expression to be a constant under the current optimisation settings. Using this, we can write a bounds-checking function that (if possible) will perform a compile-time bounds check:

```cpp
template <typename Index>
void bounds_check(Index idx, Index limit)
{
    if (__builtin_constant_p(idx) && __builtin_constant_p(limit)) {
        if (idx < Index{0} || idx >= limit) {
            /* ...report error at compile time... */
        }
    } else {
        /* ...perform normal run-time bounds check... */
    }
}
```

Now, if the compiler statically knows the values of `idx` and `limit`, and the bounds check fails, then we can raise an error at compile time.

But wait a second -- how do we deliberately cause a compile error here? We can't use `static_assert` or anything similar because that would always fail unconditionally. Remember, we're not dealing with things that the *C++ language* considers to be compile-time constants -- things like template parameters or `constexpr` variables on which we could `static_assert` or `if constexpr`. Rather, these are things that the language considers to be run-time values; it just so happens that the optimiser has extra information about them.

The solution is to use another GCC extension, the [`error` attribute](https://gcc.gnu.org/onlinedocs/gcc/Common-Function-Attributes.html#index-error-function-attribute) (again, also available in Clang). If we declare a function with this attribute, like so:

```cpp
[[gnu::error("out-of-bounds sequence access detected")]]
void static_bounds_check_failed();
```

then compilation will fail if a call to this function doesn't eventually get optimised away. The nice thing is that (with GCC at least) we end up with a compiler backtrace which shows precisely where the original offending code is, even through a long chain of inlined functions.

## Limitations ##

In case it isn't clear, turning certain run-time errors into compile time errors like this relies on compiler optimisations, and extensions which allow us to ask the compiler what the optimiser "sees" and inject an error if the check fails.

If you're compiling without optimisation, or with an unsupported compiler, then you'll  get a program which fails an ordinary bounds check when you run it -- which is, of course, exactly what you asked for! Being able to detect certain bounds violations at compile time should be seen as a nice bonus, rather the something to rely on. (What was that, Mr Hyrum?)

If for some reason you don't want this to happen -- perhaps you're trying to test your error handling code, for example -- then you can disable it by defining the preprocessor symbol `FLUX_DISABLE_STATIC_BOUNDS_CHECKING` before `#include`-ing Flux.

## Thanks ##

A big thank you to [Matthias Kretz](https://mattkretz.github.io) for showing me this trick at CppCon and suggesting I use it in Flux.

If anyone knows how to perform the equivalent trick with MSVC please let me know!



