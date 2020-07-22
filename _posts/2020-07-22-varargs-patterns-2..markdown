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

```cpp
#include <iostream>
#include <cstdlib>

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

template<size_t I, typename T, typename...RestT>
constexpr const typename Nth<I, T, RestT...>::Type&  get(const Tuple<T, RestT...>& t)
requires (I == 0) {
    return t.value;
}

template<size_t I, typename T, typename...RestT>
constexpr const typename Nth<I, T, RestT...>::Type  get(const Tuple<T, RestT...>& t) {
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
    Int<get<2>(t)> ii;
    cout << char(ii.i) << endl;
    return 0;
}
```
<div style='text-align: right' markdown='1'>
 <a href="https://godbolt.org/z/e9o717">run</a>
</div>

```cpp
#include <iostream>
#include <cstdlib>

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

```cpp
#include <iostream>

using namespace std;

template <typename T, typename...RestT>
struct Tuple : Tuple<RestT...> {
    using Base = Tuple<RestT...>;
    static constexpr size_t NumElements = 1 + Base::NumElements; 
    T value;
    constexpr Tuple(const T& v, const RestT&...r) : Base(r...), value(v) {}
};

template <typename T>
struct Tuple<T> {
    using Base = Tuple<T>;
    static constexpr size_t NumElements = 1;
    T value;
    constexpr Tuple(const T& v) : value(v) {}
};

template <typename T>
concept IsTuple = requires {
    typename T::Base;
    T::NumElements;
};

template<size_t I, typename T>
constexpr const auto&  get(const T& t)
requires (I == 0) && IsTuple<T> {
    return t.value;
}

template<size_t I, typename T>
constexpr const auto&  get(const T& t)
requires (I > 0 && I < T::NumElements) && IsTuple<T> {
    using Base = typename T::Base;
    return get<I-1>((const Base&) t);
}

template<size_t I, typename T>
constexpr auto&  get(T& t)
requires (I == 0) && IsTuple<T> {
    return t.value;
}

template<size_t I, typename T>
constexpr auto&  get(T& t)
requires (I > 0 && I < T::NumElements) && IsTuple<T> {
    using Base = typename T::Base;
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
 <a href="https://godbolt.org/z/n57xnn">run</a>
</div>