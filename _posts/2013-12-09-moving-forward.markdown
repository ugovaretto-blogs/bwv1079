---
layout: post
title: Moving forward
categories: [C++, C++11, Move semantics]
permalink: /moving-forward/
---

C++11 introduces the concept of r-value reference.

R-value references allow you to modify a temporary object and are used to e.g.
implement move semantics to "move" content from a temporary source object to a
target object while avoiding unnecessary copies.

Such concept also requires an additional set of facilities to be implemented to
properly manage parameter forwarding and move operations.

Specifically:

* when you want to explicitly ask for an object to be "moved" you call the
`std::move` function;
* when you want to forward parameters received by a function
to another function preserving their type you call `std::forward`

`std::move` transforms any reference type `T&` into an r-value reference `T&&`.

`std::forward` returns an l-value reference if the input type is an l-value
reference type and an r-value reference if the input value is *not* an l-value
reference.

The use of `std::move` is straightforward: use it whenever you want to force an
object to be moved by passing it as an r-value reference to whatever callable
entity or assignment operator you are invoking.

To understand why `std::forward` is required consider the following:

* an r-value reference is a reference to an unnamed object (array elements are not
considered unnamed because they are accessed through a named value);
* in cases where you have a function that accepts an r-value reference and such function
wants then to call another function passing to it the received value as an
r-value reference you have a problem because the value received as an r-value
reference now has a name and it is not an r-value reference any more, see below

~~~~~~~~~cpp
template < typename T >
void bar(T&& i) {...}

template < typename T >
void foo(T&& p) { //receive an r-value reference
    //p has a name therefore it is not an r-value reference
    //pass an l-value reference to bar
    bar(p);
}

foo(2); //call foo with and r-value reference to "2"
~~~~~~~~~

The function foo receives the parameter p as an r-value reference, but now p is
the name assigned to the received value and therefore, within the body of foo, p
is not an r-value reference any more but simply an l-value, so the bar function
is now called with an l-value reference instead of an r-value reference, i.e. we
are calling bar with a type different from the type received by foo.

In order to call the bar function with the same type as received by foo you need
to:

1. check the type of p and
2. cast to T& if p is an l-value reference and T&& if p is not an l-value reference

So, let's implement a version of `std::forward` that does exactly what it is
described above:

~~~~~~~~~cpp
//T& specialization
template < typename T >
T&& forward(T& v) {
    return static_cast<T&&>(v);
}

//T&& specialization
template < typename T >
T&& forward(T&& v) {
    return static_cast<T&&>(v);
}

//works if called as forward(...)
//does not work when forwarding a reference: forward<int&>(...)
~~~~~~~~~

All that forward does internally is to perform a `static_cast<T&&>(...)`.

The return type is always an r-value reference type which acts as a
generic reference, i.e. something that becomes either an l-value or an
r-value reference after specialization and type deduction occur.

The reason for which `forward` can return the right type by returning a templated
r-value reference type is because of template type deduction rules, discussed in
[this]({{site.baseurl}}/r-value-references-and-type-deduction/) post.

Basically:

* `Type &` (l-value ref) → `T&&` → `Type& &&` →`Type&`
* `Type &&` (r-value ref or value) →  `T&&` → `Type&&` or `Type&& &&` →`Type &&`

Where `T` can also be qualified as `const` and/or `volatile`.

This means that by returning a `T&&` type you are actually returning a generic
reference which is defined as `T&` or `T&&` at compile time depending on the 
type of `T`.

Now, this version works if you do not specialize it with a reference type, to
make it work with reference types you need to remove the reference from the type
passed to the function:

~~~~~~~~~cpp
#include <type_traits>

template < typename T >
T&& forward(typename std::remove_reference<T>::type& v) {
    return static_cast<T&&>(v);
}
~~~~~~~~~

now we have an implementation that works as the standard `std::forward` library
function.

Well, almost...

~~~~~~~~~cpp
//this compiles!
int& r = forward<int&>(2);
//r is a non-const reference to a temporary object
~~~~~~~~~

the code above does create an l-value reference pointing to a temporary object!

To catch r-value reference to l-value reference conversion attempts you can use
a static_assert in the function body:

~~~~~~~~~cpp
template < typename T >
T&& forward(typename std::remove_reference<T>::type&& v) {
    static_assert(!std::is_lvalue_reference<T>::value,...);
    return static_cast<T&&>(v);
}
~~~~~~~~~

Now the implementation is almost exactly the same as the one found in most STL
implementations, where *almost* means missing constexpr and noexcept
implementations.

With the current implementation:

* a value is forwarded as an r-value reference to a temporary object
* an r-value reference is forwarded as an r-value reference
* an l-value reference is forwarded as an l-value reference

What you cannot do is this:

~~~~~~~~~cpp
template < typename T >
constexpr int C(T&& ) { return 1; }

template < typename T >
constexpr int CallC(T&& v) { return C(forward< T >(v)); }
~~~~~~~~~

That is because our implementation of `forward` is not declared as `constexpr`.

The final standard conforming implementation of forward is then:

~~~~~~~~~cpp
template < typename T >
constexpr T&&
    forward(typename std::remove_reference<T>::type& p) noexcept {
        return static_cast<T&&>(p);
}

template constexpr T&&
forward(typename std::remove_reference<T>::type&& p) noexcept {
    static_assert(
        !std::is_lvalue_reference::value,
        "ERROR: R-value reference to L-value reference cast");
    return static_cast<T&&>(p);
}
~~~~~~~~~

And here is a small program to test different calling patterns.

~~~~~~~~~cpp
#include <iostream>
#include <utility> //move and forward

//------------------------------------------------------------------------------
//overloaded functions called by Forward function
void Overfoo(int& arg) { std::cout << "L-value\n"; }
void Overfoo(int const &arg) { std::cout << "const L-value\n"; }
void Overfoo(int&& arg) { std::cout << "R-value\n"; }
//note: returning a const r-value reference is useful to forbid code to use
//temporary values returned by functions:
//foo(const int& ): accepts temporary instances of T returned by functions
//as xvalues
void Overfoo(const int&& arg) { std::cout << "const R-value\n"; }
void Overfoo(volatile const int &arg ) {
    std::cout << "volatile const L-value\n";
}
void Overfoo(volatile int &arg) { std::cout << "volatile L-value\n"; }
void Overfoo(volatile int && arg) { std::cout << "volatile R-value\n"; }
void Overfoo(volatile const int && arg) { std::cout << "const volatile R-value\n"; }

template < typename T >
void Forward(T&& arg) {
//note: arg is a local NAMED parameter so it is NEVER of type &&, in order
//to properly forward it as an r-value reference you need to use the
//forward function
    std::cout << " std::forward -> ";
    Overfoo(std::forward< T >(arg));
    //note: move transforms anything to an R-value
    std::cout << " std::move -> ";
    Overfoo(std::move(arg));
    std::cout << " -> ";
    Overfoo(arg);
}

//------------------------------------------------------------------------------
int main() {
    std::cout << "R-value ->\n";
    Forward(0);
    std::cout << "L-value ->\n";
    int i = 1;
    Forward(i);
    std::cout << "const L-value ->\n";
    const int ci = 2;
    Forward(ci);
    std::cout << "volatile L-value ->\n";
    volatile int vi = 3;
    Forward(vi);
    std::cout << "volatile const L-value ->\n";
    volatile const int cvj = 4;
    Forward(cvj);
    std::cout << "++i ->\n";
    Forward(++i);
    std::cout << "i++ ->\n";
    Forward(i++);
    return 0;
}
~~~~~~~~~

If you compile and run this program you'll see from the output that std::forward
does in fact forward parameters to Overfoo with the same exact type passed to
the `Forward` function, which is not the case when forwarding parameters without
calling `std::forward` explicitly, this behavior is also known as perfect
forwarding.
