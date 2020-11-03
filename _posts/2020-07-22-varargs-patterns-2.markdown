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
    
&#x2058; [Generic integer sequences](#generic-integer-sequence)

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
    //using Base = Tuple<T>;
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

Zipping operations refer to the capability of composing a single sequence out
of multiple sequences where each i-th element of the new sequence contains a tuple
of elements read from position i in the input sequences, or when iterators
are used, the zip iterator contains a tuple of iterators instead. 

Ideally we would like to be able to write code like:

```cpp
vector<string> words = {"first", "second", "third"};
vector<int> indices = {1, 2, 3};
vector<tuple<int, string>> indexedWords(Zipper(begin(indices), begin(words)),
                                        Zipper(end(indices), end(words)));
for(auto t: indexedWords) cout << get<0>(t) << ": " << get<1>(t) << endl;
```

ouput:

```term
1: first
2: second
3: third
```

What follows is an minimal implementation of a zip (forward) iterator useful 
to lazily invoke other iterators and/or build new zipped sequences.
Fold expression are used to simplify code by avoiding recursive template
expansion.

The minimal functionality to implement is:

1. increment operator, to advance to next element
2. dereference operator, to retrieve value
3. equality operator, to allow `begin()` - `end()` iteration

```cpp
#include <iostream>
#include <string>
#include <tuple>
#include <utility>
#include <vector>
#include <iterator>

using namespace std;

template <typename... ArgsT>
class Zipper {
  private:
    using Indices =
        std::make_index_sequence<tuple_size<tuple<ArgsT...>>::value>;
    using ValueTuple = tuple<typename ArgsT::value_type&...>;
    tuple<ArgsT...> its_;
  public:
    using difference_type = ptrdiff_t;
    using value_type = tuple<ArgsT...>;
    using pointer = value_type*;
    using reference = value_type&;
    using iterator_category = forward_iterator_tag;

  public:
    Zipper() = delete;
    Zipper(ArgsT... i) : its_(i...) {}
    Zipper(const Zipper&) = default;
    Zipper(Zipper&&) = default;
    Zipper& operator++() {
        IncIterators(Indices{});
        return *this;
    }
    ValueTuple operator*() const { return Values(Indices{}); }
    bool operator==(const Zipper& other) const {
        return Equal(other, Indices{});
    }
    bool operator!=(const Zipper& other) const { return !operator==(other); }

   private:
    template <size_t... I>
    void IncIterators(const index_sequence<I...>&) {
        (get<I>(its_)++, ...);
    }
    template <size_t... I>
    ValueTuple Values(const index_sequence<I...>&) const {
        return ValueTuple(*get<I>(its_)...);
    }
    template <size_t... I>
    bool Equal(const Zipper& other, const index_sequence<I...>&) const {
        return (... && (get<I>(its_) == get<I>(other.its_)));
    }
};

template <typename... ArgsT>
pair<Zipper<typename ArgsT::iterator...>,
     Zipper<typename ArgsT::iterator...>> constexpr Zip(ArgsT&... seqs) {
    return {Zipper<typename ArgsT::iterator...>(begin(seqs)...),
            Zipper<typename ArgsT::iterator...>(end(seqs)...)};
}

template <typename... ArgsT>
pair<Zipper<typename ArgsT::const_iterator...>,
     Zipper<typename ArgsT::
                const_iterator...>> constexpr Zip(const ArgsT&... seqs) {
    return {Zipper<typename ArgsT::iterator...>(begin(seqs)...),
            Zipper<typename ArgsT::iterator...>(end(seqs)...)};
}

template <typename F, typename S>
F constexpr begin(pair<F, S> p) {return p.first;}

template <typename F, typename S>
F constexpr end(pair<F, S> p) {return p.second;}

int main(int argc, char const* argv[]) {
    vector<int> ints{1, 2, 3};
    vector<string> strings{"1", "2", "3"};
    Zipper zzip(begin(ints), begin(strings));
    Zipper zzend(end(ints), end(strings));
    do {
        auto [i, s] = *zzip;
        cout << "{" << i << ", " << s << "} ";
    } while (++zzip != zzend);

    auto z = Zip(ints, strings);

    do {
        auto [i, s] = *z.first;
        cout << "{" << i << ", " << s << "} ";
    } while (++z.first != z.second);
    cout << endl;

    auto it = Zip(ints, strings);

    for(auto [i, x]: it) {
        cout << i << " " << x << endl;
    }

    vector<string> words = {"first", "second", "third"};
    vector<int> indices = {1, 2, 3};
    vector<tuple<int, string>> indexedWords(Zipper(begin(indices), begin(words)),
                                            Zipper(end(indices), end(words)));
    for(auto t: indexedWords) cout << get<0>(t) << ": " << get<1>(t) << endl;

    return 0;
}
```

<div style='text-align: right' markdown='1'>
  <a href="https://godbolt.org/z/cx57GT">run</a>
</div>