---
layout: post
title:  Concurrent access
categories: [C++, C++11, multithreading, synchronization]
permalink: /concurrent-access/
---

[_updated March 2020_]

In the C++ & task based concurrency posts
[1]({{site.baseurl}}/c11-task-based-concurrency-part-1) and
[2]({{site.baseurl}}/c11-task-based-concurrency-part-2) I show how to
write your own task based executor to execute any number of tasks with
a user-defined number of OS threads.

In this post I am showing a `ConcurrentAcces` type which works together
with an `Executor` instance to serialize access to a shared resource.

The technique I implement is the one described by Herb Sutter in his
talk C++ concurrency given at the C++ and Beyond 2012 conference with
one major difference, one minor difference and one addition:

1. task-based concurrency with a pre-defined number of threads instead of
creating a new thread each time an instance of `concurrent<T>` is
created;
2. no mutable: with the mutable modifier standard C++ reference
types are not supported. However `std::reference_wrapper` does work;
3. full implementation with support for `void` return type which is missing
from Herb's talk.

The basic idea is to access a resource with a
transaction-like pattern:

```cpp
...
    SharedResource sr;
    BEGIN_TRANSACTION(sr);
    sr.SetData("data");
    END_TRANSACTION;
```

## Shared access

In cases where multiple threads need to access a shared resource you
have two main options to synchronize access (i.e. only one OS thread
at a time can access the resource):

1. blocking: issue a lock method call on a mutex object and wait until
the access is granted; 
2. non-blocking: add a command to be executed into
a queue; the command will be executed in a separate thread while
execution is resumed just after the command is added to the queue.

The focus of this post is on option (2), after creating a synchronized
queue I implement a simple wrapper class that wraps the object to
access and uses the Executor class to start a task which pops and
executes actions from the command queue.

__NOTE #1__: shared access is nice and easy when you are synchronizing access to a single object which does not require itself access to other shared resources, for more complex scenarios I find much more productive to use other techniques involving message passing with e.g. ZeroMQ.


## Synchronized queue

The synchronized queue implementation uses a lock to synchronize access to its elements and a condition variable to block until the queue is not empty.

Each time a Pop operation is requested the requesting thread waits until the queue contains at least one element then extracts one element from the queue and returns it.

Each time an object is added to the queue through a Push operation a notification signal is sent so that if there is a client thread waiting for data it can wake up and resume processing.

```cpp
    template < typename T >
    class SyncQueue {
    public:
        void Push(const T& e) {
            //simple scoped lock: acquire mutex in constructor,
            //release in destructor
            std::lock_guard< std::mutex > guard(mutex_);
            queue_.push_front(e);
            cond_.notify_one(); //notify
            //lock released on exit
        }
    T Pop() {
        //cannot use simple scoped lock here because lock passed to
        //wait must be able to acquire and release the mutex
        std::unique_lock< std::mutex > lock(mutex_);
        //stop and wait for notification if condition is false;
        //continue otherwise
        //condition variable: release lock and and wait until
        //notifcation happens; if condition is true lock
        //and continue, wait otherwise
        cond_.wait(lock, [this]{return !queue_.empty();});
        T e = queue_.back();
        queue_.pop_back();
        return e;
        //lock released on exit
    }
    private:
        std::deque< T > queue_;
std::mutex mutex_;
        std::condition_variable cond_;
    };
```


## ConcurrentAccess type

Objects requiring synchronized access, i.e. only a single thread at a
time is allowed to access the object, are wrapped with instances of
the `ConcurrentAccess` type.

Access to the wrapped object takes place through actions added to an
execution queue.

Actions that operate on the wrapped object are extracted from the
queue and executed from within a thread different from the one that
adds the action into the queue.

Access to the wrapped object happens by invoking `ConcurrentAccess`
instances as functor objects, passing in a action to execute in the
form of a callable entity (function, function object/functor or
lambda).

The actual code used to access the object looks like a transaction
which ensures that while the transaction takes place no other action
can interfere with the operations being executed from within the
ConcurrentAccess call operator:

```cpp
 1     msg = "start\n";
 2     //wrap *reference* to 'msg'
 3     ConcurrentAccess< string& > text(msg, ...);
 4     ...
 5     //per-task:
 6     ...
 7     const thread::id calling_thread = this_thread::get_id();
 8     text([=](string& s ) {
 9         ostringstream oss;
10         oss << s << "added by thread " + calling_thread << endl;
11         s = oss.str();
12     });
13     ...

```

In the above code I am wrapping a reference to a string with a
_ConcurrentAccess_ object, then each thread that needs to access the
string does it on line 8 by invoking the _ConcurrentAccess_ instance as
a functor passing in a callable object (an unnamed lambda function in
this case) which receives a reference to the string to modify.

The code in lines 9-11 is executed in a thread different from the
thread that invokes text(...) and guaranteed to have exclusive access
to the msg string variable.

__NOTE #2__: it all works out if there is no code that tries to access the
wrapped object directly without going through the _ConcurrentAccess_
instance. If possible do not wrap references but do wrap any actual
object instance that requires synchronized access with a
_ConcurrentAccess_ instance. If you want to avoid copying data into the
_ConcurrentAccess_ data member you can always implement a
_ConcurrentAccess_ constructor that accepts the same parameters as the
wrapped type constructor and build the wrapped type in place, or use a
move constructor.

__NOTE #3__: in order for this to work the resources that are being
accessed by the scheduled actions must be accessible from a thread
different from the one that schedules the action for execution,
unfortunately this is not the case in many situations, e.g.: when
dealing with GPUs for both graphics and compute operations, access to
the GPU context must usually happen from within the same thread that
creates the context itself (as of 2014).

The `ConcurrentAccess::operator()(...)` method must return an
`std::future` instance used to synchronize and optionally access the
result of each scheduled action.

The `ConcurrentAccess` type must contain a data member to either store
or reference the synchronized object and shall also have access to a
synchronized queue where actions are stored for deferred execution.

### Requirements

Now, let's summarize the requirements for a complete implementation of
a _ConcurrentAccess_ wrapper type:

1. on construction: accept a reference to an Executor instance used to
schedule action execution; 
2. expose an _operator()(Action)_ where _Action_ is a callable entity 
which accepts an argument of the same type of the wrapped resource or 
a (constant) reference to the same type as the mapped resource;
3. have _operator()(Action)_ return an `std::future< T >` where `T` is the 
type returned from the invocation of the callable entity or void if 
nothing is returned;
4. forward exceptions to the thread that scheduled a specific action for execution;
5. stop the task executor on destruction of the _ConcurrentAccess_ instance; 
6. make it non-copyable.

### Implementation

#### 0. Data members

```cpp
    template < typename T >
    class ConcurrentAccess {
        T data_; //warning 'mutable' cannot be applied to references
        //keep on retrieving actions from queue until done is 'true'
        bool done_ = false;
        //action queue
        SyncQueue< std::function< void () > > queue_;
        //future returned by executor when pop-execute
        //loop is added to executor queue
        std::future< void > f_;
```

#### Construction

On construction simply add to the execution queue a function that
extracts and executes actions from the action queue in a loop which is
terminated when the data member `done_` is set to `true`.

Note that in order to wait for the extract-execute loop to finish
after requesting termination, you need to store the future instance
returned by the executor for later use.

```cpp
ConcurrentAccess(T data, Executor& e) : data_(data),
f_(e([=]{
    while(!done_) queue_.Pop()(); //extract and execute
   }))
{}
```

`Executor::operator(...)` accepts a functor and returns a future
which is then moved into the `f_` data member.

#### 2.  3. 4.  Call operator and exception forwarding

Add an unnamed function (lambda) to the action queue and return a
future to synchronize with the action at a later time.

The lambda function added to the queue calls the function object
passed to the call operator and uses a promise to return a value or
raise an exception.

The promise used by the lambda function is created in the call
operator body and a pointer to it is copied into the lambda context
(closure). Using a shared pointer type guarantees that the promise is
automatically destroyed upon exit from the lambda function body.

Exceptions are forwarded to the calling thread (the one that scheduled
the action for execution) by wrapping the execution of the passed
callable object with a try/catch block and re-throwing the exception
in the _catch{}_ block.

```cpp
    template < typename F >
    auto operator()(F f) -> std::future< decltype(f(data_)) > {
        using R = decltype(f(data_));
        auto p = std::make_shared< std::promise< R > >(std::promise< R >());
        auto ft = p->get_future();
        queue_.Push([=]{
            try {
                p->set_value(f(data_));
            } catch(...) {
                p->set_exception(std::current_exception());
            }
        });
        return ft;
    }
```

#### 5. Stop tasks on destruction

The extract-execute loop must be stopped when the _ConcurrentAccess_
instance is destroyed.

The execution loop, started in the constructor, stops when the the `done_` 
variable is set to `true`.

The check for done_ takes place after an action has been extracted and
executed, if you want the loop to terminate you therefore have to add
an action to the queue that sets done_ to true so that after such
action is executed the _while(...)_ condition evaluates to `false` and the
loop terminates.

```cpp
    ~ConcurrentAccess() {
        queue_.Push([=]{done_ = true;});
        f_.wait();
    }
```

#### 6. Copy and move
Copy and move implementations really depend on your specific cases, I
hereby decided to disable both move and copy because it is not clear
to me why and how you would need to copy a ConcurrentAccess object and
what to do with its wrapped data, if I want other functions to
interact with the wrapped data in a synchronized way I can simply pass
around a reference, _std::reference_wrapper_, _*_ptr_ to a
_ConcurrentAccess instance_.

```cpp
    ConcurrentAccess() = delete;
    ConcurrentAccess(const ConcurrentAccess&) = delete;
    ConcurrentAccess(ConcurrentAccess&&) = delete;
```

Also note that the only safe way of copying and moving the wrapped
object is through an action stored into the command queue, so if you
want to implement copy and move constructors you will have to use the
action queue to perform the actual copy or move operation e.g.

```cpp
    ConcurrentAccess::ConcurrentAccess(const ConcurrentAccess& ca) {
        ...
        //the reference passed to the functor passed to
        //ConcurrentAccess::operator() points to the
        //wrapped object
        std::future copy = ca([](const T& obj){return obj;});
        data_ = copy.get();
        ...
    }
```

## Void return type

The current implementation does not support actions that do not return
a value, to support such actions we need the call operator to invoke
different overloaded methods depending on the return type of the
passed action:

```cpp
    template < typename F >
    auto operator()(F&& f)
    -> std::future< typename std::result_of< F(T) >::type > {
        using R = typename std::result_of< F(T) >::type;
        return Invoke(std::forward< F >(f), typename Void< R >::type());
    }
```

The `Void` struct declares its internal type member as a type that
depends on the return type of the passed action:

* void return type: `Void::type` is of type `VoidType`;
* non-void return type: `Void::type` is of type `NonVoidType`.

The call operator is then able to invoke different `Invoke` overloads
depending on the return type of the user-specified callable object:

```cpp
    template < typename F >
    auto Invoke(F&& f, const NonVoidType& )
    -> std::future< typename std::result_of< F(T) >::type > {
        using R = typename std::result_of< F(T) >::type;
        auto p = std::make_shared< std::promise< R > >(std::promise< R >());
        auto ft = p->get_future();
        queue_.Push([=]() {
            try {
                p->set_value(f(data_));
            } catch(...) {
                p->set_exception(std::current_exception());
            }
        });
        return ft;
    }

    template < typename F >
    std::future< void > Invoke(F&& f, const VoidType& ) {
        auto p =
            std::make_shared< std::promise< void > >(std::promise< void >());
        auto ft = p->get_future();
        queue_.Push([=]() {
            try {
                f(data_);
                p->set_value();
            } catch(...) {
                p->set_exception(std::current_exception());
            }
        });
        return ft;
    }
```

## Final remarks

### Thread safety

The solution works if you can afford to have the wrapped objects
"touched" from threads different from the one that constructs the
instance wrapped with ConcurrentAccess. If you cannot and you need to
create and access the object from the same thread, you can still use
this solution but need to change the Executor class to support pinning
a task to a specific thread. There are a few ways to implement
task-thread pinning including:

* have one execution queue per thread and allow the client code to
  either specify which thread shall execute a task or pick one
  randomly or in a round-robin fashion;
* use a single queue and tag tasks with a thread id, when a task is
  extracted from the queue the thread id is compared to the current
  thread id: if they match the task is executed, if not it is put back
  into the queue, possibly at the front of the queue. A special tag to
  indicate that a task can be executed by any thread should be
  provided as well.

### A generic pattern

The techniques outlined in this post are useful in contexts other than
concurrency.

In general any case in which you need to perform operations before
and/or after accessing an object can make use of a transaction-like
access strategy, I personally started experimenting with this pattern
for cases where the data to use reside in a memory space not directly
accessible by the code.

Cases include mapped resources for OpenGL, Direct3D, CUDA and OpenCL
or memory regions physically residing on different nodes in a
computing cluster accessible through e.g. one-sided MPI or RDMA.

Example: wrapping an OpenCL memory object.

```cpp
    class CLByteBuffer {
        cl_command_queue cmdQueue_;
        cl_mem clbuffer_;
        size_t bufferSize_;
    ...
    public:
    ...
        template < F >
        void operator(F&& f) {
            cl_int err;
            unsigned char* data =
            clEnqueueMapBuffer(cmdQueue_,
                clbuffer_,
                CL_TRUE,
                CL_MAP_WRITE,
                0,
                bufferSize_,
                0, NULL, NULL, &err);
            if(err != CL_SUCCESS) throw(CLException(err));
            f(data, bufferSize_);
            err =
                clEnqueueUnmapMemObject(cmdQueue_,
                    clbuffer_,
                    data,
                    0, NULL, NULL);
            if(err != CL_SUCCESS) throw(CLException(err));
        }
    ...
    };

    ...
    CLByteBuffer bb;
    ...
    bb([=](unsigned char* b, size_t size) {
        ReadDataFromFile(filename, b, size);
    });
```

You could even think of using this strategy for network and file I/O:
wrap a socket/file and perform all the read/write through a
transaction which adds data to a buffer and flushes the buffer after
the lambda function returns.

```cpp
    ClientSocket socket("192.168.1.1");
    SocketStreamWrapper s(socket);
    ...
    s([](ostream& os) {
        os << "length is " << (3 inch);
    });
    ...
```

The `SocketStreamWrapper::operator()(F)` call operator can be
implemented as a method which uses internally an instance of
`std::ostringstream` which is filled by the user-specified lambda.
After the lambda returns the string is extracted from the
`std::ostringstream` instance and sent through the wrapped `ClientSocket`
object.

## Performance and storage considerations

When passing a lambda function to the object wrapper instance you
should be careful about which data you capture.

Lambda functions with a non-empty capture specification (i.e. non
empty [] expression) are converted to function objects holding a copy
or reference to the resources specified in the [] statement.

In case you have in scope objects that are expensive to copy do not
use = to capture everything but either cherry-pick the objects to
capture or just use & to capture everything by reference.

## Resources

[Source code](https://raw.github.com/ugovaretto/cpp11-scratch/master/training/task-based-executor-concurrent-generic.cpp) 
(includes an `Executor` implementation)

[C++ concurrency talk by Herb Sutter](http://channel9.msdn.com/Shows/Going+Deep/C-and-Beyond-2012-Herb-Sutter-Concurrency-and-Parallelism),
concurrent access is discussed starting at `00:30:55`

