---
layout: post
title: Forward compile time iteration
categories: [C++, C++11, templates, metaprogramming]
permalink: /forward-compile-time-iteration/
---

Just a word on compile-time iteration with templates after having seen too much
code which for no good reason forces the iteration to always go backward and
specialize with one or zero to signal the termination condition, which is
especially bad in the case of e.g. a tuple implementation where you end up with
the types stored in the reverse order.

## Backward iteration

Standard backward compile-time iteration goes like:

~~~~~~~~~cpp
template < int Counter,.... > //iteration step
struct Iterator {
    static void Do() {
        ...
        Iterator< Counter - 1, ... > ::Do()
    }
};

template < ... > //end iteration
struct Iterator< 1, ... > {
    static void Do() {
    ...
    }
};
~~~~~~~~~

or with derivation:

~~~~~~~~~cpp
template < int Counter, .... >
struct Iterable : Iterable< Counter - 1, ... > {...};

template < ... > //end iteration
struct Iterable< 0, ... > {...};
~~~~~~~~~

Apart from iterating backward, you are also losing information about the actual
iteration length in each step, this is easily fixed by adding an additional
constant (i.e. all template specializations have the same value) template
parameter holding the number of iteration steps. As an example of a compile-time
backward iteration let's implement a function that prints a message a constant
number of times:

~~~~~~~~~cpp
#include <iostream>

template < int N >
struct ReversePrinter {
    static void Print() {
        for(int t = 0; t != N - 1; ++t) std::cout << " ";
            std::cout << "Iteration " << N << '\n';
        ReversePrinter< N - 1 > ::Print();
    }
};

template <>
struct ReversePrinter< 1 > {
    static void Print() {
        std::cout << "Iteration 0" << std::endl;
    }
};

int main(int, char**) {
    std::cout << "Standard reverse iterator:\n";
    ReversePrinter< 3 > ::Print();
    return 0;
}
~~~~~~~~~

If you compile and run the above code you get:

```
Standard reverse iterator:
    Iteration 3
  Iteration 2
Iteration 1
```

Note that it is not possible to properly indent the output because there is no
information about the number of steps so an additional template parameter has to
be added e.g.

~~~~~~~~~cpp
template < int N, ...., int NumIterations = N >
~~~~~~~~~

In order to keep the public interface intact it is also possible to create a
helper class accepting both the counter and number of iterations and use it to
perform the actual iteration.

## Forward iteration

Forward iteration can easily be implemented with the following pattern:

~~~~~~~~~cpp
template < int NumIterations, int Counter, ... >
struct Iterator {
    static void Do() {
        ...
        Iterator< NumIterations, Counter + 1, ... > ::Do()
    }
};

template < ... > //end iteration
struct Iterator< NumIterations, NumIterations, ... > {
    static void Do() {
        ...
    }
};
~~~~~~~~~

The trick hre is to use the amazing pattern matching capabilities of C++ templates
and specialize with `Counter == NumIterations`, i.e. stopping the iteration when
both `Counter` and `NumIterations` have the same value.
With this approach the compile-time print algorithm is easily implemented as:

~~~~~~~~~cpp
#include <iostream>

template < int N, int Count = 1 >
struct Printer {
    static void Print() {
        for(int t = 0; t != Count - 1; ++t) std::cout < < " ";
        std::cout << "Iteration " << Count << '\n';
        Printer< N, Count + 1 > ::Print();
    }
};

template < int N >
struct Printer< N, N > {
    static void Print() {
        for(int t = 0; t != N - 1; ++t) std::cout << " ";
        std::cout << "Iteration " << N << std::endl;
    }
};

int main(int, char**) {
    std::cout << "Forward iterator:\n";
    Printer< 3 > ::Print();
    return 0;
}
~~~~~~~~~

which leads to the following output:

```
Forward iterator:
Iteration 1
  Iteration 2
    Iteration 3
```

As you can see the iteration takes place in forward order.

The `Counter` itself can be default initialized to one or we can use a helper
class to leave the public interface untouched.

## Resources

[Code](https://github.com/ugovaretto/cpp11-scratch/blob/master/training/forward-template-iteration.cpp)

[A recursive tuple implementation
(C++11)](https://github.com/ugovaretto/cpp11-scratch/blob/master/training/tuple/tuple11.cpp)
