---
layout: post
title: Varargs patterns
categories: [C++, C++11, C++14, C++17]
permalink: /varargs-patterns/
---

# Varargs patterns

`C++11` introduces variable length template argument lists, meaning that the
list of template arguments does not have to be fully specified using an actual
type, the `...` operator can be used instead i.e.

```cpp
template <typename...ArgsT>
```

instead of

```cpp
template <typename T1, typename T2>
```

and it does work equally well with non-type template parameters.

The `...` operator causes the replacement, of `Expression...` with the
actual comma separated list of template expressions defined at compile time.

The standard way of expanding the parameter pack at compile time is to
employ pattern matching, with recursive calls and a termination condition:

```cpp
#include <iostream>

using namespace std;

// termination condition
void FooImpl() {}

// pattern matching: ArgsT... --> Head, ArgsT... - extract and consume head
template <typename HeadType, typename...TailTypes>
void FooImpl(HeadType&& head, TailTypes&&...tail) {
    cout << "called with head = " << head << endl;
    FooImpl(std::forward<TailTypes>(tail)...);
}

// varargs function, relies on helper function to extract and process
// one element at a time
template <typename... ArgsT>
void Foo(ArgsT&&...args) {
    FooImpl(std::forward<ArgsT>(args)...);
}

int main() {
  
  Foo(1,2,3,4, "hello");
  
  return 0;
}
````

<div style="text-align: right"> 
    <a href="https://godbolt.org/z/axW5dP">code</a>
</div>

The line

```cpp
FooImpl(std::forward<ArgsT>(args)...);
```

gets expanded to:

```cpp
void FooImpl<int, int, int, int, char const (&) [6]>(int&&, int&&, int&&,
                                                     int&&, char const (&) [6])
```
