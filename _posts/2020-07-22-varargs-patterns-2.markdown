---
layout: post
title: Variadic template patterns - part two
categories: [C++, C++11, C++14, C++17, variadic templates, templates, concepts]
permalink: /varargs-patterns-2/
---

In the [first part]({{site.baseurl}}/../2020-07-02-varargs-patterns-1) we looked
at the basics of variadic templates and showed a few usage patterns, in this
second part we will expand the use cases to cover more advanced patterns
including support for concepts, introduced in C++20.

### Content
    
&#x2058; [Tuple](#tuple)

&#x2058; [Zip iterator](#zip-iterator)
    
&#x2058; [Integer index sequence](#integer-sequence)
    
&#x2058; [Generic integer sequences](#fold-expressions)

*Under construction*

## Tuple

`tuple` has been part of the C++ standard since the C++11 release. Before
C++11, developers could rely on other widely available implementations such as
`boost::tuple`, which however, because of the lack of variadic templates and
other constructs turns out to be a pretty convoluted implementation with
a pre-set limit on the number of supported arguments, newer implementations
of `boost::tuple` do have full support for the latest C++ standard features.

Writing a full C++98 tuple implementation in-line in this blog article is not
viable but you can check for yourself one of my implementations [here](https://github.com/ugovaretto-codex/cpp/blob/master/tuple98-virtual-inheritance.cpp).

This is a blog about variadic templates but other features such as fold
expressions, to trigger iteration over a variable number of parameters, and concepts, to allow for partial template specialization of functions are instrumental in reducing the complexity as
shown in the code below.

### 1. Explicit element selection

Using `Nth<>` construct to extract the n-th element and declare return type
of `get` funcitons; concepts used only to specialise `index == 0` case.

```cpp
#include <iostream>

using namespace std;

template <typename T, typename...RestT>
struct Tuple : Tuple<RestT...> {
    using Base = Tuple<RestT...>;
    T value;
    constexpr Tuple(const T& v, const RestT&...r) : Base(r...), value(v) {}
};

template <typename T>
struct Tuple<T> {
    using Base = Tuple<T>;
    T value;
    constexpr Tuple(const T& v) : value(v) {}
};
```

```cpp
template <size_t I, typename...T>
struct Nth {
    static_assert(I < sizeof...(T));
};

template<typename T, typename...Rest>
struct Nth<0, T, Rest...> {
    using Type = T;
};
template <size_t I, typename T, typename...RestT>
struct Nth<I, T, RestT...> {
    using Type = typename Nth<I - 1, RestT...>::Type;
};
```

Element extraction functions, `constexpr` allows `get` to be used at compile time
inside template declarations. `consteval` not viable in this case because
it would prevent use at run-time.

```cpp
template<size_t I, typename T, typename...RestT>
constexpr const typename Nth<I, T, RestT...>::Type&  get(const Tuple<T, RestT...>& t)
requires (I == 0) {
    return t.value;
}

template<size_t I, typename T, typename...RestT>
constexpr const typename Nth<I, T, RestT...>::Type&  get(const Tuple<T, RestT...>& t) {
    using Base = typename Tuple<T, RestT...>::Base;
    return get<I-1>((const Base&) t);
}

template<size_t I, typename T, typename...RestT>
constexpr typename Nth<I, T, RestT...>::Type&  get(Tuple<T, RestT...>& t)
requires (I == 0) {
    return t.value;
}

template<size_t I, typename T, typename...RestT>
constexpr typename Nth<I, T, RestT...>::Type&  get(Tuple<T, RestT...>& t) {
    using Base = typename Tuple<T, RestT...>::Base;
    return get<I-1>((Base&) t);
}

template <int I>
struct Int {
    static const int i = I;
};

int main(int, char**) {
    constexpr Tuple<int, float, char> t = {1, 3.2f, 'c'};
    cout << t.value << endl;
    const auto& a = ((typename Tuple<int, float, char>::Base) t);
    cout << a.value << endl;
    constexpr auto i = get<1>(t);
    cout << i << endl;
    Int<get<2>(t)> ii; // constexpr case
    cout << char(ii.i) << endl;
    return 0;
}
```
<div style='text-align: right' markdown='1'>
 <a href="https://godbolt.org/z/hM6qbx">run</a>
</div>

### 2. Element selection through concepts

Instead of an `Nth` struct, concepts are used directly with the element
extraction functions to select elements; `auto` used as the return type.

```cpp
#include <iostream>

using namespace std;

template <typename T, typename...RestT>
struct Tuple : Tuple<RestT...> {
    using Base = Tuple<RestT...>;
    T value;
    constexpr Tuple(const T& v, const RestT&...r) : Base(r...), value(v) {}
};

template <typename T>
struct Tuple<T> {
    using Base = Tuple<T>;
    T value;
    constexpr Tuple(const T& v) : value(v) {}
};

template<size_t I, typename T, typename...RestT>
constexpr const auto&  get(const Tuple<T, RestT...>& t)
requires (I == 0) {
    return t.value;
}

template<size_t I, typename T, typename...RestT>
constexpr const auto&  get(const Tuple<T, RestT...>& t)
requires (I > 0 && I < sizeof...(RestT) + 1) {
    using Base = typename Tuple<T, RestT...>::Base;
    return get<I-1>((const Base&) t);
}

template<size_t I, typename T, typename...RestT>
constexpr auto&  get(Tuple<T, RestT...>& t)
requires (I == 0) {
    return t.value;
}

template<size_t I, typename T, typename...RestT>
constexpr auto&  get(Tuple<T, RestT...>& t) 
requires (I > 0 && I < sizeof...(RestT) + 1) {
    using Base = typename Tuple<T, RestT...>::Base;
    return get<I-1>((Base&) t);
}

template <int I>
struct Int {
    static const int i = I;
};

int main(int, char**) {
    constexpr Tuple<int, float, char> t = {1, 3.2f, 'c'};
    cout << t.value << endl;
    const auto& a = ((typename Tuple<int, float, char>::Base) t);
    cout << a.value << endl;
    constexpr auto i = get<1>(t);
    cout << i << endl;
    Int<get<2>(t)> ii;
    cout << char(ii.i) << endl;
    Tuple<char, int> ci = {'a', 3};
    get<1>(ci) = 5;
    cout << get<1>(ci) << endl;
    return 0;
}
```

<div style='text-align: right' markdown='1'>
  <a href="https://godbolt.org/z/1Mjc4x">run</a>
</div>

## Zip iterator

