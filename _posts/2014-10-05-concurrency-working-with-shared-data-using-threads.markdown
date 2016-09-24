---
layout: post
title:  "Concurrency: Working with shared data using threads"
date:   2014-10-05 06:32:0 +0530
categories: jekyll update
---

In continuation with the [last week’s
experiment](http://127.0.0.1/rants/) on protecting shared data in a
concurrent system, let’s look at how we can make it work using C++ and
threads.

Here’s a brief recap: We’re building a Airport scenario, where we have a
single status board screen. The status board is simultaneously being
edited by multiple airliners. The status board can be read by any
passenger at any given moment for any flight status based on the flight
number.

Let’s build the concept within a single-threaded environment:

``` {.brush: .cpp; .title: .; .notranslate title=""}

class StatusBoard {

    std::forward_list<int> statusBoard;

    

public:

    void AddFlight()

    {

        int newFlight = rand()%MAX_FLIGHTS;

        statusBoard.push_front(newFlight);

    }

    

    bool FindFlight(const int flightNum) const

    {

        return (std::find(statusBoard.begin(), 

                          statusBoard.end(), 

                          flightNum) != statusBoard.end());

    }

};



StatusBoard statusBoard;



void StatusWriter()

{

    debugCount.write++;

    statusBoard.AddFlight();

}



bool StatusReader(const int searchFlightNum)

{

    debugCount.read++;

    return statusBoard.FindFlight(searchFlightNum);

}



void SerialLoop()

{

    while (!StatusReader(SEARCH_FLIGHT)) {

        StatusWriter();

    }

}
```

On my machine, it outputs:

``` {.brush: .plain; .title: .; .notranslate title=""}

Flight found after 26 writes and 27 reads.
```

Now let’s begin with making this implementation multithreaded.

``` {.brush: .cpp; .title: .; .notranslate title=""}

void StatusWriter()

{

    std::this_thread::sleep_for(std::chrono::seconds(1));

    

    debugCount.write++;

    statusBoard.AddFlight();

}



void ParallelLoop()

{

    while (!std::async(std::launch::async, StatusReader, SEARCH_FLIGHT).get()) {

        /* write operation */

        std::thread writeOp(StatusWriter);

        writeOp.detach();

    }

}
```

The good thing about C++ is that much of the code looks almost similar
to the single threaded code and that is due to the way the thread
library has been designed. At first glance you won’t be even able to
tell where the threading code actually is. Here we’re invoking two
operations on two threads at each iteration of the loop.

First we have a read operation

``` {.brush: .cpp; .title: .; .notranslate title=""}

while (!std::async(std::launch::async, StatusReader, SEARCH_FLIGHT).get()) {
```

This makes use of std::future to invoke the StatusReader function on a
async thread. The get() call at the end actually start the process. We
shall talk about futures and promises in some future experiment. In this
experiment it would be suffice to understand that a call to std::async()
starts the StatusReader() function in a concurrent thread that magically
returns back the result with the get() to the caller thread.

Another is simply spawning of a std::thread using the direct use with
detach that we’re already familiar of from [our previous
experiment](http://127.0.0.1/rants/?p=1121 "Concurrency: Spawning Independent Tasks").

Here we just start the StatusWriter function on a new thread every time
the loop iterates.

``` {.brush: .cpp; .title: .; .notranslate title=""}

/* write operation */

std::thread writeOp(StatusWriter);

writeOp.detach();
```

On my machine, when the code doesn’t crashes, it prints:

``` {.brush: .plain; .title: .; .notranslate title=""}

Flight found after 26 writes and 1614 reads.
```

Obviously, this code isn’t thread-safe. We’re reading and writing to the
same forward list from more than one threads, and this code is supposed
to crash a lot. So, next step let’s make it thread safe using a mutex.

A mutex is the basic object that is used to provide mutual exclusivity
to a portion of code. The mutex in C++ are used to lock a shared
resource.

First of all we start with a std::mutex object. This object guarantees
that the code is locked for a single thread use at a time. We can use
the lock() and unlock() member functions on std::mutex object like:

``` {.brush: .cpp; .title: .; .notranslate title=""}

void foo()

{

    std::mutex mutex;

    

    mutex.lock();

    

    /* 

     * some resource locking code here

     */

    

    mutex.unlock();

}
```

If you’ve used C++ long enough, you’ll know that this code is prone to
all sort of errors. In many cases the foo() can exit before the unlock()
gets executed. This could be due to any early returns that you have
added to the function. Or, in a more likely scenario, your code does
something illegal and the foo() throws some exception. Even if foo() is
very robust, it could call some another function which isn’t that
robust.

If you’re an old time C++ user, you must know that to solve this problem
space we often use
[RAII](http://en.wikipedia.org/wiki/Resource_Acquisition_Is_Initialization)
idiom. Good for you, the standard thread library already provides this
for you in form of a std::lock\_guard. The std::lock\_guard is designed
to work with any lock like object.

Using std::mutex and std::lock\_guard our code is now:

``` {.brush: .cpp; .title: .; .notranslate title=""}

class StatusBoard {

    std::forward_list<int> statusBoard;

    mutable std::mutex mutex;

    

public:

    void AddFlight()

    {

        std::lock_guard<std::mutex> lock(mutex);

        

        int newFlight = rand()%MAX_FLIGHTS;

        statusBoard.push_front(newFlight);

    }

    

    bool FindFlight(const int flightNum) const

    {

        std::lock_guard<std::mutex> lock(mutex);

        

        return (std::find(statusBoard.begin(), statusBoard.end(), flightNum) != statusBoard.end());

    }

};
```

The mutable keyword is used as we’re using the mutex in a const member
function FindFlight(). On my machine it executes as:

``` {.brush: .plain; .title: .; .notranslate title=""}

Flight found after 10 writes and 1591 reads.
```

Sometime, the code also throws the following exception after exiting
from main:

``` {.brush: .plain; .title: .; .notranslate title=""}

libc++abi.dylib: terminating with uncaught exception of type std::__1::system_error: mutex lock failed: Invalid argument
```

This is due to the fact that we’ve explicitly added the thread delay for
debugging and some of out async threads are executing as detached and
might not be over before the main exits. I don’t think this is going to
be a problem in any real code.

So, there, we’ve a basic thread-safe code. This is not the best
implementation possible. There are many things to look out for. Like,
we’ve to take care of not pass out any references or pointers of shared
data, otherwise the code might become unsafe again. Always remember that
at the low level all threads share the same address space, and whenever
we’ve a shared resource without a lock, we have a race condition. This
is more of the design error from the programmer’s perspective.

As always this code is available online at
[github.com/chunkyguy/ConcurrencyExperiments](https://github.com/chunkyguy/ConcurrencyExperiments).
Check it out and have fun experimenting on your own.
