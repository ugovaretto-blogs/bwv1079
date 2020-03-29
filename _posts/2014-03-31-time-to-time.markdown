---
layout: post
mathjax: true
title: Time to time
categories: [C++, C++11, time, timing]
permalink: /time-to-time/
---
{% include mathjax.html %}

_[edited March 2020]_

The C++11 standard defines a multi-platform framework to handle time
information which can be used to represent full time with date or time
differences suitable for timing code execution.

This means no more messing around with _clock_gettime()_,
_gettimeofday(), _QueryPerformanceCounter()_ or _GetSystemTime()_.


This article does cover the use of C++11 features for computing
elapsed time in different scenarios, for complete information on the
subject do have a look at the standard specification document (ISO/IEC
14882:2011), chapter 20, sections 10 and 11.

After introducing some basic concepts the various types are analyzed
bottom up starting with the basic types and ending with the clock
type.

The source code referenced at the end of the post has sample code
showing the usage of each of the types described below.

## Basic Concepts

The time framework is defined in the `<chrono>` include file.

Three clock types are provided:

* _system_: wall clock time, matches the system time
* _steady_: guarantees that the returned time value never decreases as
  physical time increases
* _high_resolution_: uses the shortest possible tick period, on most
  systems all clocks use a tick of one nanosecond anyway

A point in time is recorded into a `time_point` instance.

A time span is stored into a `duration` instance.

A difference between two `time_point`s returns a duration instance.

A duration represents a number of ticks or periods.

The period represents the length of a tick in seconds and it is
encoded with a rational constant.

The rational (numerator over denominator) constants used to describe a
tick period in seconds are represented with the ratio type.

Facilities exist to compare and cast durations and time points as well
as to convert to standard C types and print time information.

The _steady_ clock is the preferred choice for timing code execution.

All the needed types and functions other than ratio are declared
inside the `std::chrono` namespace.

`ratio` is declared inside the `std` namespace.

## Rational constants

Rational constants i.e. $\frac{Numerator}{Denominator}$ are used
hroughout the chrono framework to specify fractions of
seconds. Also a number of predefined constants is available to specify
commonly used units such as milli or micro.

Rational constants are declared as `ratio` type instances.

```cpp
    template < intmax_t N, intmax_t D = 1 >
    class ratio { 
    public: 
        typedef ratio< num, den > type; 
        static constexpr intmax_t num; 
        static constexpr intmax_t den; 
    };
```

The `intmax_t` type is the integer type that able to represent the biggest
possible signed integer.

Numerator and denominator values are defined at compile time as
integer numbers.

Arithmetic operations and comparisons can be executed at compile time
through the _ratio_*_ functions and types.

## Duration

The duration type represents a difference between points in time
defined as a number of periods (ticks).

From the standard:

A duration type measures time between two points in time
(time_points). A duration has a representation which holds a count of
ticks and a tick period. The tick period is the amount of time which
occurs from one tick to the next, in units of seconds. It is expressed
as a rational constant using the template ratio.

The number of ticks is the representation type which can be an integer
or a floating point number.

A period is an interval in seconds specified as a rational constant.

The following code defines a duration of ten milliseconds and prints
out the number of periods in the duration.

```cpp
    //1/1000
    using Milli = ratio<INTMAX_C(1), INTMAX_C(1000)>;
    using Milliseconds = duration<intmax_t, Milli>;
    //declare a duration of 10 periods where each period is equal
    //to 1/1000 seconds i.e. ten milliseconds
    Milliseconds ms(10);
    //print the number of periods (ticks) in the duration instance
    cout << ms.count() << endl;
```

The `intmax_t` type is the integer type that can represent the biggest
possible signed integer.

`INTMAX_C` is a macro that converts the integer constant into an
`intmax_t` instance.

It is possible to use floating point values as the type that
represents the number of tick counts as well.

You convert between durations with different representations and
period types by means of the `duration_cast` function.

A duration can be converted to a duration with a different tick period
only if the number of ticks is represented as a floating point number.

To detect if a tick count representation is a floating point type you
use the `treat_as_floating_point` type trait:

```cpp
    using FloatNano = duration< double, nanoseconds >;
    //check if floating point
    static_assert(
    treat_as_floating_point< FloatNano::rep >::value,
    "Duration representation must be a floating point type&amp;amp");

    //if we get here it means the duration tick count type is
    //a floating point type
    FloatNano fd(500.0);
    auto d = duration_cast<milliseconds>(fd); //0.0005 ms
```

The minimum, maximum and zero representation for a specific tick count
type are returned by the _min()_, _max()_ and _zero()_ static methods found
in the _duration_ type.

## Time point

```cpp
    template<
        class Clock,
        class Duration = typename Clock::duration
    > class time_point;
```

The `time_point` type is the representation of a point in time and it is
parametrized on the clock and duration types.

The reason for which the _time_point_ type is dependent on the clock
type is because the time point represents the number of periods
(ticks) elapsed from the beginning of time and such beginning changes
depending on the clock type: it could be e.g. the start of the current
process or a specific point in physical time.

The dependency on the duration type is required because the _time_point_
type is implemented as a duration representing the elapsed time since
the beginning of time.  Such duration is returned by the
_time_since_epoch()_ member. The default duration type is the duration
type declared in the clock type.

You convert between time points with different durations with the
`time_point_cast` function.

To convert a `time_point` to a standard `time_t` type use the static
member function `to_time()` found in the clock type.

To print a time point you use the `put_time()` library function.

You can also easily print a `time_point` instance by first converting
it to `time_t`:

```cpp
    #include <chrono>
    #include <ctime>
    ...
    using Clock = std::chrono::system_clock;
    std::time_t now = Clock::to_time_t(Clock::now());
    cout << std::ctime(now) << endl;
```

## Clock

The clock type returns the time_point instance representing the
current time whenever its static _now()_ method is invoked.

The three clock types required to be available in C++11 STL
implementations are:

* _system_clock_: tracks physical wall clock time; invoking now() on this
clock at two different times does not guarantee that the second call
to now() returns a time greater than the previous call to now()
* _steady_clock_: monotonic time function, the time returned by now() is
always increasing
* _high_resolution_clock_: smallest possible period

On many systems the resolution period is one nanosecond for all clock
types.

You have actually already seen all you need to know about clock usage
in the previous section i.e. how to call the _now()_ and _to_time_t()_
static methods.

Now let's look at some properties unique to each clock type.

The properties that identify a specific clock type are:

* period 
* origin of time
* steadiness

The following code shows how to retrieve and print the above information
given a clock type:

```cpp
    using Clock = std::chrono::system_clock;
    using TimePoint = Clock::time_point;
    const bool isSteady = Clock::is_steady;
    //one period (tick) is equal to num/den seconds
    const intmax_t periodNum = Clock::period::num;
    const intmax_t periodDen = Clock::period::den;

    //the default constructor for a time_point initializes
    //it to the origin of time
    std::time_t epoch = Clock::to_time_t(TimePoint());
```

## Timing execution

You now have all the information required to easily time your own
code.

For timing execution do use a clock which has an is_steady member set
to true e.g. steady_clock.

```cpp
    using namespace std;
    using namespace chrono;
    using Timer = steady_clock;
    using Time = Timer::time_point;
    using Duration = Timer::duration;
    using ms = ratio< INTMAX_C(1), INTMAX_C(1000) >;
    auto Tick = []{ return Timer::now(); };

    const Time begin = Tick();
    //execute code
    ...
    const Time end = Tick();
    const Duration elapsed = end - begin;
    cout << "Elapsed time: "
    << duration_cast< duration< intmax_t, ms > >(elapsed)
    << "ms" << endl;
```

Instead of your own ratio constant you can use the predefined
constants: milli, micro, ...

Instead of your own duration constant you can use the predefined
constants: milliseconds, microseconds, ...

e.g.:

```cpp
...
    //use std::chrono::milliseconds duration type
    cout << duration_cast< milliseconds >(elapsed).count() << endl;
...
```

Checkout the ISO standard, cppreference or your own STL documentation
for a complete list of such constants.

## Code

Sample code showing the use of the various chrono types:
[Sample code](https://github.com/ugovaretto/cpp11-scratch/blob/master/training/time-1.cpp)

A minimal implementation of a C++11 compliant timer class:
[Timer](https://github.com/ugovaretto/cpp11-scratch/tree/master/training/timer

