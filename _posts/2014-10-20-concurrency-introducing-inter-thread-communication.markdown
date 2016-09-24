---
layout: post
title:  "Concurrency: Introducing Inter-Thread Communication"
date:   2014-10-20 10:23:00 +0530
categories: jekyll update
---


Picture the scenario: You’re on the phone with your friend, when
suddenly he say ‘Hey! Hold on. I think I’m getting another call’ and
before you can say anything, he puts you on hold. Now, you have no idea
whether to hang up, or wait. Your friend could be coming back in 3
seconds or 3 hours you’ve no idea. Wouldn’t it be great if your phone
service provided you a mechanism that whenever you’re put on hold, you
can freely disconnect and whenever your friend disconnects the other
call, you immediately get a ring. Let’s talk about managing
synchronization between threads.

On a broader level the above is a [classical producer-consumer
problem](http://en.wikipedia.org/wiki/Semaphore_(disambiguation)). Both
consumer and producer are synchronized by a shared data. What we
actually need is a way to communicate between the producer and the
consumer. The producer needs to signal the consumer the data is ready
for consumption.

So far in all our concurrency experiments whenever we’ve some shared
data, we explicitly block one or more threads while one thread gets to
play with the resource. The shared data here seems to be a boolean flag
of some sort. But we can do better with std::condition\_variable

``` {.brush: .cpp; .title: .; .notranslate title=""}

    std::mutex mutex;

    std::condition_variable condition;



    void StartConsumer()

    {

        std::cout << std::this_thread::get_id() << " Consumer sleeping" << std::endl;

        std::unique_lock<std::mutex> consumerLock(mutex);

        condition.wait(consumerLock);

        consumerLock.unlock();

        

        std::cout << std::this_thread::get_id() << " Consumer working" << std::endl;

    }

    

    void StartProducer()

    {

        std::cout << std::this_thread::get_id() << " Producer working" << std::endl;

        std::this_thread::sleep_for(std::chrono::seconds(1));

        std::lock_guard<std::mutex> producerLock(mutex);

        condition.notify_one();

        std::cout << std::this_thread::get_id() << " Producer sleeping" << std::endl;

    }

    

    void StartProcess()

    {

        std::thread consumerThread(StartConsumer);

        std::thread producerThread(StartProducer);

        

        consumerThread.join();

        producerThread.join();

    }
```

Output

``` {.brush: .plain; .title: .; .notranslate title=""}

0x100ced000 Consumer sleeping

0x100d73000 Producer working

0x100d73000 Producer sleeping

0x100ced000 Consumer working
```

In the above code both the consumer and producer execute simultaneously
in concurrent threads. The consumer waits for the producer to finish
working on whatever it was doing. But instead of continuously checking
for whether the producer is done as in

``` {.brush: .cpp; .title: .; .notranslate title=""}

while (!done) {

    std::unique_lock<std::mutex> consumerLock(mutex);

    consumerLock.unlock();

    std::this_thread::sleep_for(std::chrono::microseconds(1));

}
```

We make use of condition variable to inform us whenever the producer is
done. The producer then using notify\_one() (or maybe notify\_all())
sends a notification to all the waiting threads to continue.

Moving to GCD, we’ve two ways. First is to use dispatch\_group.

``` {.brush: .cpp; .title: .; .notranslate title=""}

let concurrentQueue:dispatch_queue_t = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0)

let group:dispatch_group_t = dispatch_group_create()



func StartConsumer()

{

    println("Consumer sleeping")

    dispatch_group_wait(group, DISPATCH_TIME_FOREVER);

    println("Consumer working")

}



func StartProducer()

{

    dispatch_group_async(group, concurrentQueue) { () -> Void in

        println("Producer working")

        sleep(1)

        println("Producer sleeping")

    }

}



func StartProcess()

{

    StartProducer()

    StartConsumer();

}
```

Output:

``` {.brush: .plain; .title: .; .notranslate title=""}

Producer working

Consumer sleeping

Producer sleeping

Consumer working
```

The first thing that you’ll notice in this code is that we start the
producer code first and then the consumer waits. This is because if we
start the consumer first, the queue has nothing in it to wait for.

dispatch\_group is actually meant to synchronize a set a tasks in a
queue. For example, you’re loading the next level in your game. For
that, you need to download a lot of images and other data files from a
remote server before finishing the loading. What you’re doing is
synchronizing a lot of async tasks and wait for them to finish before
doing something else.

For this experiment, we can better use a dispatch\_semaphore.
dispatch\_semaphore is a functionality based on the [classic semaphores
concepts]() you must’ve learned during your college days.

``` {.brush: .cpp; .title: .; .notranslate title=""}

let sema:dispatch_semaphore_t = dispatch_semaphore_create(1);



func StartConsumer()

{

    println("Consumer sleeping")

    dispatch_semaphore_wait(sema, DISPATCH_TIME_FOREVER)

    println("Consumer working")

}



func StartProducer()

{

    println("Producer working")

    sleep(1)

    println("Producer sleeping")

    dispatch_semaphore_signal(sema)

}



func StartProcess()

{

    StartProducer()

    StartConsumer();

}
```

If you look closely, we haven’t even used any dispatch\_queue here. This
is because dispatch\_semaphore is more about synchronization than about
threading.

dispatch\_semaphore is basically just a count based blocking system. We
just create a dispatch\_semaphore object by telling it how many counts
can be decremented before it blocks. Then on each wait it decrements the
count and on each signal it increments the count. If the count reaches
less than 0, the thread is blocked until somebody issues a signal.

Here’s another example to illustrate on using semaphores between two
queues

``` {.brush: .cpp; .title: .; .notranslate title=""}

let sema:dispatch_semaphore_t = dispatch_semaphore_create(0);

let concurrentQueue:dispatch_queue_t = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0)



func StartConsumer()

{

    println("Consumer sleeping")

    dispatch_semaphore_wait(sema, DISPATCH_TIME_FOREVER)

    println("Consumer working")

}



func StartProducer()

{

    dispatch_async(concurrentQueue) {

        println("Producer working")

        sleep(1)

        println("Producer sleeping")

        dispatch_semaphore_signal(sema)

    }

}



func StartProcess()

{

    StartProducer()

    StartConsumer();

}
```

Again note that we’re starting producer first, and because we initialize
the dispatch\_semaphore with 0, the consumer is going to block the main
thread as soon as it is invoked. It won’t even give the chance to run
the producer code in parallel.

Here’s the output of above code:

``` {.brush: .plain; .title: .; .notranslate title=""}

Producer working

Consumer sleeping

Producer sleeping

Consumer working
```

This is just an scratch of the surface of inter-thread communication.
Most of problems with concurrency actually somehow deal with
inter-thread communication. In later sessions we shall experiment more
on this. The idea for this article came from the [this
discussion](http://www.reddit.com/r/gamedev/comments/2ivsp6/async_loading_and_syncing_threads_with_your/)
on problem of loading game data in a parallel thread and [the
solution](https://gist.github.com/chunkyguy/e4f52c0a75f5ae4c7088) I
worked on.

The code for today’s experiment is available at
[github.com/chunkyguy/ConcurrencyExperiments](https://github.com/chunkyguy/ConcurrencyExperiments/tree/master/104_InterThreadCommunication).
Check it out and have fun experimenting on your own.

