---
layout: post
title: Variadic template patterns - part three
categories: [C++, C++11, C++14, C++17, variadic templates, templates]
permalink: /varargs-patterns-3/
---

This is the last of three articles about variadic templates in which we explore
different parameter pack expansion patterns.

Here we expand on the previous article and implement generic integer sequences
with custom generators: instead of simply increasing the index by one at
every expansion step a new index is generated as a function of the previous
*n* indices.

The key difference is that instead of simply expanding the parameter pack by
adding *1* to the current parameter pack size, we use a generator function:

$$index_{n} \larr f(index_0,\ldots,index_{n-1})$$

I.e.

From

```cpp
template <int Size>
struct GenerateIndexSequence {...}
```

To

```cpp
template <int Size, template <int... I> class GenT, int... Start>
struct GenerateIndexSequence {...}
```

`GenT` is the compile-time generator.
`Start...` is required as the seed sequence to be passed to the generator at
the first expansion step.

The code below implements the following generator functions:

* *Inc*: increment by one
* *IncStep<D>*: increment by *D*
* *Fibonacci*: the Fibonacci series
  
Storing floating point numbers in compile time integer sequences allows to
implement generic compile-time series generators of any kind (e.g. Taylor polynomial)
and will be the subject of the next blog article, but code is already available
[here](https://github.com/ugovaretto-codex/cpp/blob/master/float_constexpr.cpp).


```cpp
#include <cassert>
#include <iostream>

//------------------------------------------------------------------------------
template <int S, template <int... I> class GenT, int... N>
struct Sequence : Sequence<S - 1, GenT, N..., GenT<N...>::value> {};

template <int... N>
struct Idx {};

template <template <int... I> class GenT, int... N>
struct Sequence<0, GenT, N...> {
    using Index = Idx<N...>;
};

template <int Size, template <int... I> class GenT, int... Start>
struct GenerateIndexSequence {
    static_assert(Size > 0, "Size <= 0");
    using Type = typename Sequence<Size - 1, GenT, Start...>::Index;
};

//------------------------------------------------------------------------------
template <int H, int... I>
struct Last : Last<I...> {};

template <int I>
struct Last<I> {
    enum : int { value = I };
};

template <int Pos, int H, int... T>
struct Nth {
    enum : int { value = Nth<Pos - 1, T...>::value };
};

template <int H, int... T>
struct Nth<0, H, T...> {
    enum : int { value = H };
};

//------------------------------------------------------------------------------
template <int... I>
struct Inc {  // increment last index
    enum : int { value = Last<I...>::value + 1 };
};

template <int D>
struct IncStep { // increment last index by step D
    template <int...I>
    struct Type {
        enum : int { value = Last<I...>::value + D};
    };
};

template <int... I>
struct Fibonacci {  // generate next number in Fibonacci sequence
    static_assert(sizeof...(I) > 1,
                  "Fibonacci requires size of starting sequence > 1");
    enum : int {
        value = Nth<sizeof...(I) - 2, I...>::value +
                Nth<sizeof...(I) - 1, I...>::value
    };
};


template <int Size, int Start = 0>
using IndexSequence =
    typename GenerateIndexSequence<Size, Inc, Start>::Type;

template <int Size>
using FibonacciSequence =
    typename GenerateIndexSequence<Size, Fibonacci, 1, 1>::Type;

#if __cplusplus >= 201703L
template <int... I>
void PrintIndices(Idx<I...>) {
    (..., (std::cout << I << " "));
    std::cout << std::endl;
}
#else

template <typename T>
void PrintIndicesHelper(T) {}

template <int H, int... Tail>
void PrintIndicesHelper(Idx<H, Tail...>) {
    std::cout << H << " ";
    PrintIndicesHelper(Idx<Tail...>());
}

template <int... I>
void PrintIndices(Idx<I...>) {
    PrintIndicesHelper(Idx<I...>());
    std::cout << std::endl;
}
#endif

int main(int argc, char const *argv[]) {
    std::cout << std::endl;
    const auto FS = FibonacciSequence<10>();
    PrintIndices(FS);
    using IndexStepSequence =
        typename GenerateIndexSequence<10, IncStep<3>::Type, 1>::Type;
    const auto ISS = IndexStepSequence();
    PrintIndices(ISS);
    return 0;
}
```

<div style='text-align: right' markdown='1'>
  <a href="https://godbolt.org/z/3Me6xd">run</a>
</div>
