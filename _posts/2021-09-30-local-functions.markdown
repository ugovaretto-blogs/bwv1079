---
layout: post
title: Local functions
categories: [C++, C++11]
permalink: /local-functions/
---

Local functions are supported through non-portable custom extensions in GCC
and Clang but are not part of the C++ standard.
They have however always been supported through static member functions of local
classes with visibility (closed) on all `static` and `thread_local` variables
declared in the enclosing scope.
Starting with C++11 local functions are also supported trough non-capturing
lambdas, which are also closed on `thread_local` and `static` local variables.

Local functions can be used to e.g. implement generators or return stateful
free functions interoperable with `C` libraries.

## Implementation

A counter generator looks like:

```cpp
using Counter = int (*)();

Counter GenCounter(int start) {
    thread_local int val = start;
    struct __ {
        static int _() { return val++; }
    };
    // C++11: return [](){ return val++;}
    return __::_;
}
```

...used like this:

```cpp
// generate counter starting at 10
auto count = GenCounter(10); 
for(int i = 0; i != 10; ++i) cout << count() << " ";
```

which results in the following output:

```sh
10 11 12 13 14 15 16 17 18 19
```

Now, this works but what you cannot di is using the same generator function
to create multiple independent counters.
The following code will have both `counter1` and `counter2` increment the same
variable.

```cpp
auto counter1 = GenCounter(2); // initialize counter variable to 2
auto counter2 = GenCounter(3); // initilize same counter variable to 3
cout << counter1() << " " << counter2() << endl;
```

output

```sh
3 4
```

instead of the desired `2 3` output.

One way to generare unique generators at compile time is to just add a template
parameter to idendify different instances.

```cpp
template <int ID = 0>
Counter GenUniqueCounter(int start) {
    thread_local int val = start;
    struct __ {
        static int _() { return val++; }
    };
    // C++11: return [](){ return val++;}
    return __::_;
}

```

now the following code works as expected:

```cpp
auto counter1 = GenUniqueCounter(2); // initialize counter variable to 2
auto counter2 = GenUniqueCounter(3); // initilize same counter variable to 3
cout << counter1() << " " << counter2() << endl;
```

output

```sh
2 3
```

This approach allows to e.g. implement OOP concepts by storing a local instance
of the data inside an enclosing function and accessing the instance through
a standard C structure with function pointers or C++ classes not containing any
data members.

The following code implements a constructor returnin a C++ `struct` containing
only function pointers and a couple of methods to perform type-safe access to
data.

```cpp
// Create Object containing C function pointers to access the Sphere
// instance stored within the function
template <int ID>
Object MakeSphere(double radius, const Vec3D& position) {
    thread_local Sphere s = {SPHERE, radius, position};
    using ApplyFun = void (*)(Sphere*);
    struct __ {
        // return empty intersection data
        static ISect Intersect_(const Ray& ray) { return ISect(); }
        static void Apply_(void* f, const std::type_info& ti) {
            if(!f || ti != typeid(Sphere)) throw std::bad_cast();
            auto fun = reinterpret_cast<ApplyFun>(f);
            fun(&s);
        }
        static void* Get_(const std::type_info& ti) {
            if(ti != typeid(Sphere)) throw std::bad_cast();
            return &s;
        }
    };
   return {.Intersect = __::Intersect_, .CheckedApply = __::Apply_,
           .CheckedGet = __::Get_};
};

```

Code implementing the discussed concepts and more is available
[here](https://github.com/ugovaretto/cpp11-scratch/blob/master/training/local_functions.cpp)

## Run-time instances

The above code can only generate instances at compile-time; run-time generation
requires storing instances in local data stores (e.g. `unordered_map`)
and passing an instance id around.
One advantage of keeping all instances in one place outside the class used to
access the data is the ability to use an external centralised (possibly remote)
data storage, with the client code staying the same regardless of where
the data is actually stored.

```cpp
template <typename...ArgsT>
pair<InstanceID, Methods> GenInstance(TypeID type, ArgsT...initParameters) {
    thread_local unordered_map<InstanceID, InstanceData> instanceStore;
    const InstanceID id = NewID();
    instanceStore[id] = CreateInstance(type, initParameters...);
    struct __ {
        static void Render(InstanceID id) { RenderFun(instanceStore[id]);}
        static void Delete(InstanceID id) { if(Valid(id)) instanceStore.erase(id);}
        ...
    };
    return {id, {.Render = __::Render, .Delete = __::Delete,...}}; 
}

class Object {
    template <typename...ArgsT>
    Object(const ArgsT&...args) {
        auto im = GenInstance(args...);
        instance_ = im.first;
        methods_ = im.second;
    }
    Object(Object&& other) : instance_(other.instance_) {
        other.instance_ = InvalidInstanceID();
    }
    void Render() { methods_.Render(instance_); }
    ~Object() { methods_.Delete(instance_); }
private:
    Methods methods_;
private:
    InstanceID instance_;
}
```

Smart pointers and lock guards can be employed to allow the use of `static`
instead of `thread_local` variable and sharing.

