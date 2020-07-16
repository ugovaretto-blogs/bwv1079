---
layout: post
title: Varargs patterns
categories: [C++, C++11, C++14, C++17]
permalink: /varargs-patterns/
---

# Variadic template patterns

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

##Basics

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
    <a href="https://godbolt.org/z/axW5dP">run</a>
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

##Argument expansion

Now, trying to trigger an expansion by writing

```cpp
...
template <typename T>
void Print(const T&, char sep = ' ') {
    cout << T << sep;
}
...
// pattern matching: ArgsT... --> Head, ArgsT... - extract and consume head
template <typename HeadType, typename...TailTypes>
void FooImpl(HeadType&& head, TailTypes&&...tail) {
    cout << "called with head = " << head << " tail = ";

    Print(tail)...; // <<<<<!

    FooImpl(std::forward<TailTypes>(tail)...);
}
...

```

will not work because in order for the variadic expression to be expanded
it has to be consumed by some other callable entitiy or initializer list,
which in turn requires the expression to be returning a value.

The following code will do the trick:

```cpp
#include <iostream>

using namespace std;

template <typename T>
int Print(const T& v) { // added return value
    cout << ' ' << v;
    return 0;
}

// termination condition
void FooImpl() {}

// pattern matching: ArgsT... --> Head, ArgsT... - extract and consume head
template <typename HeadType, typename...TailTypes>
void FooImpl(HeadType&& head, TailTypes&&...tail) {
    cout << "called with head = " << head << " tail =";
    //---------------------------------------------------------------------
    struct Expand {
        Expand(...) {} // C va_args list
    };

    Expand{Print(tail)...}; // <<<<! Trigger expansion
    //---------------------------------------------------------------------

    cout << endl;
    FooImpl(std::forward<TailTypes>(tail)...);
}

// varargs function, relies on helper function to extract and process
// one element at a time through the standard `list` --> `head:tail`
// pattern matching
template <typename... ArgsT>
void Foo(ArgsT&&...args) {
    FooImpl(std::forward<ArgsT>(args)...);
}

int main() {
  
  Foo(1,2,3,4, "hello");
  
  return 0;
}
```

<div style="text-align: right"> 
    <a href="https://godbolt.org/z/h6ojqh">run</a>
</div>

##Fold expressions

Instead of triggering an expansion by passing the vararg expression as an 
argument to a callable entity, fold expressions, available starting with
*C++17* can be used:

```cpp
#include <iostream>

using namespace std;

// termination condition
void FooImpl() {}

// pattern matching: ArgsT... --> Head, ArgsT... - extract and consume head
template <typename HeadType, typename...TailTypes>
void FooImpl(HeadType&& head, TailTypes&&...tail) {
    cout << "called with head = " << head << " tail =";
    //-------------------------------------------------------------------------
    (...,(cout <<  " " << tail)); // <<<<<!
    //-------------------------------------------------------------------------
    cout << endl;
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
```

<div style="text-align: right"> 
    <a href="https://godbolt.org/z/Ese78d">run</a>
</div>

Fold expressions are specified as:

```txt
... <operator> <vararg expression>`
```

where the `...` operator causes the expansion of the variadic template
expression following the operator.

In the code above `(...,(cout << " " << tail))` is interpreted as:

* `...`: fold operator
* `,`: comma operator
* `cout << " " << tail`: variadic expression, repeated as many times as there
  are elements in `tail`, separated by the `,` operator

## Template template parameters

When specifying template template parameters in parameter lists, without using
the ellipsis operator, you are required to know how many template parameters
the template template parameter type accepts; using a variadic template list
makes the code more generic:

```cpp
#include <iostream>
#include <cstdlib>
#include <vector>

using namespace std;

template <class T, size_t Alignment = sizeof(T)>
class AlignedAllocator { // >= C++11: 
                         // allocator_traits<allocator<T>> fills defaults
public:
    static constexpr size_t alignment = Alignment;
    using value_type = T;
    template <class U> struct rebind {
        using other = AlignedAllocator<U, alignment>;
    };
    AlignedAllocator() = default;
    template <class U> 
    AlignedAllocator(AlignedAllocator<U, Alignment> const&) noexcept {}
    value_type*  allocate(std::size_t n) {
        void* aa = aligned_alloc(alignment, n*sizeof(value_type));
        return static_cast<value_type*>(aa);
    }
    void deallocate(value_type* p, std::size_t) noexcept {
        ::operator delete(p);
    }
};

template < template <typename...ArgsT> typename ContainerT,
           typename...ArgsT >
struct Stack {
    using ContainerType = ContainerT<ArgsT...>;
    using ValueType = typename ContainerType::value_type;
    ContainerType container;
    void Push(const ValueType& v) {container.push_back(v);}
    ValueType Pop() {
        ValueType v = container.back();
        container.pop_back();
        return v;
    }
};

int main() {
  Stack<vector, float, AlignedAllocator<float, 4>> s;
  s.Push(2.4f);
  cout << s.Pop() << endl;
  return 0;
}

```

<div style="text-align: right"> 
    <a href="https://godbolt.org/z/bPYPT9d">run</a>
</div>
