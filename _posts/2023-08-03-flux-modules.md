---
layout: post
title: Experimenting with Modules in Flux
author: Tristan Brindle
permalink: /posts/flux-modules-experiments
comments: false
tags: C++
---

**TL;DR**: [Flux](https://github.com/tcbrindle/flux), my C++20 sequence-orientated programming library, now has experimental support for being used as a module. Provided you're using a recent compiler and don't mind a few rough edges, you can say `import flux` and get going.

The future is (almost) here!

## Trying it out ##

If you fancy giving it a try, you'll need a recent version of one of the three major compilers: I've got it working with Clang 16, GCC 13.1 and MSVC 17.6, but slightly older compilers might work too.

First, grab a fresh checkout of Flux [from Github](https://github.com/tcbrindle/flux). Then `cd` to the checkout directory and save the following as `modules_test.cpp`:

```cpp
import flux;

int main()
{
    constexpr int arr[] = {1, 2, 3, 4, 5};
    static_assert(flux::sum(arr) == 15);
}
```

### Clang 16 ###

Starting in the top-level directory of the Flux checkout, the first step is to compile the binary module interface (BMI) with:

```
> clang++ -std=c++20 -I include/ --precompile -x c++-module module/flux.cpp
```

(You could also provide `-DNDEBUG` here to compile the library without extra debug checks, or your preferred [Flux configuration flags](https://github.com/tcbrindle/flux/blob/main/include/flux/core/config.hpp) if you wish.)

This should generate a file called `flux.pcm`. Next, we need to generate an object file to link to:

```
> clang++ -c flux.pcm
```

which should output `flux.o`. Finally, we can compile our test program:

```
> clang++ -std=c++20 -fprebuilt-module-path=. flux.o modules_test.cpp -o modules_test
```

If this compiles and outputs the `modules-test` executable, then congratulations! Everything is working.

### GCC 13 ###

For GCC, we need to provide the `-fmodules-ts` flag to enable modules support: just selecting C++20 mode is not enough.

```
> g++-13 -std=c++20 -fmodules-ts -I include/ module/flux.cpp -c
```

This should generate both the BMI (in a folder called `gcm.cache/flux.gcm`) and the `flux.o` object file.

Next we can compile the test file with:

```
> g++-13 -std=c++20 -fmodules-ts flux.o modules_test.cpp -o modules_test
```

Note that needs to be done from the directory that contains the `gcm.cache` folder. Presumably it's possible to persuade GCC to look elsewhere for BMIs, but I wasn't able to find the right options.

### MSVC 17.6 ###

For MSVC, open up a Visual Studio developer prompt and `cd` to your Flux checkout. Then run:

```
> cl.exe /std:c++20 /EHsc /I include /c /interface /TP module\flux.cpp
```

This will generate the `flux.ifc` BMI and `flux.obj` object file.

We can then compile the test program with

```
> cl.exe /std:c++20 /EHsc flux.obj module_test.cpp
```

> Note that at the time of writing, MSVC doesn't seem to like the `flux::ref()` function when using Flux via an `import`. It works fine via `#include`, and when importing in Clang and GCC, so it might just be a compiler bug.
>
> As a workaround, you can say `flux::from(std::cref(seq))` instead of `flux::ref(seq)`, which is semantically equivalent (though quite ugly).

### CMake ###

If you're using Clang or MSVC and are feeling particularly brave, you can also try out modular Flux using CMake.

You'll need to add Flux as a subproject and configure it with `FLUX_BUILD_MODULE=On`, which will add an extra library target called `flux-mod`. You should then be able be able to use `target_link_libraries()` to link your own executable with `flux-mod`, which will take care of adding all the necessary build commands.

Currently this uses Victor Zverovich's [modules.cmake](https://github.com/vitaut/modules) rather than CMake's very new built-in module support, although that might change at some point as the built-in CMake support matures.

## Modularising the code ##

Flux is a C++20 library that uses lots of cutting-edge features, but the lack of compiler and build system support meant that I had to write it as a "traditional" single-header library rather than using modules.

Then a few days ago I was watching Daniel Ruoso's C++Now 2023 talk, ["The Challenges of Implementing C++ Header Units"](https://youtu.be/_LGR0U5Opdg). The whole talk is great (if a bit depressing), but I was particularly interested in a slide near the end, where Daniel presents a simple way for authors of existing (non-modular) libraries to produce a wrapper module:

![Screenshot from CppNow](/assets/images/ruoso_wrapper_module.png)

That... actually seemed pretty easy, so I thought I'd give it a go. I whipped up very quick test file:

```cpp
module;

#include <flux.hpp>

export module flux;

export namespace flux {
    // Just export a few names to see if this works
    using flux::sequence;
    using flux::filter;
    using flux::map;
    using flux::write_to;
}
```

Amazingly, once I'd figured out the correct Clang command-line incantations, it seemed to work -- which was very different to my experiences of trying modules up to this point. Flushed with success, I went through and added all of the other public names to the `export namespace` block: [about 300 in all](https://github.com/tcbrindle/flux/blob/8dbbeee126787ecb9d8d1cf5fc6e4a47d82dfbec/module/flux.cpp).

I now had a modular version of Flux that worked with Clang and MSVC. GCC didn't like it for some reason -- and to be honest, I didn't like it all that much either. Duplicating all the public names in two places seemed very inelegant, and it would be annoying to have to remember to update the module file for every public symbol added to the library.

Hunting for an alternative, I looked at the only major project I know of that currently provides a public module: Victor Zverovich's [{fmt}](https://github.com/fmtlib/fmt/blob/master/src/fmt.cc). Following this example, I went through an added a `FLUX_EXPORT` macro to all the public names in the library, which expands to `export` if `FLUX_MODULE_INTERFACE` is defined, and nothing otherwise.

I then re-wrote the module file as

```cpp
module;

#include <array>
#include <compare>
/* ... all our other stdlib includes ... */

export module flux;

#define FLUX_MODULE_INTERFACE
#include <flux.hpp>
```

If I understand correctly -- and that's quite a big "if", as I'm still new to modules! -- this attaches all of the names in the library with the `flux` module, which seems like... what we want? But by first `#include`-ing all of the standard library headers outside of the module purview, we avoid taking ownership of any of the stdlib stuff.

I tried out the new approach and found that not only did it work just as well with Clang and MSVC, but GCC was also much happier now.

## ...but is it fast? ##

The short answer: yes.

I was interested to see what, if any, speedup we'd get using Flux via a module versus traditional header. As a test, I used a small program which solves the [Sushi For Two](https://codeforces.com/contest/1138/problem/A) problem from Conor Hoekstra's [Top 10 Algorithms](https://github.com/codereport/top10) set (which Conor also talks about in his fantastic "New Algorithms in C++23" talk -- look out for it when it drops online!). Here it is:


```cpp
#include <algorithm>
#include <vector>

#ifdef USE_MODULES
import flux;
#else
#include <flux.hpp>
#endif

constexpr int sushi_for_two(std::vector<int> const& sushi)
{
    return 2 * flux::ref(sushi)
                .chunk_by(std::equal_to{})
                .map(flux::count)
                .pairwise_map(std::ranges::min)
                .max()
                .value();
};

static_assert(sushi_for_two({2, 2, 2, 1, 1, 2, 2}) == 4);
static_assert(sushi_for_two({1, 2, 1, 2, 1, 2}) == 2);
static_assert(sushi_for_two({2, 2, 1, 1, 1, 2, 2, 2, 2}) == 6);

int main()
{}
```

The `#include`s at the top are intended to replicate a somewhat realistic scenario -- most real-world TUs are going to use some standard library facilities, after all.

I then used Clang's `-ftime-trace` facility to look at the internal timings during compilation:

* *Without* modules, the `#include <flux.hpp>` line took around **570ms** to process on my laptop (of which around half was `#include`-ing other standard library headers, and the rest parsing the Flux code itself)

* *With* modules, the `import flux` step took **57ms** -- almost exactly **ten times faster**!

Of course, this is only a single, unscientific test with one compiler on one machine, but the numbers certainly look encouraging.

## Looking forward ##

All this playing around has finally -- three years after C++20 was finished! -- got me excited about modules. There's still a very long way to go yet in terms of seamless compiler and build system support, but things seem to be heading in the right direction.

Flux will remain a header-based library for the foreseeable future, but I'll keep trying to improve the modules story. And who knows, maybe by the time C++26 rolls around we'll finally be able to (mostly) consign `#include` to the history books.

