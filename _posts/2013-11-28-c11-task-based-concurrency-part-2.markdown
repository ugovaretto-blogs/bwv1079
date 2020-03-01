---
layout: post
title: C++11 & task based concurrency - part 2
categories: [C++, C++11, Concurrency, Tasks]
permalink: /c11-task-based-concurrency-part-2/
---

# C++11 & task based concurrency


*Updated February 29, 2020*

As mentioned in the [previous]({{site.baseurl}}/c11-task-based-concurrency-part-1) post, C++11
does not offer an out-of-the-box implementation of task based concurrency but
only the option to schedule threads for execution.

This post shows a simple but functional implementation of a task based executor
using C++11 facilities such as threads and futures.

The way I would like to use a task-based executor is something like:

~~~~~~~~~cpp
const int NUMBER_OF_OS_THREADS = 2;
try {
    Executor exec(NUMBER_OF_OS_THREADS);
    auto f1 = exec([](){...});
    auto f2 = exec(sum, 1, 10, 1);
    auto f3 = exec(...);
    auto f4 = exec(...);
    ...
    f1.wait();
    int s = f2.get();
    ...
} catch(const std::exception& e) {
    std::cerr << e.what() << std::endl;
}
~~~~~~~~~

The number of OS threads is set to two meaning that at any given time no more
than two OS threads are active.

The number of tasks can be anything but at most two tasks are executed
concurrently.

## Requirements

To implement an *Executor* object that supports the desired functionality we
need to:

1. create OS threads once when the `Executor` instance is created and reuse them
later to invoke the callable objects
2. store the callable object together with its parameters for deferred execution
3. implement an `operator ()(...)` that accepts a callable object and any number
of parameters of any type which are passed to the callable object when invoked
4. forward exceptions from the OS threads to the parent thread (creator of
   `Executor` instance) 
5. find a way to stop execution

## Implementation 

Let's go through the requirements one by one.

### 1. - create threads once and reuse when needed

`std::thread`s shall be created when the *Executor* object is created and put
in a wait state by waiting for callable objects to be added to a queue shared
among threads.

Futures and promises are used to push results to the parent thread.

A thread shall:

1. pop a callable object from the shared queue or wait if none available
2. execute the callable object
3. go back to (1)

E.g.

~~~~~~~~~cpp
for(int t = 0; t != nthreads_; ++t) {
    threads_.push_back(std::move(std::thread( [this] {
        while(true) {
            ICaller* c = queue_.Pop();
            if(c->Empty()) { //interpret an empty Caller as a
                //'terminate' message
                break;
            }
            c->Invoke();
            delete c; //Use std::unique_ptr<ICaller> instead
        }
    })));
}
~~~~~~~~~

The code also shows how empty tasks are used to signal a termination request.

No memory allocation/ownership considerations discussed but most likely an
`std::unique_ptr` should be used to wrap caller objects.

### 2. and 3. - create task for deferred execution

`Executor::operator()(...)` shall accept a callable object and any number of
parameters through a variadic template list.

`Executor::operator()(...)` shall return a future which client code can use
to wait for results to be computed.

The actual implementation of operator()(...) shall package the callable object
together with the call parameters into a single object to be stored into an
execution queue shared by all OS threads.

~~~~~~~~~cpp
template < typename F, typename... Args >
auto operator()(F&& f, Args... args)
-> std::future< typename std::result_of< F (Args...) >::type > {
    if(threads_.empty()) throw std::logic_error("No active threads");
    using ResultType = typename std::result_of< F (Args...) >::type;
    Caller< ResultType >* c =
        new Caller< ResultType >(std::forward< F >(f),
                                 std::forward< Args >(args)...);
    std::future< ResultType > ft = c->GetFuture();
    queue_.Push(c);
    return ft;
}
~~~~~~~~~

Since all the objects in the shared queue have to be of the same type, we must
define an abstract base class then add to the queue instances of derived
concrete classes which store the actual callable object and parameters.

The full implementation for a *callable* class is shown below.
`void`specialization not shown.

~~~~~~~~~cpp
//interface and base class for callable objects
struct ICaller {
    virtual bool Empty() const = 0;
    virtual void Invoke() = 0;
    virtual ~ICaller() {}
};

//callable object stored in queue shared among threads: parameters are
//bound at object construction time
template < typename ResultType >
class Caller : public ICaller {
public:
    template < typename F, typename... Args >
    Caller(F&& f, Args&&... args) :
        f_(std::bind(std::forward(f),
                     std::forward(args)...)),
        empty_(false) {}
    Caller() : empty_(true) {}
    std::future< ResultType > GetFuture() {
        return p_.get_future();
    }
    void Invoke() {
    try {
        ResultType r = ResultType(f_());
        p_.set_value(r);
    } catch(...) {
        p_.set_exception(std::current_exception());
    }
    }
    bool Empty() const { return empty_; }
private:
    std::promise< ResultType > p_;
    std::function< ResultType () > f_;
    bool empty_;
};
~~~~~~~~~

### 4. - forward exceptions

Threads shall execute the callable objects inside a `try/catch` block and
in case an exception occurs they shall forward the exception to the parent
thread through`promise::set_exception.`.

### 5. - Stop execution

One way to stop execution is to add one empty callable object
per thread: when a thread pops a task from the queue and that task is marked
as `empty` the thread exits from the loop.

## The shared queue

A key part of the *Executor* implementation is the shared queue object
accessed by all OS threads.

The queue must use a synchronization mechanism to serialize access to its
elements to allow only a single thread at a time to push or pop data.

The push method locks the queue and adds an element.

The pop method locks the queue then checks if the queue is empty, if it is, it
unlocks the queue and waits for elements to be added. These operations are
executed as a unique atomic operation through the `std::wait` function.

Since the `SyncQueue` class is only used by `Executor` it might be a good idea to
make it a private inner class of `Executor` itself.

## Code

A sample implementation of an *Executor* class with comments and sample usage
is [available on
GithHub](https://github.com/ugovaretto/cpp11-scratch/blob/master/training/task-based-executor.cpp).

~~~~~~~~~cpp
template < typename T >
class SyncQueue {
public:
    void Push(const T& e) {
        std::lock_guard< std::mutex > guard(mutex_);
        queue_.push_front(e);
        cond_.notify_one(); //notify
    }
    T Pop() {
        std::unique_lock< std::mutex > lock(mutex_);
        //stop and wait for notification if condition is false;
        //continue otherwise
        cond_.wait(lock, [this]{return this->QueueFilled();});
        T e = queue_.back();
        queue_.pop_back();
        return e;
    }
friend class Executor; //to allow calls to Clear
private:
    //Return true if queue is not empty
    bool QueueFilled() const { return !queue_.empty(); }
    void Clear() { queue_.clear(); }
private:
    std::deque< T > queue_;
    std::mutex mutex_;
    std::condition_variable cond_;
};
~~~~~~~~~

## Going further

If you want to play with the code and extend it you might
consider:

1. implementing a *Pause* method to pause (not stop) and resume execution
2. specify task dependencies through e.g. an overloaded operator that also
   accepts a future
3. or `shared_future` returned by another task
4. priorities: use a priority queue or multiple queues

*New update*: [Coroutines
(C++20)](https://en.cppreference.com/w/cpp/language/coroutines)
