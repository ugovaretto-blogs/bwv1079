---
layout: post
title: Variadic template patterns - part two
categories: [C++, C++11, C++14, C++17, variadic templates, templates, concepts]
permalink: /varargs-patterns-2/
---

In the [first part]({{site.baseurl}}/../2020-07-02-varargs-patterns-1) of 
this article we looked at the basics of variadic templates and showed a few usage patterns, in this second part we will expand the use cases to cover more advanced patterns. Concepts, introduced in the *C++20* standard
and fold expressions (*C++17) are used to simplify implementations.

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
other constructs, it turns out to be a pretty convoluted implementation with
a pre-set limit on the number of supported arguments, newer implementations
of `boost::tuple` do have full support for the latest C++ standard features.

Writing a full C++98 tuple implementation in-line in this blog article is not
viable but you can check for yourself one of my implementations 
[here](https://github.com/ugovaretto-codex/cpp/blob/master/tuple98-virtual-inheritance.cpp).

Two implementations are presented: one is employing explicit element selection through a 
utility `struct`, the other is using concepts to
specialise template implementations, greatly simplifying the code; do compare
the concept-based implementation with the C++98 version referenced above.

### 1. Explicit element selection

An `Nth<>` construct is implemented to extract the n-th element from the tuple
and declare the return type of `get` function.

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

Template construct to extract n-th type from type list, required to retrieve
type information from tuple index.

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

No partial template specialisation allowed on functions: overload.
The following `struct` transforms an index into a type to allow specialisation
for `index == 0` case.

```cpp
template <size_t I> struct IntToType {};
```

Have `get<Index>()`function invoke helper functions, overloaded with `IntToType<Index>` type.


```cpp
template<size_t I, typename T, typename...RestT>
constexpr const typename Nth<I, T, RestT...>::Type&  
get_helper(const Tuple<T, RestT...>& t, const IntToType<0>&) {
    return t.value;
}

template<size_t I, typename T, typename...RestT>
constexpr const typename Nth<I, T, RestT...>::Type&  
get_helper(const Tuple<T, RestT...>& t, const IntToType<I>&) {
    using Base = typename Tuple<T, RestT...>::Base;
    return get_helper<I-1>((const Base&) t, IntToType<I-1>());
}

template<size_t I, typename T, typename...RestT>
constexpr typename Nth<I, T, RestT...>::Type&  
get_helper(Tuple<T, RestT...>& t, const IntToType<0>&) {
    return t.value;
}

template<size_t I, typename T, typename...RestT>
constexpr typename Nth<I, T, RestT...>::Type&  
get_helper(Tuple<T, RestT...>& t, const IntToType<I>&) {
    using Base = typename Tuple<T, RestT...>::Base;
    return get_helper<I-1>((Base&) t, IntToType<I-1>());
}

template <size_t I, typename...ArgsT>
constexpr typename Nth<I, ArgsT...>::Type&
get(Tuple<ArgsT...>& t) {
    return get_helper(t, IntToType<I>());
}

template <size_t I, typename...ArgsT>
constexpr const typename Nth<I, ArgsT...>::Type&
get(const Tuple<ArgsT...>& t) {
    return get_helper(t, IntToType<I>());
}
```

Finish up. `constexpr` declaration allows `get` to be used in non-type
template parameter lists.

```cpp
template <int I>
struct Int {
    static const int i = I;
};

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
 <a href="https://godbolt.org/z/haYeh9">run</a>
</div>

### 2. Element selection through *concepts*

Instead of an `Nth` construct and function specialisation through overloading,
concepts are used directly to partially specialise functions without the need
of helpers.

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
3. not equal operator, to allow `begin()` - `end()` iteration
4. iterator type requirements

Inernally, an operator tuple stores all the iterators to zip.

### 1. Increment operator

The increment operator (`++`) takes care of incrementing all the iterators at once, 
by calling a method like:

```cpp
    template <size_t... I>
    void IncIterators(const index_sequence<I...>&) {
        (get<I>(its_)++, ...);
    }
```

An index sequence type is required to have the `get<>` function applied to 
all the iterator tuple elements through template parameter pack expansion.
The fold operator `,...` triggers the expansion.

### 2. De-reference operator

Values are returned by dereferencing (`*`) the iterator as done for common
pointers.
As previously done an index sequence is used to invoke the `get<>` function
to apply it to all the the elements in the iterator tuple.  

```cpp
    template <size_t... I>
    ValueTuple Values(const index_sequence<I...>&) const {
        return ValueTuple(*get<I>(its_)...);
    }
```

### 3. Equality operator

In order to support algorithms which iterate over a sequece using iterators
with code like

```cpp
while(b++ != e) {
    ...
}
```

where `b` and `e` are iterators, `operator !=` must be implemented, an
implementation of an equality method follows.
An index sequence is required to access all elements in the tuple and a fold
operator (`... &&`) is used to trigger the expansion and returnd the reduced
`and` operation. 

```cpp
    template <size_t... I>
    bool Equal(const Zipper& other, const index_sequence<I...>&) const {
        return (... && (get<I>(its_) == get<I>(other.its_)));
    }
```

### 4. Iterator type requirements

The *Standard Template Library* employs *traits* types to optimize value
initialisation and algorithm execution by specialising the implementation
depending on the specific type definitions declared in the class.

The following type definitions are used to specialise the *iterator* type:

```cpp
    using difference_type = ptrdiff_t;
    using value_type = tuple<ArgsT...>;
    using pointer = value_type*;
    using reference = value_type&;
    using iterator_category = forward_iterator_tag;
```

### Implementation

A full implementation is shown below.

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

## Index sequences

As shown above index sequences are used to expand the variable argument 
parameter pack in case of non-type, integral, template parameters.
This type has been available since the *C++14* standard, but it might be useful
to look at an actual minimal implementation.
The type itself is only useful to transfer the non-type template argument list
to a function or class and it is rarely used directly.

See [here](https://en.cppreference.com/w/cpp/utility/integer_sequence) for sample usage.

The implementation of a compile-time integer sequence is straightforward, while
normally templates are expanded recursively decrementing a counter and
reducing the number of elements passed to the next iteration step, in this
case the counter gets incremented but the non-type template list grows at
each step.

A new number equal to the size of the computed argument list plus one.
This works because in zero-based indexing the size of the sequence is always
equal to one plus the last index value.

```cpp
template <int S, int... N>
struct Sequence : Sequence<S - 1, N..., sizeof...(N) + 1> {};
```

Full implementation shown below. Fold expressions to iterate over elements used
for C++17 and above compilation.

```cpp

#include <iostream>

template <int... N>
struct Idx {};

template <int S, int... N>
struct Sequence : Sequence<S - 1, N..., sizeof...(N)> {};

template <int... N>
struct Sequence<0, N...> {
    using Index = Idx<N...>;
};

template <int Size>
struct MakeIndexSequence {
    static_assert(Size > 0, "Size <= 0");
    using Type = typename Sequence<Size - 1, 0>::Index;
};

#if __cplusplus >= 201703L
template <int...I>
void PrintIndices(Idx<I...>) {
    (..., (std::cout << I << " "));
    std::cout << std::endl;
}
#else
template <typename T>
void PrintIndicesHelper(T) {}

template <int H, int...Tail>
void PrintIndicesHelper(Idx<H, Tail...>) {
    std::cout << H << " ";
    PrintIndicesHelper(Idx<Tail...>());
}

template <int...I>
void PrintIndices(Idx<I...>) {
    PrintIndicesHelper(Idx<I...>());
    std::cout << std::endl;
}
#endif

int main(int argc, char const *argv[]) {
    constexpr int SIZE  = 3;
    PrintIndices(MakeIndexSequence<SIZE>::Type());
    return 0;
}
```

<div style='text-align: right' markdown='1'>
  <a href="https://godbolt.org/z/4Kb8oE">run</a>
</div>
