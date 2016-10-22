---
layout: post
title:  "Concurrency: Spawning Independent Tasks"
date:   2014-09-21 11:24:00 +0530
categories: concurrency
---

Concurrency: Spawning Independent Tasks
----------------------------------------

First step in getting with concurrency is spawning tasks in background
thread.

Lets say we’re building a chat application where a number of users chat
in a common room. In this sort of scenario every user is probably
chatting at the same instance. For sake of simplicity let’s say each
user can only have a single character username. And obviously such a
constraint is definitely going to outrage the users. So, pissed off as
they are, they’re just printing their usernames over and over again.

To support such a functionality, we might need a function like such:

``` {.brush: .cpp; .title: .; .notranslate title=""}

void PrintUsername(const char uname, const int count)

{

    for (int i = 0; i < count; ++i) {

        std::cout << uname << ' ';

    }

    std::cout << std::endl;

}
```

In a single threaded environment, whatever user invokes this method gets
the control of the CPU and their username is going to get printed until
the loop expires.

If there are two users in the chat room with usernames ‘X’ and ‘Y’, each
wants to print their username 10 times. The calling code might look
something like:

``` {.brush: .cpp; .title: .; .notranslate title=""}



void SingleThreadRoom()

{

    PrintUsername('X', 10);

    PrintUsername('Y', 10);

}
```

The output is as expected:

``` {.brush: .plain; .title: .; .notranslate title=""}

X X X X X X X X X X 

Y Y Y Y Y Y Y Y Y Y 
```

Now, to make the app slightly better, let’s give both the users equal
weights. Each user should be able to perform the same task in their own
thread. Simplest way to spawn a new thread is by calling join()

``` {.brush: .cpp; .title: .; .notranslate title=""}



void MultiThreadRoom()

{

    std::thread threadX(PrintUsername, 'X', 10);

    std::thread threadY(PrintUsername, 'Y', 10);

    

    threadX.join();

    threadY.join();

}
```

On my machine this outputs as:

``` {.brush: .plain; .title: .; .notranslate title=""}

YX  YX  YX  YX  YX  YX  YX  YX  YX  YX  


```

Calling join() on a thread attaches the thread to the calling thread.
When the app launches we already get the main thread, and since we’re
calling join() on the main thread, the two spawned threads get joined to
the main thread.

If we need a thread for some task that is of kind fire-and-forget, we
can call detach(). But, if we call detach() we’ve no control of the
thread lifetime anymore. It’s something like the main thread doesn’t
cares anymore.

For a program like:

``` {.brush: .cpp; .title: .; .notranslate title=""}

void MultiThreadRoomDontCare()

{

    std::thread threadX(PrintUsername, 'X', 10);

    std::thread threadY(PrintUsername, 'Y', 10);

    

    threadX.detach();

    threadY.detach();

}



int main()

{

    MultiThreadRoomDontCare();

    std::cout << "MainThread: Goodbye" << std::endl;

    return 0;

}
```

The output on my machine is:

``` {.brush: .plain; .title: .; .notranslate title=""}

MainThread: Goodbye

XY  XY
```

As you can see the main thread doesn’t care if it’s child thread are
still not finished.

Getting over to Swift, we don’t have to manually track the threads, but
think in more higher terms as tasks and queues.

First problem we face when trying out async code in Playground is that
the NSRunLoop isn’t running in Playground as it is with out native apps.
The solution is to import the XCPlayground and call
XCPSetExecutionShouldContinueIndefinitely() method. [All thanks to the
internet](http://stackoverflow.com/a/24066317/286094).

Here’s a test code that works:

``` {.brush: .cpp; .title: .; .notranslate title=""}

import UIKit

import XCPlayground



XCPSetExecutionShouldContinueIndefinitely(continueIndefinitely: true)



NSOperationQueue.mainQueue().addOperation(NSBlockOperation(block: { () -> Void in

    println("Hello World!!")

}))



var done = "Done"
```

Since, we’re talking in terms of queues and tasks, we have to change the
way of thinking. Now, we don’t care about threads anymore, only tasks.
The task is to print the username. This task is going to be submitted to
the queue and the queue internally shall manage the threads.

A code like:

``` {.brush: .cpp; .title: .; .notranslate title=""}

func printUsername(uname:String)

{

        println(uname)

}



let chatQueue:NSOperationQueue = NSOperationQueue()



func multiThreadRoom()

{

    for (var i = 0; i < 5; ++i) {

        chatQueue.addOperation(NSBlockOperation(block: { () -> Void in

            printUsername("X")

        }))

        

        chatQueue.addOperation(NSBlockOperation(block: { () -> Void in

            printUsername("Y")

        }))

    }

}



multiThreadRoom()



println("Main Thread Quit")
```

Prints on my machine as:

``` {.brush: .plain; .title: .; .notranslate title=""}

X

Y

X

Y

X

Y

X

Y

X

Y

Main Thread Quit
```

This is the very basics of spawning concurrent independent tasks. If all
your requirements are working on independent set of tasks, this is
almost all you need to learn about concurrency to get the job done. I
hope the real world were this simple
![;)](http://127.0.0.1/rants/wp-includes/images/smilies/icon_wink.gif){.wp-smiley}

As always, the entire code is also available online at:
<https://github.com/chunkyguy/ConcurrencyExperiments>
