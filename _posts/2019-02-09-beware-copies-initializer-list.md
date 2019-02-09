---
layout: post
title: Beware of copies with std::initializer_list!
author: Tristan Brindle
permalink: /posts/beware-copies-initializer-list
comments: false
tags: C++
---

Pop quiz: what does the following C++ program do?

```cpp
#include <memory>
#include <string>
#include <vector>

using namespace std;

int main()
{
    vector<unique_ptr<string>> vec{
        make_unique<string>("foo"),
        make_unique<string>("bar"),
        make_unique<string>("baz")
    };
}
```

This is a trick question of course: the answer is that the above program will [fail to compile](https://godbolt.org/z/HAyZK9). If this surprises you then you're not alone. When I first wrote something like the above, I expected it to work just fine.

If you want to know why this fails and how we can work around it, then read on.

## What's going on? ##

Looking at the error message, we can see that the compiler is complaining that it’s trying to use `std::unique_ptr`’s copy constructor, which is of course deleted.

Which is odd. The calls to `std::make_unique()` are returning rvalues, so they should simply be moved into place in the vector. Perhaps we need to explicitly use `std::move()`?

```cpp
int main()
{
    vector<unique_ptr<string>> vec{
        move(make_unique<string>("foo")),
        move(make_unique<string>("bar")),
        move(make_unique<string>("baz"))
    };
}
```

Nope, that doesn’t work, and neither should we expect it to -- explicitly `move()`-ing something that is already an prvalue won't do anything good. (In fact, Clang with `-Wall` will warn about a pessemistic move in this case, which is rather nice.) What’s going on here? *Surely* vector’s `initializer_list` constructor doesn’t always copy, does it?

Let’s have a closer look.

## Counting instances ##

To investigate further, we'll use a wrapper class which counts how many times a type is constructed, copied, moved and destructed, and prints these stats after `main()` returns:

```cpp
template <typename T>
struct instance_counter {

    instance_counter() noexcept { ++icounter.num_construct; }
    instance_counter(const instance_counter&) noexcept { ++icounter.num_copy; }
    instance_counter(instance_counter&&) noexcept { ++icounter.num_move; }
    // Simulate both copy-assign and move-assign
    instance_counter& operator=(instance_counter) noexcept
    {
        return *this;
    }
    ~instance_counter() { ++icounter.num_destruct; }

private:
    static struct counter {
        int num_construct = 0;
        int num_copy = 0;
        int num_move = 0;
        int num_destruct = 0;

        ~counter()
        {
            std::printf("%i direct constructions\n", num_construct);
            std::printf("%i copies\n", num_copy);
            std::printf("%i moves\n", num_move);
            const int total_construct = num_construct + num_copy + num_move;
            std::printf("%i total constructions\n", total_construct);
            std::printf("%i destructions ", num_destruct);
            if (total_construct == num_destruct) {
                std::printf("(no leaks)\n");
            } else {
                std::printf("WARNING: potential leak");
            }
        }
    } icounter;
};

template <typename T>
typename instance_counter<T>::counter instance_counter<T>::icounter{};

template <typename T>
struct counted : T, private instance_counter<T>
{
    using T::T;
};
```

Let’s now try to construct a vector of `counted<std::string>` using an initializer list, and see what happens:

```cpp
int main()
{
    vector<counted<string>> vec{
        "foo", "bar", "baz"
    };
}
```

This reports

```
3 direct constructions
3 copies
0 moves
6 total constructions
6 destructions (no leaks)
```

This reveals the sad truth: things are just as bad as we feared. We are indeed constructing three temporary `counted<string>` instances, and then copying (not moving!) them into place.

A quick search turns up the explanation, nicely given in [this StackOverflow answer](http://stackoverflow.com/a/24793143). As stated in the answer, an initializer list behaves like an array of `const T` – and since the elements are `const`, we cannot move them out of the array.

Well, that’s disappointing. Can we work around this somehow?

## Fill your vector like it's 1998 ##

We want efficient vector construction, but it seems like `initializer_list` is off the table. Instead, let's do things the old-fashioned way, by first constructing the vector and then calling `push_back()` for each element:

```cpp
int main()
{
    std::vector<counted<string>> vec;
    vec.push_back("foo");
    vec.push_back("bar");
    vec.push_back("baz");
}
```

This time (on my system) we get

```
3 direct constructions
0 copies
6 moves
9 total constructions
9 destructions (no leaks)
```

This is potentially better -- we're not copying the strings any more -- but not ideal. We're getting six moves, when really only three should be required. Why is that?

The reason is that as we're adding elements to the vector one-by-one, it needs to resize itself to accommodate the new elements. As it does this, it will move the already-constructed elements into the newly-allocated block of memory (but *only* if the value type has a `noexcept` move constructor -- otherwise it will copy them into place). But since we know in advance how many elements we are going to add to the vector, we can avoid this reallocation by calling `vector::reserve()` up front:

```cpp
int main()
{
    std::vector<counted<string>> vec;
    vec.reserve(3);
    vec.push_back("foo");
    vec.push_back("bar");
    vec.push_back("baz");
}
```

This time we get

```
3 direct constructions
0 copies
3 moves
6 total constructions
6 destructions (no leaks)
```

Just three constructions followed by three moves, which is (nearly) ideal.

## This sounds a bit iffy ##

Using the above method is more efficient in terms of copies than using an initializer list, but one downside is that we cannot make our vector `const`, since we need to modify it after its initial construction. Fortunately, we can get around that by using an [immediately-invoked function expression](https://en.wikipedia.org/wiki/Immediately-invoked_function_expression) ("IIFE"), in the form of a lambda:

```cpp
int main()
{
    const auto vec = [] {
        std::vector<counted<string>> v;
        v.reserve(3);
        v.push_back("foo");
        v.push_back("bar");
        v.push_back("baz");
        return v;
    }();
}
```

Using immediately-invoked lambdas like this to allow complex initialisation of `const` variables is a really nice pattern that doesn't seem to be used all that much in C++.

## Fewer lambdas, more templates ##

So we've now got a way of (almost) optimally constructing a vector from a bunch of elements, but having to write lambdas all the time will get tiresome pretty quickly. Can we somehow abstract this into a reusable function? The answer is yes, using a variadic function template:

```cpp
template <typename T, typename... Args>
std::vector<T> make_vector(Args&&... args)
{
    std::vector<T> vec;
    vec.reserve(sizeof...(Args));
    (vec.push_back(std::forward<Args>(args)), ...);
    return vec;
}

int main()
{
    const auto vec = make_vector<counted<string>>(
        "foo", "bar", "baz"
    );
}
```

The `push_back()` call here is in a [fold expression](http://en.cppreference.com/w/cpp/language/fold), which is a new feature in C++17. For C++11 and 14, we can emulate the same thing by taking advantage of the rules around pack expansions in initializer lists. If you're squeamish, look away now:

```cpp
template <typename T, typename... Args>
std::vector<T> make_vector(Args&&... args)
{
    std::vector<T> vec;
    vec.reserve(sizeof...(Args));
    using arr_t = int[];
    (void) arr_t{0, (vec.push_back(std::forward<Args>(args)), 0)...};
    return vec;
}
```

If you're interested in how this monstrosity works, check out [this StackOverflow question](https://stackoverflow.com/questions/17339789/how-to-call-a-function-on-all-variadic-template-args?lq=1). The gist of it is that we are creating a temporary, unnamed `int` array and filling it with zeros, and calling `push_back()` for each element as a side-effect. The cast to `void` is just to avoid a compiler warning about an unused variable.

Running the above code, we can see that we get the same results as before: three constructions followed by three moves.

```
3 direct constructions
0 copies
3 moves
6 total constructions
6 destructions (no leaks)
```

## Removing the moves ##

The next question is, do we really need the three moves? Can't we just construct the strings directly inside the vector? Fortunately we can, by changing our `push_back()` calls to `emplace_back()`:

```cpp
template <typename T, typename... Args>
std::vector<T> make_vector(Args&&... args)
{
    std::vector<T> vec;
    vec.reserve(sizeof...(Args));
    using arr_t = int[];
    (void) arr_t{0, (vec.emplace_back(std::forward<Args>(args)), 0)...};
    return vec;
}

int main()
{
    const auto vec = make_vector<counted<string>>(
        "foo", "bar", "baz"
    );
}
```

This time, we get

```
3 direct constructions
0 copies
0 moves
3 total constructions
3 destructions (no leaks)
```

which is perfect, as far as the number of constructor invocations goes. However, there is a (potential) downside to this: `emplace_back()` will happily call `explicit` constructors, whereas `push_back()` (which relies on an implicit conversion to the value type first) will not. This doesn't make any difference in this case (since `std::string` has an implicit constructor from `const char*`), but it would be noticable with, for example, `string_view`:

```cpp
int main()
{
    using namespace std::string_view_literals;

    const auto vec = make_vector<counted<string>>(
        "foo"sv, "bar"sv, "baz"sv
    );
}
```

The above will work if we use `emplace_back()`, but will fail with `push_back()` because the `std::string(std::string_view)` constuctor is `explicit`. Since the whole point of explicit constructors is that their use should be, well, *explicit*, my preference would normally be to avoid hiding them and just use `push_back()`.

But we've got this far, so we may as well aim for perfection: we can guard against using explicit constructors by inserting a `static_assert` inside `make_vector()`, along the lines of

```cpp
static_assert((std::is_convertible_v<Args, T> && ...),
              "All arguments to make_vector() must be implicitly convertible to T");
```

and then use `emplace_back()` for maximal efficiency. An alternative would be to use `std::enable_if` to disable the function if each argument type is not implicitly convertible to the vector's value type.


## Finishing touches ##

We're getting there now, but there's another improvement we can make. C++17 introduces [class template argument deduction](http://en.cppreference.com/w/cpp/language/class_template_argument_deduction), which allows you to say

```cpp
auto v = std::vector{1, 2, 3};
```

which will deduce the vector's value type as `int`. We can do something similar with our `make_vector()` by introducing a tiny bit of metaprogramming:

```cpp
namespace detail {

template <typename T, typename...>
struct vec_type_helper {
    using type = T;
};

template <typename... Args>
struct vec_type_helper<void, Args...> {
    using type = typename std::common_type<Args...>::type;
};

template <typename T, typename... Args>
using vec_type_helper_t = typename vec_type_helper<T, Args...>::type;

}
```

Now, `detail::vec_type_helper_t<T, Args...>` will be `T`, except if that is `void`, in which case it will be the common type of the arguments. We can then modify our `make_vector()` function to use this, defaulting the first template parameter:

```cpp
template <typename T = void, typename... Args,
          typename V = detail::vec_type_helper_t<T, Args...>>
std::vector<V> make_vector(Args&&... args)
{
    std::vector<V> vec;
    vec.reserve(sizeof...(Args));
    using arr_t = int[];
    (void) arr_t{0, (vec.push_back(std::forward<Args>(args)), 0)...};
    return vec;
}
```

Now in simple cases we can say

```cpp
const auto vec = make_vector(1, 2, 3);
```

and have the value type correctly deduced for us. As an added bonus, this works all the way back to C++11, not just C++17. And of course, we can still specify the value type explicitly too:

```cpp
const auto vec = make_vector<string>("foo", "bar", "baz");
```

What's more, if we don't supply the value type then this will fail to compile if the arguments have no common type (or a common type which is ambiguous). So, no surprises.

## Conclusion ##

A complete implementation of `make_vector()` can be found in [this gist](https://gist.github.com/tcbrindle/e55bea06d4e3ac1f603f9989d44dceb9). It backwards compatible to C++11, minimises the number of moves and copies, allows deduction of the value type from its arguments, and works with move-only types like `std::unique_ptr`. It's superior to `vector`'s `initializer_list` constructor whenever the value type is not trivially copyable, and exactly equivalent otherwise.

Of course, the real lesson here is that a function like `make_vector()` shouldn't be needed at all. **This is something the language should handle properly**. It's especially sad given that if we use a raw dynamic array rather than a `vector`, things work just fine:

```cpp
int main()
{
    auto vec = std::unique_ptr<counted<string>[]>{
        new counted<string>[]{"foo", "bar", "baz"}
    };
}
```

```
3 direct constructions
0 copies
0 moves
3 total constructions
3 destructions (no leaks)
```

All in all, it's hard to escape the conclusion that C++11's "uniform initialisation" (or "unicorn initialisation" as some call it) was a mistake -- for this and other reasons. Sadly it's one that may be too late to fix now.
