---
layout: post
title: R-value references and type deduction
categories: [C++, C++11, move semantics, r-value references, type deduction]
permalink: /r-value-references-and-type-deduction/
---

While experimenting with C++11 perfect forwarding (i.e. all arguments to a
function are forwarded in their actual current form to another function
maintaining all the original qualifiers and types) I ended up with the following
code which compiles fine:

~~~~~~~~~cpp
template < typename T >
void Forward(T&& ) {
...
}
...
int i = int();
Forward(i);
~~~~~~~~~

and the following code that does not

~~~~~~~~~
void Forward(int&& ) {
...
}
...
int i = int();
Forward(i);
~~~~~~~~~

Th reported error is:

~~~~~~~~~
: error: no matching function for call to 'Forward'
Forward(i);
^~~~~~~
~~~~~~~~~

To understand what is going on let's modify the code like this:

~~~~~~~~~cpp
template < typename T >
void Forward(T&& v) {
v = T(2);
}
...
int i = 1;
Forward(i);
~~~~~~~~~

This time the error we get when compiling is:

~~~~~~~~~cpp
error: functional-style cast from rvalue to reference type 'int &'
v = T(2);
^~~
~~~~~~~~~

It is telling us the `T` type is an `int&` and not simply an `int`.

So how come an `int&` declaration got transformed into an `int&` reference ?

That is because i is an lvalue and when passing an lvalue to an rvalue reference
the lvalue is passed by reference.

The template version works because, through template type deduction rules, the
type `T (int)` is automatically transformed into `T& (int&)` and `T& &&` automatically
collapses to `T&`.

When not using templates there is no `T` ⇒ `T&` conversion taking place and
therefore the function does not receive a reference to an lvalue but an actual
lvalue instead, and it does complain that the received type is not valid.

The template deduction rules are crafted this way to keep the original type
information intact when passing values to rvalue reference parameters, i.e.:

* an lvalue reference is mapped to an lvalue reference
* an rvalue is mapped to an rvalue

Keeping the original type information intact is what allows parameters
received as rvalue references to be forwarded to callable objects (functions,
methods, function objects) through the `std::forward` function; for a sample use
of this feature checkout note 4 on page 571 of the ISO/IEC 14882:2011
standard where the function call wrapper interface is explained:

> A forwarding call wrapper is a call wrapper that can be called with an arbitrary
> argument list and delivers the arguments to the wrapped callable object as
> references. This forwarding step shall ensure that rvalue arguments are
> delivered as rvalue-references and lvalue arguments are delivered as
> lvalue-references. [Note: In a typical implementation forwarding call wrappers
> have an overloaded function call operator of the form
> 
> template
> R operator()(UnBoundArgs&&... unbound_args) cv-qual;
>
> —end note ]
