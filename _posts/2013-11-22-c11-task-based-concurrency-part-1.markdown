---
layout: post
title: C++11 & task based concurrency - part 1
categories: [C++, C++11, Concurrency, Tasks]
permalink: /c11-task-based-concurrency-part-1/
---

# C++11 & task based concurrency - part 1

Task-based concurrency refers to the action of scheduling a number of tasks for
execution and have a number of execution paths (threads) run concurrently to
execute the tasks.

Task-based concurrency is not built into C++11 implementations since it is not
part of the standard which I believe makes a lot of sense: I do not think it is
possible to easily specify a task-based solution generic enough which makes
sense to "standardize".

I'm personally never happy with the various solutions offered by e.g. TBB or
OpenMP, since I want/need to deal with things like:

* dynamic parallelism (change the number of parallel executing threads as you go)
* dynamic task mapping: change the number of tasks assigned to a signle thread at
  run-time
* logging: have threads log the parameters and results passed to them
  automatically without having to explicitly write logging code in the thread body
* hybrid architectures: need to synchronize CPU threads with parallel execution on
  accelerators
* shared cache/memoization: what if I want to intercept the
  parameters passed to a callable object scheduled for execution, use the
  parameters as a key and return a pre-computed value if available ?
* parallelization strategy:
    * single shared task queue
    * multiple shared task queues load balanced with round robin 
    * one data queue per thread with work stealing
    ...

As you will see in the second part of this post where I show a possible
implementation for a task executor, creating your own solution with the new
C++11 facilities is actually pretty straightforward.


### A crash course on C++ 11 concurrency

I am introducing here the bare minimum amount of concepts required to understand
the second part the article, a short list of resources useful for learning about
parallel programming with C++ is presented at the end of this post.

## Let's do it together

When you need to solve a problem through computation on machines with multiple
processing units (cores) it might be useful to split execution into multiple
tasks to be executed in parallel to cooperatively carry on the computation.

One way of having multiple concurrent execution paths is to create threads.

Threads are concurrent execution paths created from within a process (a computer
program being executed); the process itself runs inside a thread which is
created when the process starts.

Tasks can be mapped to threads:

one to one: one task is executed by exactly one thread many to one: multiple
tasks are executed sequentially by a single thread (1) implements thread-level
concurrency, directly supported by C++ 11 and (2) implements task-based
concurrency, not supported by C++ 11.

Note however that task-based concurrency is anyway supported at the operating
system level: if you create more active threads than CPU cores the operating
system will need to have multiple logical threads (created by user code)
executed by a single hardware thread. And the operating system might indeed be
very good at that by the way, so in some cases you might decide to just map one
task to a single thread regardless of the number of threads you create.

## C++11 threads

In C++ 11 threads are created through the std::thread object which accepts in
its constructor the callable object (function, function object...) to execute
together with the parameters to pass to it.

In the rest of the article I use the term parent thread to identify a thread
which creates other threads and the term child thread to describe threads
created by a parent thread.

In order to retrieve the results computed by child threads we need the child
threads to write data into a buffer shared with the parent thread.

The parent thread will then have to wait until all of its child threads join the
main thread, i.e. finish execution, and then read the results from the shared
buffer(s).

The solution works provided the threads do indeed finish execution but what if
the threads retrieve and process data from a queue in a loop ?

In this case you need a way to be notified each time data is available while the
threads are still running; also, when you read the data you have to somehow
protect the buffer you read from so that while a thread reads from the memory
buffer other thread cannot access it for writing.

One way of dealing with this scenario is to wait until a result is available
then lock somehow the memory region you want to read from and perform the read
operation; in this case it is important to treat the wait-read sequence as an
atomic operation so that when you are notified that data is available the memory
region is already locked; the flow looks like:

1. start thread
2. wait and acquire lock when notified
3. read data
4. release lock

Provided you implement __correctly__ all the required logic you still have one
problem: exceptions.

If a thread throws an exception how do you forward it to the parent thread ?

You can think of using one shared buffer per thread to store exceptions, then
check if an exception was raised once you are notified by the child thread.

Implementing also the exception handling part adds additional complexity to an
already complex problem, so let's look at another solution which happens to be
readily available in C++ 11

## Promise me you'll remember

Another way to pass results from child threads to the parent thread is through
some form of object which acts as a messenger between the child (created) thread
and the parent (creator) thread.

The child thread tells the messenger to dispatch the result to the parent thread
once the result is ready.

The parent thread retrieves the result from a mailbox provided by the messenger
object at the messenger creation time, in case the result is not available the
parent thread is automatically put into the wait state until the message is
delivered to the mailbox.

*__Note__*: Here we discuss messages sent by the created thread to its creator
but the messanger used to send messages can be passed to any thread waiting for
the result returned from another thread.

In pseudo-code:

~~~~~~~~~cpp
//Search two halves of the same text concurrently for a specific
//word and report the index of such word if found or a null/invalid value.

Search(start, end, messenger) {
...

  messenger.Send(index_of_found_text);
}
...
Messenger messenger1;
Messenger messenger2;
Mailbox mailbox1 = messenger1.mailbox();
Mailbox mailbox2 = messenger2.mailbox();

Thread search_1st_half = Thread(Search,
                                start,
                                start + half_text_length
                                messenger1);
Thread search_2nd_half = Thread(Search,
                                start + half_text_length,
                                start + full_text_length,
                                messenger2);

Index text_index_1st_half = mailbox1.Recv();
Index text_index_2nd_half = mailbox2.Recv();

if(valid(text_index_1st_half)) report(valid_index_1st_half);
else if(valid(text_index_2nd_half)) 
    report(valid_index_2nd_half + half_text_length);
else report("NOT FOUND");
...
~~~~~~~~~

For the above code to work, you need two objects, Messenger and Mailbox, because
they are accessed from two separate threads: Messenger is accessed by the child
thread and Mailbox by the parent thread.

Now, what if on top of the shown functionality the Messenger object also had a
way to forward exceptions from child to parent thread ?

That would be great and that's exactly what C++ 11 allows you to do.

The messenger is an `std::promise`, which represents a promise to perform a
computation and the mailbox is an `std::future` which represents a value that
will be available in the future.

Nice, but how do we signal an error without throwing an exception ?

You could return a composite data type with two members: 

* a valid/invalid flag
* the actual return value if valid flag is `true`

C++14 was supposed to support such a case through the `std::optional` type but
it was voted out of the standard and moved to a separate technical report;
`boost::optional` is available though.

## Task-based concurrency - take 1

We have seen what threads look like, now let's look at a simple way to perform
task based concurrency.

One easy way to have a pre-defined number of threads execute tasks, where the
number of tasks is greater than the number of threads, is to launch
the same number of threads multiple times and wait for completion until all
tasks have been executed.

In the following example a task is represented by the execution of the function
*f*: We execute the function *f* `num_tasks` times with `num_threads` threads
running concurrently.

~~~~~~~~~cpp
for(int t = 0; t != num_tasks; t += num_threads) {
    ThreadArray ta;
    for(int i = 0; i != num_threads; ++i) {
        t.push_back(Thread(f, data, t + i)); //calls f(data t + i)
    }
    //join all threads before continuing to the next batch of tasks
    barrier(ta.begin(), ta.end()); //call .join() method on each thread
}
~~~~~~~~~

This works, but all the threads execute only once per task and new threads
are created at each iteration.

## Stop and go

To have a fixed number of pre-created threads perform all the computation you
need to:

1. create the threads
2. put the threads in a wait state
3. add tasks to a shared queue and wake up the threads
4. have the threads retrieve and execute tasks in the queue
5. go to (2) 

The queue itself can be a single queue shared by all the threads.

You also need to have a way to send a termination request to the threads, should
they need to be terminated before the process exists, which they always do
unless you want to exit the process with a number of threads still in execution
which results in a call to abort().

If all you want is to get rid of the error just override the default termination
handler with your own:

~~~~~~~~~cpp
void quiet_termination() { exit(0); }

int main(int argc, char** argv) {
    std::set_terminate(quiet_termination);
    ...
}
~~~~~~~~~

One way of gracefully terminating threads running a wait-get-execute loop is to
insert data (e.g. an empty task) recognized as a termination condition in the
execution queue.

## References

*updated February 28 2020*

Disclaimer: the first reference is to the official ISO C++ 11
standard document, if you can, do have a look at it; all the threading feautures
are well described and in some cases easier to understand than on various online
resources, and sample code is provided as well.

* [ISO, IEC 14882-2011 C++](http://isocpp.org/std/the-standard), chapter 30:
  `http://isocpp.org/std/the-standard`
* [cppreference](http://en.cppreference.com/w/)
* Anthony Williams, *C++ Concurrency in Action* - Manning Publications
* [Bartoz Milewzki's blog](http://bartoszmilewski.com/category/concurrency/)
* [Bartoz Milewzki's C++ concurrency
  videos](http://www.youtube.com/playlist?list=PL1835A90FC78FF8BE&feature=c4-feed-u)
* [Herb Sutter blog on concurrency](http://herbsutter.com/category/concurrency)
* [Scott Meyers on std::future](http://scottmeyers.blogspot.com/search?q=future)
* [Scott Meyers'Effective
  C++11/14](https://www.amazon.com.au/Effective-Modern-C-Scott-Meyers/dp/1491903996)
