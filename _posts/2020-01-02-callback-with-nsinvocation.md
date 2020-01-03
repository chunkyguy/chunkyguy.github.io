---
layout: post
title:  "Implementing callback with NSInvocation"
date:   2020-01-02 23:28:54 +0200
categories: objc architecture
published: true
---

Let's say we wanted to write a [callback](https://en.wikipedia.org/wiki/Callback_(computer_programming)) with `NSInvocation`. Why would you need that? Maybe you do not like the block syntax, or maybe you do not like how blocks capturing works, or maybe you're working with a really old codebase which can not use blocks.

### Anatomy of a callback

Anyone who has been writing software for a while knows what a callback is. I struggled a bit when I first encountered the concept for the first time. In my novice head programs were simple chain of procedures that execute one after the other. Like, **flowA -> flowB**. I had never experienced a flow where a procedure would need to be interrupted for a while and then resume where it left. Like **flow_begin -> interrupt -> flow_end**.

I distinctly remember the day I first encountered the `signal` API.

```c
void (*signal (int, void (*)(int))) (int);
```

If this is the first time you're looking at the monstrosity, let me break it down in simpler words. `signal` is a function that takes in 2 arguments, a `int` signal value and a function pointer of type `void (*)(int)` and returns a function pointer of type `void (*)(int)`. The function pointers here represent callbacks.

Another simpler example is of the C standard library [quicksort](https://en.wikipedia.org/wiki/Qsort):

```c
void qsort( void *ptr, size_t count, size_t size,
            int (*comp)(const void *, const void *) );
```

The interesting thing here is that `qsort` takes in a `comp` as a callback. So the algorithm can simply work on some abstract level as long as a user provides the comparison function.

### Callback with Delegation

All modern languages provide a simpler way to represent callbacks. Objective-C since 2011 supports blocks, which although looks very similar to C function pointer but is a lot more than just simple callbacks. A block also capture the current state of stack implicitly that helps a lot with sharing data between **flow_begin**, **interrupt** and **flow_end**.

Before blocks, there were some old school ways of handling callbacks in Objc. One the most used pattern was to use delegation, which went something like:

1. Define a `protocol` for the **interrupt**
1. **Flow** is injected with a instance that conforms to `protocol`, called a **delegate**
1. Whenever required the **flow** calls the **delegate** methods.

As an example, the `qsort` above might be written in Objc as:

```objc
@protocol PLComparison <NSObject>
- (int)compare:(void *)first to:(void *)second;
@end

@interface PLQSort : NSObject
- (void)qsortData:(void *)data count:(size_t)count
  compareDelegate:(id<PLComparison>)delegate;
@end
```

### Callback with Block

These days blocks are the most accepted way of implementing a callback. This is even more true with swift. The idea is very simple.

1. Define a method signature

```objc
typedef void(^PLCalcAddCompletion)(NSInteger);
```

2. Define a method that implements the flow

```objc
@interface PLNonEscapingCalc : NSObject
- (void)add:(NSInteger)first
       with:(NSInteger)second
      block:(PLCalcAddCompletion)block;
@end

@implementation PLNonEscapingCalc
- (void)add:(NSInteger)first
       with:(NSInteger)second
      block:(PLCalcAddCompletion)block;
{
  NSInteger sum = first + second;
  block(sum);
}
@end
```

3. Call the flow and inject the callback

```objc
- (void)run
{
  PLNonEscapingCalc *calc = [[PLNonEscapingCalc alloc] init];
  [calc add:10 with:20 block:^(NSInteger sum) {
    [self print:sum];
  }];
  [calc release];
}

- (void)print:(NSInteger)sum
{
  NSLog(@"sum: %ld\n", sum);
}
```

With blocks the `PLNonEscapingCalc` has to nothing about its caller, as all the type information it needs is available right there in the signature.

### Callback with NSInvocation

With `NSInvocation` the flow is very much the same. 

1. Define a method that implements the flow. But since this is Objc, let the runtime figure out the callback signature for later.

```objc
@interface PLNonEscapingCalc : NSObject
- (void)add:(NSInteger)first
       with:(NSInteger)second
 invocation:(NSInvocation *)invocation;
@end

@implementation PLNonEscapingCalc
- (void)add:(NSInteger)first
       with:(NSInteger)second
 invocation:(NSInvocation *)invocation;
{
  NSInteger sum = first + second;
  [invocation setArgument:&sum atIndex:2];
  [invocation invoke];
}
@end
```

Notice that we set the argument as index 2. This because in Objc index 0 and 1 are reserved for *receiver* and *selector*. We have to provide those values from the other end:

```objc
- (void)run
{
  PLNonEscapingCalc *calc = [[PLNonEscapingCalc alloc] init];
  NSInvocation *invocation = [NSInvocation invocationWithMethodSignature:
                              [self methodSignatureForSelector:
                               @selector(print:)]];
  [invocation setTarget:self];
  [invocation setSelector:@selector(print:)];
  [calc add:30 with:40 invocation:invocation];
  [calc release];
}
```

With `NSInvocation` approach the `PLNonEscapingCalc` knows absolutely nothing about its caller. The only thing it cares about is to set data at index 2. So, we rely on a proper documentation on make sure the contract is valid.

### Build a networking layer

For a final exercise, lets try to build a `NSURLSession` like API but based on `NSInvocation` instead of blocks.

First we need something to hold the `NSInvocation` and the `NSData` as the `NSURLSessionDataDelegate` might return partial data. From the [doc](https://developer.apple.com/documentation/foundation/nsurlsessiondatadelegate/1411528-urlsession?language=objc):

> This delegate method may be called more than once, and each call provides only data received since the previous call. The app is responsible for accumulating this data if needed.

```objc
@interface PLNetworkTask : NSObject
{
  NSURLSessionTask *_task;
  NSMutableData *_data;
  NSInvocation *_invocation;
}
@property (nonatomic, readonly) NSURLSessionTask *task;
@end

@implementation PLNetworkTask
+ (instancetype)futureWithTask:(NSURLSessionTask *)task
                    invocation:(NSInvocation *)invocation
{
  return [[[self alloc] initWithTask:task
                          invocation: invocation] autorelease];
}

- (instancetype)initWithTask:(NSURLSessionTask *)task
                  invocation:(NSInvocation *)invocation
{
  self = [super init];
  if (self) {
    _task = [task retain];
    _data = [[NSMutableData alloc] init];
    _invocation = [invocation retain];
  }
  return self;
}

- (void)dealloc
{
  [_invocation release];
  [_task release];
  [_data release];
  [super dealloc];
}

- (void)appendData:(NSData *)data
{
  [_data appendData:data];
}

- (void)completeWithError:(NSError *)error
{
  [_invocation setArgument:&_data atIndex:2];
  [_invocation setArgument:&error atIndex:3];
  [_invocation invoke];
}
@end
```

Next we need something like a `NSURLSession` than manages the `PLNetworkTask`s.

```objc
@interface PLNetworkSession() <NSURLSessionDataDelegate>
{
  NSURLSession *_session;
  NSMutableArray *_tasks;
  NSOperationQueue *_sessionQueue;
}
@end

@implementation PLNetworkSession

- (instancetype)init
{
  self = [super init];
  if (self) {
    _sessionQueue = [[NSOperationQueue alloc] init];
    [_sessionQueue setMaxConcurrentOperationCount:1];
    _tasks = [[NSMutableArray alloc] init];
    _session = [[NSURLSession sessionWithConfiguration:
                 [NSURLSessionConfiguration defaultSessionConfiguration]
                                              delegate:self
                                         delegateQueue:_sessionQueue]
                retain];
  }
  return self;
}

- (void)dealloc
{
  [_sessionQueue release];
  [_tasks release];
  [_session release];
  [super dealloc];
}

- (void)dispatchRequest:(NSURLRequest *)request
             invocation:(NSInvocation *)invocation
{
  NSAssert([invocation target] != nil, @"Target should be set");
  NSAssert([invocation selector] != nil, @"Selector should be set");

  NSURLSessionDataTask *task = [_session dataTaskWithRequest:request];
  [_tasks addObject:[PLNetworkTask futureWithTask:task
                                       invocation: invocation]];
  [task resume];
}

- (PLNetworkTask *)futureWithTask:(NSURLSessionTask *)task
{
  for (PLNetworkTask *future in _tasks) {
    if ([future task] == task) {
      return future;
    }
  }
  return nil;
}

#pragma mark NSURLSessionDataDelegate
- (void)URLSession:(NSURLSession *)session
          dataTask:(NSURLSessionDataTask *)dataTask
    didReceiveData:(NSData *)data;
{
  [[self futureWithTask:dataTask] appendData:data];
}

#pragma mark NSURLSessionTaskDelegate
- (void)URLSession:(NSURLSession *)session
              task:(NSURLSessionTask *)task
didCompleteWithError:(nullable NSError *)error;
{
  NSLog(@"PLNetworkSession::didCompleteWithError");
  PLNetworkTask *future = [self futureWithTask:task];
  [future completeWithError:error];
  [_tasks removeObject:future];
}

@end
```

And finally we can start using `PLNetworkSession` to schedule async tasks

```objc
- (void)handleData:(NSData *)data error:(NSError *)error
{
  if (!data) {
    return;
  }
  NSString *dataStr = [[NSString alloc] initWithData:data
                                            encoding:NSUTF8StringEncoding];
  NSLog(@"data %@", dataStr);
  [dataStr release];
}

- (void)runNetwork
{
  self.session = [[PLNetworkSession alloc] init];
  NSInvocation *invocation = [NSInvocation invocationWithMethodSignature:
                              [self methodSignatureForSelector
                               :@selector(handleData:error:)]];
  [invocation setTarget:self];
  [invocation setSelector:@selector(handleData:error:)];
  NSURLRequest *request = [NSURLRequest requestWithURL:
                           [NSURL URLWithString:
                            @"https://jsonplaceholder.typicode.com/todos/1"]];
  [_session dispatchRequest:request invocation:invocation];
}
```

Output

```
data {
  "userId": 1,
  "id": 1,
  "title": "delectus aut autem",
  "completed": false
}
```

### Afterwords

This was more or less just an demo on how `NSInvocation` can be used to build a callback pattern. Swift does not even exposes `NSInvocation` so one has to use some other mechanism when working with Swift. I personally have no favorites, like every other thing when writing softwares we have to be judicious when selecting the right tool for the job. If for example there is no data transfer, rather only control transfer among several procedures, `NSInvocation` could be a better fitting tool. Like say the *libdispatch* library.