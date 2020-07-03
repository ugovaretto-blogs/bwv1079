---
layout: post
title: Varargs fun
categories: [C++, C++11, C++14, C++17]
permalink: /varargs-fun-1/
---

# Varargs fun

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

The `...` operator causes the expansion of the expression into 

Typical usage patterns include iteration over the element through recursive
compile time expansion such as:

...

In some cases it is possible to force expansion without applying recursion, this however will not work:

...

Because expansion 