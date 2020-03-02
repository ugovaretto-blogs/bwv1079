---
layout: post
title: Variadic templates and code maintainability
categories: [C++, C++11, variadic templates, templates]
permalink: /variadic-templates-and-code-maintainability/
---

This post was originally simply titled *Variadic void***, but since it shows how
C++11 does indeed help in making it easier to maintain legacy code I decided to
change the title.

## The problem

While developing a new framework for generating CUDA code through
Clang/LLVM I restarted experimenting with the CUDA driver API which now has an
easier (as compared to < 4.x versions) way of launching kernels: one option is
to pass parameters through an array of void pointers.

Now, since in the past I created a C++ library
[gpupp](https://github.com/ugovaretto-accel/gpupp) to simplify OpenCL and
CUDA usage, I also wanted to update such library to work with the latest version
of OpenCL and the CUDA driver API.

When I first developed my little gpupp library variadic templates were not
available, so I resorted to type erasure through an *[Any](https://github.com/ugovaretto-accel/gpupp/blob/master/utility/Any.h)* type, and parameters were
passed to the kernel launcher function through a custom built *[varargs](https://github.com/ugovaretto-accel/gpupp/blob/master/utility/varargs.h)* type,
packing an array of *Any* instances.

## The solution

It turns out that through the new *void*** interface and variadic
templates offering a generic kernel launcher function is astonishingly easy.

I am publishing here the code used to implement a variadic list → void* array
conversion, note that the code allows the storing of references to temporary
objects because the use case I need to support is something like:

~~~~~~~~~cpp
template< typename ArgTypes... >
StatusCode Launch(dim3 blocks,
                  dim3 threads,
                  KernelFunction f, //CUFunction
                  ArgTypes&&...argumentsPassedToGPUKernel) {
    void* args[...];
    //make args[] elements point to references to
    //argumentsPassedToGPUKernel elements
    ...
    return cuLaunchKernel(/*launch grid configuration...*/,
                          f,
                          args,
                          nullptr);
}
~~~~~~~~~

All I need to do is to pass an array of pointers to references to a library
function which does not need the referenced elements to be valid after it
returns, so it is perfectly fine to create an argument array storing pointers to
references to parameters passed to the `Launch` function even if some of the
parameters are temporaries.

It turns out that all you need to convert a variadic argument list to a `void*`
array with elements pointing to the elements of the variadic argument list
itself is this one line:

~~~~~~~~~cpp
void* args[] = { &args... };
~~~~~~~~~

This works if you do not pass in const references, if you do, you get an error
from the compiler complaining that it cannot take the address of a reference and
convert it to a void pointer.

One easy fix is to just C-cast to void*.

~~~~~~~~~cpp
void* args[] = { (void*) &args... };
~~~~~~~~~

Bam!

Now you can call the function like e.g.

~~~~~~~~~cpp
double matrix[16];
const int iterations = 10;
Launch(...,
    ...,
    //templated arguments -> template deduction context
    iterations, // T = const int & -> const int & && -> const int &
    0.2, // T = double&& -> double&&
    matrix // T = double*& -> double*& && -> double&
);
~~~~~~~~~

If you are not happy with the C-style (void*) conversion and you really, really
want to use C++ cast operators you can write something like:

~~~~~~~~~cpp
void* args[] =
{const_cast< void* >(
static_cast< const void* >(
/*const_cast< typenam std::remove_reference<Args>::type* const>(&args)))...} OR*/
(typename std::remove_reference<Args>::type* const)(&args)))...};
/*OR other combination involving the use of reinterpret_cast*/
~~~~~~~~~

i.e.:`T or T& or const T& → const T* → const void* → void*`

Feel free to go ahead and compare this solution with the dozens of lines of code
I was forced to write in the past:

* `SetupKernelParameters` function in
  *[gpupp.cpp](https://github.com/ugovaretto/gpupp/blob/master/cuda/gpupp.cpp)*
* *[Any](https://github.com/ugovaretto/gpupp/blob/master/utility/Any.h)*
* *[VArgList](https://github.com/ugovaretto/gpupp/blob/master/utility/varargs.h)*

I actually also had another problem at the time: parameters could not be just
passed in as an array of void pointers but had to be manually added to a  stack
also specifying a properly aligned offset at each step, but 90+% of the code I
had to write was because variadic templates were not available!

By the way: in the meantime I also realized that with smart pointers (with
custom deleters) and std::chrono the entire utility directory in my gpupp
library becomes useless.

## Conclusion

DO use C++11 and start getting acquainted with rvalue references, auto, variadic
templates, lambda functions, decltype and the updated STL (but beware of the
differences among implementations).

You might find out that most of the code you have written can be rewritten in a
handful of lines or simply replaced with library functions and types, this also
includes some of the template metaprogramming, and macro code commonly
implemented through the use of the Boost libraries.

I also believe it is especially important to review old code you still need to
maintain and port it to C++11 because by reducing the amount of lines of code
and using STL features you'll make it easier to maintain and you might also fix
some bugs in the process (this is what happended to me anyway).

I personally find variadic templates the most effective device for simplifying
legacy code both in the case of interfaces to C libraries accepting (or
returning) arguments through arrays and when dealing with generic code that uses
boost::preprocessor or multiple overloads/specializations to implement types and
functions that accept a variable number of arguments.

For an additional example of how variadic templates simplify code previously
implemented with C++98 do check my different versions of a
[tuple
implementation](https://github.com/ugovaretto/cpp11-scratch/tree/master/training/tuple)
and compare the C++11 version with the others.
