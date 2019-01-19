---
layout: post
title:  "Concurrency: Working with shared data using queues"
date:   2014-09-27 23:28:54 +0530
categories: concurrency
---

# Concurrency: Working with shared data using queues

All problems with concurrency basically boil down to two categories: Race conditions and deadlocks. Today let’s look into one specific problem under race conditions: Working with shared data.

If every thread had to deal with the only data provided exclusively to it, we won’t ever get into any sort of data race problems. In fact, this is one of the main features of functional programming languages like Haskell that advertises heavily about concurrency and parallelism.

But there’s nothing to be worried about, functional programming is more about adopting a design pattern, just as object oriented programming is. Yes, some programming languages offer more towards a particular pattern, but that doesn’t means that we can not adopt that pattern in any other language. We just have to restrict ourselves to a particular convention. For example, there are a plenty or large softwares written with C programming language adopting the object oriented pattern. And for us, both C++ and Swift are very much capable of adopting the functional programming paradigm.

Getting back to the problem of shared data. Let’s look into when is shared data a problem with multiple threads. Suppose we have an application where two or more threads are working on a shared data. If all each threads ever does is a read activity, we won’t have any data race problem. For example, you’re at the airport and checking out your flight status on the giant screen. And, you’re not the only one looking at the status screen. But, does it really matters how many others are reading the information from the same screen? Now suppose the scenario is a little more realistic. Every airliner has a control with which they can update the screen with new information, while a passenger is reading the screen, the situation can get a little difficult. At any given instance, one or more airliner can be updating the screen at same instance, which could result in some garbled text to the passengers reading the screen. The two or airliners are actually in a race condition for a single resource, the screen.

If you’re using Cocoa, you must be familiar with the NSMutableArray. Also, you must be familiar that NSMutableArray isn’t thread-safe as of this writing. So, the question is how do make use of NSMutableArray in a situation like this, where we have concurrent read and write operations going on?

Let’s start with a simple single threaded model of what we’re doing:
``` objc
/**
 * Concurreny Experiments: Shared data
 * Compile: clang SharedData.m -framework Foundation && ./a.out
 */

#import <Foundation/Foundation.h>

@interface Airport : NSObject {
    NSMutableArray *airportStatusBoard;
    NSUInteger writeCount;
    NSUInteger readCount;
}
@end

@implementation Airport

- (id)init
{
    self = [super init];
    if (!self) {
        return nil;
    }
    
    airportStatusBoard = [[NSMutableArray alloc] init];
    writeCount = 0;
    readCount = 0;
    
    return self;
}

- (void)dealloc
{
    [airportStatusBoard release];
    [super dealloc];
}

- (void)addFlightEntry
{
    NSNumber *newEntry = @(random() % 10);
    [airportStatusBoard addObject:newEntry];
}

- (void) statusWriter
{
    NSLog(@"writing begin: %@", @(writeCount));
    [self addFlightEntry];
    NSLog(@"writing ends: %@", @(writeCount));
    writeCount++;
}

- (BOOL) statusReaderWithFlightNumber:(NSNumber *)searchFlightNum
{
    NSLog(@"reading begin: %@", @(readCount));
    
    BOOL found = NO;
    for (NSNumber *flightNum in airportStatusBoard) {
        if ([flightNum isEqualToNumber:searchFlightNum]) {
            found = YES;
            break;
        }
    }
    
    NSLog(@"reading ends: %@", @(readCount));
    
    readCount++;
    
    return found;
}

- (void) serialLoop
{
    srandom(time(0));
    while (![self statusReaderWithFlightNumber:@(7)]) {
        [self statusWriter];
    }
    
    NSLog(@"Flight found after %@ writes and %@ reads", @(writeCount), @(readCount));
}

@end

int main()
{
    @autoreleasepool {
        Airport *airport = [[Airport alloc] init];
        [airport serialLoop];
    }
    
    return 0;
}
```

Here’s the output for the above program on my machine:

```
2014-09-27 14:32:12.636 a.out[9601:507] reading begin: 0
2014-09-27 14:32:12.638 a.out[9601:507] reading ends: 0
2014-09-27 14:32:12.639 a.out[9601:507] writing begin: 0
2014-09-27 14:32:12.639 a.out[9601:507] writing ends: 0
2014-09-27 14:32:12.640 a.out[9601:507] reading begin: 1
2014-09-27 14:32:12.640 a.out[9601:507] reading ends: 1
2014-09-27 14:32:12.641 a.out[9601:507] writing begin: 1
2014-09-27 14:32:12.641 a.out[9601:507] writing ends: 1
2014-09-27 14:32:12.642 a.out[9601:507] reading begin: 2
2014-09-27 14:32:12.642 a.out[9601:507] reading ends: 2
2014-09-27 14:32:12.643 a.out[9601:507] Flight found after 2 writes and 3 reads
```

As you can observe, we’re performing sequential writes and reads until we find the required flight number in the array, and everything works out great. But, if we now try to run each write and read concurrently. To do that we create a queue for reading and writing. We perform all the reading and writing operation on a concurrent queue. We’re trying to implement the scenario where one passenger (you) is interested in reading the status board from top to bottom and check if their flight is listed. When they reach the end of the list and did not find the flight number they start again from the top. While, many airliner services are trying to update the single status board concurrently.

``` objc
readWriteQueue = dispatch_queue_create("com.whackylabs.concurrency.readWriteQ", DISPATCH_QUEUE_CONCURRENT);

- (void)parallelLoop
{
    srandom(time(0));

    __block BOOL done = NO;
    while(!done) {
        
        /* perform a write op */
        dispatch_async(readWriteQueue, ^{
            [self statusWriter];
        });
        
        /* perform a read op */
        dispatch_async(readWriteQueue, ^{
            if ([self statusReaderWithFlightNumber:@(SEARCH_FLIGHT)]) {
                done = YES;
            }
        });
    }
    
    NSLog(@"Flight found after %@ writes and %@ reads", @(writeCount), @(readCount));
}
```

If you now try to execute this program, once in a while you might get the following error:

```
Terminating app due to uncaught exception 'NSGenericException', reason: '*** Collection <__NSArrayM: 0x7f8f88c0a5d0> was mutated while being enumerated.'
```

This is the tricky part about debugging race conditions, most of the time your application might to be working perfectly, unless you deploy it on some other hardware or you get unlucky, you won’t get the issues. One thing you can play around with is, deliberately sleeping a thread to cause race conditions with sprinkling thread sleep code around for debugging:
``` objc
- (void) statusWriter
{
    [NSThread sleepForTimeInterval:0.1];
    writeCount++;
    NSLog(@"writing begin: %@", @(writeCount));        
    [self addFlightEntry];
    NSLog(@"writing ends: %@", @(writeCount));
}
```

If the above code doesn’t crashes on your machine on the first go, edit the sleep interval and/or give it a few more runs.

Now coming to the solution of such a race condition. The solution to such problem is in two parts. First is the write part. When multiple threads are trying to update the shared data, we need a way to lock the part where the actual update happens. It’s something like when multiple airliners are trying to update the single screen, we can figure out a situation like, only one airliner has the controls required to update the screen, and when it’s done, it releases the control to be acquired by any other airliner in queue.

Second, is the read part. We need to make sure that when we’re performing a read operation some other thread isn’t mutating the shared data as `NSEnumerator` isn’t thread-safe either.

A way to acquire such a lock with GCD is to use dispatch_barrier()
``` objc
- (void) statusWriter
{
    dispatch_barrier_async(readWriteQueue, ^{
        
        [NSThread sleepForTimeInterval:0.1];
        
        NSLog(@"writing begin: %@", @(writeCount));
        
        [self addFlightEntry];
        
        NSLog(@"writing ends: %@", @(writeCount));
        writeCount++;
        
    });
}

- (BOOL) statusReaderWithFlightNumber:(NSNumber *)searchFlightNum
{
   __block BOOL found = NO;
    
    dispatch_barrier_async(readWriteQueue, ^{
        
        NSLog(@"reading begin: %@", @(readCount));
        
        for (NSNumber *flightNum in airportStatusBoard) {
            if ([flightNum isEqualToNumber:searchFlightNum]) {
                found = YES;
                break;
            }
        }
        
        NSLog(@"reading ends: %@", @(readCount));
        
        readCount++;
        
    });
    
    return found;
}
```
If you try to run this code now, you’ll observe that the app doesn’t crash anymore, but the app has become significantly slower.

The first performance improvement that we can make is to not return from the read code immediately. Instead, we can let the queue handle the completion:
``` objc
- (void) statusReaderWithFlightNumber:(NSNumber *)searchFlightNum completionHandler:(void(^)(BOOL success))completion
{
    
    dispatch_barrier_async(readWriteQueue, ^{
        
        NSLog(@"reading begin: %@", @(readCount));
        
        BOOL found = NO;
        
        for (NSNumber *flightNum in airportStatusBoard) {
            if ([flightNum isEqualToNumber:searchFlightNum]) {
                found = YES;
                break;
            }
        }
        
        NSLog(@"reading ends: %@", @(readCount));
        
        readCount++;
       
        completion(found);
    });
}

- (void)parallelLoop
{
    srandom(time(0));
    
    __block BOOL done = NO;
    while(!done) {
        
        /* perform a write op */
        dispatch_async(readWriteQueue, ^{
            [self statusWriter];
        });
        
        /* perform a read op */
        dispatch_async(readWriteQueue, ^{
            [self statusReaderWithFlightNumber:@(SEARCH_FLIGHT) completionHandler:^(BOOL success) {
                done = success;
            }];
        });
    }
    
    NSLog(@"Flight found after %@ writes and %@ reads", @(writeCount), @(readCount));
}
```

Another thing to note is that, our code is no longer executing in parallel. As each dispatch_barrier() blocks the queue until all the pending tasks in the queue are complete. As is evident from the final log.

Flight found after 154 writes and 154 reads
In fact, the processing could be even worse than just running it sequentially, as the queue has to take care of the locking and unlocking overhead.

We can make the read operation as non-blocking again, as blocking the reads is not getting us any profit. One way to achieve that is to copy the latest data within a barrier and then use the copy to read the data while the other threads update the data.
``` objc
- (BOOL) statusReaderWithFlightNumber:(NSNumber *)searchFlightNum
{
    
    NSLog(@"reading begin: %@", @(readCount));
    
    __block NSArray *airportStatusBoardCopy = nil;
    dispatch_barrier_async(readWriteQueue, ^{
        
        airportStatusBoardCopy = [airportStatusBoard copy];
        
    });
    
    
    BOOL found = NO;
    
    for (NSNumber *flightNum in airportStatusBoardCopy) {
        if ([flightNum isEqualToNumber:searchFlightNum]) {
            found = YES;
            break;
        }
    }

    if (airportStatusBoardCopy) {
        [airportStatusBoardCopy release];
    }
    
    NSLog(@"reading ends: %@", @(readCount));
    
    readCount++;
    
    return found;
}
```

But, this won’t solve the problem. Most of reads are probably a waste of time. What we actually need is a way to communicate between the write and read. The actual read should only happen after the data has been modified with some new write.

We can start by separating the read and write tasks. We actually need the read operations to be serial, as we are implementing the case where a passenger is reading the list top-down and when they reach at the end, they start reading from the beginning. Whereas, many airliners can simultaneously edit the screen, so they still need to be in a concurrent queue.
``` objc
readQueue = dispatch_queue_create("com.whackylabs.concurrency.readQ", DISPATCH_QUEUE_SERIAL);
writeQueue = dispatch_queue_create("com.whackylabs.concurrency.writeQ", DISPATCH_QUEUE_CONCURRENT);

- (void)parallelLoop
{
    srandom(time(0));
    
    __block BOOL done = NO;
    while(!done) {
        
        /* perform a write op */
        dispatch_async(writeQueue, ^{
            [self statusWriter];
        });
        
        /* perform a read op */
        dispatch_async(readQueue, ^{
            [self statusReaderWithFlightNumber:@(SEARCH_FLIGHT) completionHandler:^(BOOL success) {
                done = success;
            }];
        });
    }
    
    NSLog(@"Flight found after %@ writes and %@ reads", @(writeCount), @(readCount));
}
```

Now, in the read operation we take a copy of the shared data. The way we can do it is by having an atomic property. Atomicity guarantees that the data is either have be successful or unsuccessful, there won’t be any in-betweens. So, in all cases we shall have a valid data all the time.
``` objc
@property (atomic, copy) NSArray *airportStatusBoardCopy;
```

In each read operation, we simply copy the shared data, so don’t care about mutability anymore.
``` objc
self.airportStatusBoardCopy = airportStatusBoard;
```
And further on we can use the copy to execute the read operations.
```
2014-09-27 16:20:33.145 a.out[12855:507] Flight found after 11 writes and 457571 reads
```

This is the queue level solution of the problem. In our next update we will take a look at how we can solve the problem at thread level using C++ standard thread library.

As always, the code for today’s experiment is available online at the github.com/chunkyguy/ConcurrencyExperiments.

Posted in concurrency, _dev	| Tagged C, Concurrency, Experiment, Swift	| 0 Comments


