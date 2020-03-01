---
layout: post
title: Constant r-value references
categories: [C++, C++11, r-value references, move semantics]
permalink: /constant-r-value-references/
---

R-value references are a type of reference which supports references to
temporary objects such as as the ones returned from functions.

The main uses of r-value references are:

* "move" the inner part of an object from a source to a destination instance to
avoid copying data from an object that will be destroyed anyway just after the
assignment or construction statement is executed
* transfer ownership (i.e. the responsibility to delete an object)

So why would anyone want to use a constant r-value reference which prevents the
move action to take place?

Because if you are holding a reference to an object for future use you do not
want to assign to it a reference to a temporary object which will disappear just
after the assignment operation and there is one case where in order to prevent
the assignment of a reference to a temporary object you need to deal with
constant r-value references.


Consider the following code:

~~~~~~~~~cpp
#include <iostream>

struct B {
    B(int i = 1) : i_(new int(i)) {}
    int* i_;
    ~B() {
        delete i_;
        i_ = nullptr;
        std::cout << "~B()" << std::endl;
    }
};


class C {
public:
    C(const B& b) : bref_(b) {}
    const B& b() const { return bref_; }
private:
    const B& bref_;
};

B NewB() {
    return B();
}

int main(int, char**) {
    C c(NewB());
    std::cout << *c.b().i_ << std::endl;
    return 0;
}
~~~~~~~~~

Compiling and running the above code results in an error:

~~~~~~~~~
~B()
Segmentation fault: 11
~~~~~~~~~

This is because the temporary reference to the `B` object points to an object that has
been destroyed.

In order to intercept such kind of errors you can add an explicit copy
constructor receiving an r-value reference which matched the temporary object
returned by the `NewB` function and throw and exception or report an error:


~~~~~~~~~cpp
class C {
public:
    C(const B& b) : bref_(b) {}
    const B& b() const { return bref_; }
    C(B&& b) : bref_(b) {
        std::cerr << "#####" << std::endl;
        throw std::runtime_error("Reference to temporary object");
    }
private:
    const B& bref_;
};
~~~~~~~~~
~~~~~~~~~
libc++abi.dylib: terminating with uncaught exception of type std::runtime_error: Referenc
e to temporary object
Abort trap: 6
~~~~~~~~~

It would be much better however, to report a compile-time error, which is what you
achieve by explicitly deleting the r-value reference constructor:

~~~~~~~~~cpp
class C {
public:
    C(const B& b) : bref_(b) {}
    const B& b() const { return bref_; }
    C(B&& b) = delete;
private:
    const B& bref_;
};
~~~~~~~~~

Now a proper compile time error is reported:

~~~~~~~~~
bash-3.2$ clang++ -std=c++11 s1.cpp
s1.cpp:29:7: error: call to deleted constructor of 'C'
    C c(NewB());
      ^ ~~~~~~
s1.cpp:19:5: note: 'C' has been explicitly marked deleted here
    C(B&& b) = delete;
    ^
1 error generated.
~~~~~~~~~

Great, so we fixed the problem!

...not quite; try to change the return type of the `NewB` function from `B` to
`const B`:

~~~~~~~~~cpp
const B NewB() {
    return B();
}
~~~~~~~~~

~~~~~~~~~
bash-3.2$ ./a.out
~B()
Segmentation fault: 11
~~~~~~~~~

What??

That's because in this case although the returned value is a temporary object,
being of `const` type it matched the standard const reference constructor.

Easy to fix though: just explicitly delete the `C(const B&&)` constructor:

~~~~~~~~~cpp
class C {
public:
    C(const B& b) : bref_(b) {}
    const B& b() const { return bref_; }
    C(B&& b) = delete;
    C(const B&& b) = delete;
private:
    const B& bref_;
};
~~~~~~~~~

Again, error at compile time:

~~~~~~~~~
bash-3.2$ clang++ -std=c++11 s1.cpp
s1.cpp:30:7: error: call to deleted constructor of 'C'
    C c(NewB());
      ^ ~~~~~~
s1.cpp:20:5: note: 'C' has been explicitly marked deleted here
    C(const B&& b) = delete;
    ^
1 error generated.
~~~~~~~~~

Now we have covered all the cases.

## A word on templates

In case you are dealing with template callable objects, i.e. instead of `B&`
references you have `T&` references where `T` is a template parameter, you do not
need to add an explicit delete for the const T&& case, this is because when a
callable object returns a constant value passed to another callable object the
value is already of type const U and it automatically invokes the `T&&` callable
object where `T` is a constant type of type `const U`.

To summarize: if you pass an object of type `const U` to `T&&` you end up with `const
U&&`, and if you pass it to `const T&&` you get `const const U&&`.

As an exercise try to have two versions of the same template function with `&&`
and `const &&` signatures and write code that invokes the `const &&` version.

In order to verify that removing the `T&&` version of callable entities is enough
to avoid receiving constant temporary instances I did replace the`const T&&`
signatures with `T&&` in *clang 3.3* standard library's `std::(c)` ref functions and
got exactly the same results whether I use `T&&` or `const T&&`.

Here is the compiler output when compiling code that tries to create a reference
to a constant temporary object obtained after changing the `std::cref` signature
to `T&&`:

~~~~~~~~~cpp
...
../const-r-value-ref.cpp:49:16: error: call to deleted function 'cref'
auto ref = std::cref(([] {
^~~~~~~~~
/usr/bin/../lib/c++/v1/__functional_base:425:27: note: candidate function [with _Tp = const C] has been explicitly deleted
template <class _Tp> void cref(_Tp&&) = delete;
...
~~~~~~~~~

pay attention to `with _Tp = const C and void cref(_Tp&&) = delete`.

What happened is that `std::cref` got a constant instance and substituted the
template parameter with a constant type, so no need to delete the`const &&`
version here because the case is already covered by the `&&` overload.

No idea why in the standard library they are deleting the `const T&&` overload, but
there probably is a reason, I just do not see it.

## Code

Feel free to play with the code available
[here](https://github.com/ugovaretto/cpp11-scratch/blob/master/training/const-r-value-ref.cpp)
experimenting with the various compilation options (read the comments).

