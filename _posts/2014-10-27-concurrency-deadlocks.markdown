
---
layout: post
title:  "Concurrency: Deadlocks"
date:   2014-10-27 05:54:54 +0530
categories: jekyll update
---

We’ve explored so much with our concurrency experiments and yet there’s
one fundamental topic we haven’t touched so far. Deadlocks. Whenever we
think of concurrency, we can’t help thinking of locks for protecting
shared resources, managing synchronization and what not. But, as Uncle
Ben has said, with great power comes great responsibility, similarly
with great locks comes great deadlocks.

**Deadlocks with multiple locks**

In the classical concurrency theory, deadlocks happen when more than one
thread needs mutual exclusivity over more than one shared resource.
There’s a classic [Dining Philosophers
problem](http://en.wikipedia.org/wiki/Dining_philosophers_problem) that
demonstrates this deadlock situation.

Here’s the copy paste from the [wikipedia
page](http://en.wikipedia.org/wiki/Dining_philosophers_problem):

> Five silent philosophers sit at a round table with bowls of spaghetti.
> Forks are placed between each pair of adjacent philosophers.
>
> Each philosopher must alternately think and eat. However, a
> philosopher can only eat spaghetti when he has both left and right
> forks. Each fork can be held by only one philosopher and so a
> philosopher can use the fork only if it’s not being used by another
> philosopher. After he finishes eating, he needs to put down both forks
> so they become available to others. A philosopher can grab the fork on
> his right or the one on his left as they become available, but can’t
> start eating before getting both of them.

There are many problems hidden inside this one problem, but lets focus
on the situation where every philosopher gets super hungry has acquired
the fork on their left side and waiting for someone to release a fork
before they can begin eating. But since, no philosopher is willing to
release their fork, we have a deadlock situation.

Let’s reduce this problem to just two philosophers dining. Here’s a
simple implementation

``` {.brush: .cpp; .title: .; .notranslate title=""}

#define MAX_FORKS 2



std::mutex fork[MAX_FORKS];



class Philosopher {

private:

    std::string name_;

    int holdForkIndex_;



    void Think()

    {

        std::this_thread::sleep_for(std::chrono::seconds(1));

    }

    

public:

    Philosopher(const std::string &name, const int startForkIndex) :

    name_(name),

    holdForkIndex_(startForkIndex)

    {}



    void Eat()

    {

        std::cout << name_ << ": Begin eating" << std::endl;



        Think();

        

        std::lock_guard<std::mutex> locka(fork[holdForkIndex_]);

        std::cout << name_ << ": Hold fork: " << holdForkIndex_ << std::endl;

        holdForkIndex_ = (holdForkIndex_ + 1) % MAX_FORKS;



        Think();



        std::lock_guard<std::mutex> lockb(fork[holdForkIndex_]);

        std::cout << name_ << ": Hold fork: " << holdForkIndex_ << std::endl;

        holdForkIndex_ = (holdForkIndex_ + 1) % MAX_FORKS;



        // eating

        std::this_thread::sleep_for(std::chrono::microseconds(10));

        

        std::cout << name_ << " End eating" << std::endl;

    }

};



void DiningPhilosophers()

{

    Philosopher s("Socrates", 0);

    Philosopher n("Nietzsche", 1);

    

    std::thread sEat(&Philosopher::Eat, &s);

    std::thread nEat(&Philosopher::Eat, &n);

    

    sEat.join();

    nEat.join();

}



/* this is the main() */

void Deadlocks_main()

{

    DiningPhilosophers();

}
```

We have two philosophers and two forks. In each Eat() function we follow
the order:

1.  Think
2.  Grab a fork
3.  Think
4.  Grab another fork
5.  Eat
6.  Release both forks

Here’s the output on my machine, before the program just hangs:

``` {.brush: .plain; .title: .; .notranslate title=""}

Socrates: Begin eating

Nietzsche: Begin eating

Socrates: Hold fork: 0

Nietzsche: Hold fork: 1
```

As you can see, each philosopher acquires a fork and then just waits
indefinitely. One way to break this deadlock is to use
[std::lock()](http://en.cppreference.com/w/cpp/thread/lock) function.
What this function does is that it provides a functionality to lock one
or more mutexes at once if it can, otherwise it locks nothing.

``` {.brush: .cpp; .title: .; .notranslate title=""}

void Eat()

{

    std::cout << name_ << ": Begin eating" << std::endl;



    int lockAIndex = holdForkIndex_;

    int lockBIndex = (holdForkIndex_ + 1) % MAX_FORKS;

    

    std::lock(fork[lockAIndex], fork[lockBIndex]);

    

    Think();

    

    std::lock_guard<std::mutex> locka(fork[lockAIndex], std::adopt_lock);

    std::cout << name_ << ": Hold fork: " << lockAIndex << std::endl;



    Think();



    std::lock_guard<std::mutex> lockb(fork[lockBIndex], std::adopt_lock);

    std::cout << name_ << ": Hold fork: " << lockBIndex << std::endl;



    // eating

    std::this_thread::sleep_for(std::chrono::microseconds(10));

    

    std::cout << name_ << " End eating" << std::endl;

}
```

First, the philosopher tries to lock both the forks. If successful, then
they proceed further to eat, otherwise they just wait.

Notice two things here, first of all we’re not using any explicit loops
to lock-test-release mutex. All of that is handled by std::lock().
Second, we’re providing the extra parameter std::adopt\_lock as a second
argument to
[std::lock\_guard](http://en.cppreference.com/w/cpp/thread/lock_guard).
This gives a hint to std::lock\_guard to not lock the mutex at the
construction, as we’re using the std::lock() for locking mutexes. The
only role of std::lock\_guard() here is to guarantee an unlock when the
Eat() goes out of scope.

Here’s the output for the above code:

``` {.brush: .plain; .title: .; .notranslate title=""}

Socrates: Begin eating

Nietzsche: Begin eating

Socrates: Hold fork: 0

Socrates: Hold fork: 1

Socrates End eating

Nietzsche: Hold fork: 1

Nietzsche: Hold fork: 0

Nietzsche End eating
```

**Deadlocks with single lock**

Deadlocks happen more often when more than one mutex is involved. But,
that doesn’t means that deadlocks can’t happen with a single mutex. You
must be thinking, how much stupid one has to be just using a single
mutex and still deadlocking the code. Picture this scenario:

> Homer Simpson works at Sector 7-G of a high security Nuclear Power
> Plant. To avoid the unproductive water-cooler chit-chat, their boss
> Mr. Burns has placed the water cooler inside a highly protected cabin
> which is protected by a passcode, allowing only single employee in at
> a time. Also, to keep track of how much water is being consumed by
> each employee, Mr. Burns has equipped the water cooler with mechanism
> that every employee needs to swipe their card in order to use it.
>
> Homer is thirsty, so he decides to get some water. He enters the
> passcode and is inside the protected chamber. Once there, he realizes
> he’s left his card at his desk. Being lazy, he decides to call his
> friend Lenny to fetch his card. Lenny goes to Homer’s office, grabs
> his card and goes straight to the water cooler chamber, only to find
> it locked from the inside.
>
> Now, inside Homer is waiting for Lenny to get his card. While, outside
> Lenny is waiting for the door to get unlocked. And we have a deadlock
> situation.

Here’s a representation of above problem in code

``` {.brush: .cpp; .title: .; .notranslate title=""}

std::mutex m;



void SwipeCard()

{

    std::cout << "Card swiping ... " << std::endl;

    std::lock_guard<std::mutex> waterCoolerLock(m);

    std::cout << "Card swiped" << std::endl;

}



void GetWater()

{

    std::cout << "Chamber unlocking ... " << std::endl;

    std::lock_guard<std::mutex> waterCoolerLock(m);

    std::cout << "Chamber unlocked" << std::endl;



    SwipeCard();



    std::cout << "Water pouring" << std::endl;

}



void Sector7GSituation()

{

    GetWater();

}
```

And here’s the output before the program hangs:

``` {.brush: .plain; .title: .; .notranslate title=""}

Chamber unlocking ... 

Chamber unlocked

Card swiping ... 
```

The moral of the story is, calling of user code after acquiring a lock
should be handled with uttermost care. For example, in above code
calling SwipeCard() after acquiring a lock is a big clue that there are
some design errors with this code.

The solution is to the restructure the code with something like:

``` {.brush: .cpp; .title: .; .notranslate title=""}

std::mutex m;



void SwipeCard()

{

    std::cout << "Card swiping ... " << std::endl;

    std::lock_guard<std::mutex> waterCoolerLock(m);

    std::cout << "Card swiped" << std::endl;

}



void EnterChamber()

{

    std::cout << "Chamber unlocking ... " << std::endl;

    std::lock_guard<std::mutex> waterCoolerLock(m);

    std::cout << "Chamber unlocked" << std::endl;

}



void GetWater()

{

    EnterChamber();

    SwipeCard();



    std::cout << "Water pouring" << std::endl;

}



void Sector7GSituation()

{

    GetWater();

}
```

Output:

``` {.brush: .plain; .title: .; .notranslate title=""}

Chamber unlocking ... 

Chamber unlocked

Card swiping ... 

Card swiped

Water pouring
```

**Deadlocks and libdispatch**

Moving over to GCD, we don’t have to worry about such nitty-gritty
details, as we don’t have to care about mutex and locks. But still
deadlocks can happen, because deadlocks are more of design errors than
anything else.

Here’s one example of a deadlock using GCD

``` {.brush: .cpp; .title: .; .notranslate title=""}

let queue:dispatch_queue_t = dispatch_queue_create("deadlockQueue", nil)



func SimpleDeadlock()

{

    dispatch_sync(queue) {

        println("Enter task: 0")

        dispatch_sync(queue) {

            println("Enter task: 1")

            println("Exit task: 1")

        }

        println("Exit task: 0")

    }

}



SimpleDeadlock()
```

The jobs sounds simple enough, we have a serial queue. We have two
tasks, we need them to be executed sequentially. Each task has a taskId
associated with it, just so that we can keep track of what’s getting
executed. We use dispatch\_sync because we want our tasks to be executed
one after the other.

Here’s the output, before it gets deadlocked:

``` {.brush: .plain; .title: .; .notranslate title=""}

Enter task: 0
```

So, why isn’t the task 1 getting executed? Well, lets understand how
dispatch\_sync works. A dispatch\_sync blocks the queue until it has
been executed fully. Consider the following situation:

> Say there is a hair saloon, they’ve a very strict first come first
> serve policy. A mother and a daughter visit the saloon. The mother
> steps in first so gets the client id 0 while the daughter gets client
> id 1. According to the rules, they have to serve the mother first,
> because she has the lowest numbered client id. But, the mother insists
> that they first serve her daughter. If they start serving the daughter
> they’re actually breaking the rule, as then they would be serving
> client 1 before 0. Hence, the deadlock.

And this is exactly why task 1 is not getting initiated, because the
serial dispatch\_queue is waiting for the task 0 to finish. But, the
task 0 is insisting the queue to finish the task 1 first.

There are two ways to resolve this deadlock. First is using a concurrent
queue instead of a serial queue.

``` {.brush: .cpp; .title: .; .notranslate title=""}

let queue:dispatch_queue_t = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0)



func SimpleDeadlock()

{

    dispatch_sync(queue) {

        println("Enter task: 0")

        dispatch_sync(queue) {

            println("Enter task: 1")

            println("Exit task: 1")

        }

        println("Exit task: 0")

    }

}



SimpleDeadlock()
```

Output

``` {.brush: .plain; .title: .; .notranslate title=""}

Enter task: 0

Enter task: 1

Exit task: 1

Exit task: 0
```

The reason why this works is that a concurrent queue is allowed to work
on more than one submitted task at a time. So, first task 0 is
submitted. Task 0 then submits task 1 and demands it to be executed
synchronously or in other words to be finished before finishing task 0.
Since this is a concurrent queue, it’s distributes its time over all the
queued tasks.

In terms of our mother-daughter-and-the-saloon example above. Lets say
the saloon has a big clock that ticks every 1 minute. For that 1 minute
they focus on a single client and then at the next tick they switch to
another client, irrespective of what state the client is in.

So at first tick they focus on the mother, the mother just sits there
not allowing them to even touch her hairs before her daughter is done.
At next tick, they switch to the daughter and start serving her. This
repeats until the daughter is fully done, then the mother allows them to
work on her hairs.

This works, but it has a problem. Remember, we wanted the task 0 to be
fully completed before task 1. That’s probably why we thought of using
serial queues and synchronous tasks. This solution actually breaks it.
What we actually need is a serial queue with tasks asynchronously
submitted.

``` {.brush: .cpp; .title: .; .notranslate title=""}

let queue:dispatch_queue_t = dispatch_queue_create("deadlockQueue", nil)



func SimpleDeadlock()

{

    dispatch_async(queue) {

        println("Enter task: 0")

        dispatch_async(queue) {

            println("Enter task: 1")

            println("Exit task: 1")

        }

        println("Exit task: 0")

    }

}



XCPSetExecutionShouldContinueIndefinitely(continueIndefinitely: true)



SimpleDeadlock()
```

Output:

``` {.brush: .plain; .title: .; .notranslate title=""}

Enter task: 0

Exit task: 0

Enter task: 1

Exit task: 1
```

This solution submits task 1 after task 0 to a serial queue. Since, this
is a serial queue, the task ordering is preserved. Also, since tasks do
not depend on each other, we never see any deadlocks.

The lesson here is to use dispatch\_sync very carefully. Remember
deadlocks are always design errors.

The code for todays experiment is available at
[github.com/chunkyguy/ConcurrencyExperiments](https://github.com/chunkyguy/ConcurrencyExperiments/tree/master/105_Deadlocks).
Check it out and have fun experimenting on your own.

