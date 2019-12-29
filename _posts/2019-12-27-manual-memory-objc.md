---
layout: post
title:  "Lost art of manual memory management"
date:   2019-12-27 21:16:00 +0200
categories: objc mrr arc
published: true
---

Manually managing of memory is an art that is getting extinct at an alarming pace. If you started working with Objective-C after 2011 or with Swift there is a high chance you'd probably never worked with manual memory management aka **Manual Retain Release (MRR)**. And if one never had an opportunity of first hand experiencing MMR, one might even think of it as some outrageous technique where the entire code is blathered with calls of retain and release all over the place. Even more so when its antagonist, the **Automatic Reference Cycle (ARC)** claims to solve a problem which was not as monstrous as ARC claims in the first place.

**Myth**: MRR is dead.

**Fact**: MRR is still pretty much in use. ARC does help with hiding some redundant information from our instant vision at cost of rare but hard to diagnose bugs, but internally the system still performs the same very classic reference counting.

## Manual Retain Release (MRR)

Before we dive any deeper we would have to refresh some basic concepts.

### Stack vs heap

From a certain perspective a program can be imagined as made up of many smaller procedures chained together. Think of a giant stack of procedures with following rules:

1. A procedure is just a group of instructions.
1. Every procedure is pushed to the stack with some data passed as arguments and some local memory space, lets call it as the *stack space* of the procedure.
1. A procedure can push another child procedure on the stack.
1. At any given moment only the stacks's topmost procedure is executing and the rest of the procedures are frozen with their last states.
1. When a procedure is complete, it is popped and optionally some data is returned back to its parent procedure. 
1. When popped, the *stack space* of the procedure just popped is cleared and any local data in that space is now invalid for access. Note that, access to any parent procedure's *stack space* is always valid for access but may not be visible to child procedure.
1. The first procedure is called `main()` and is pushed with some predefined system arguments and returns some predefined system values.

So the only way for procedures to pass data among each other is via the arguments and return data. Sometimes we might need the data in a procedure's *stack space* to have a longer lifespan than the procedure itself. For example a procedure might compute some expensive data that we wish to reuse. Then we can move this data to some global shared memory space called as the *heap space*.

Once some data has been moved to *heap space* it will live there as long as we explicitly do not clean it.

### Reference counting

With languages like ObjC most of data is allocated in heap. In fact it is not even allowed to allocate pure ObjC objects in stack.

Smart people at ObjC team realized this problem very early on. So they built the entire system of reference counting. The way it works is by keeping a counter for every object allocated in heap. Every procedure that needs to keep that object alive increments the count by calling `retain` and decrement when done by calling `release`. As long as the retain count is more than zero the object is kept alive.

Of course not every object access has to be retained. For example, if we know for sure that the object passed as argument is retained by the parent's stack space and we know that our access is short lived than the parent's stack space we don't need to retain it.

```objc
- (void)bindViewModel:(PAListItemViewModel *)viewModel;
{
    // No need to retain viewModel
    _imageView.image = viewModel.image;
}
```

###  Memory leaks vs corruption

With manually memory management there are mostly two kind of memory related errors.

**Memory leaks** happens when we forget to call `release` and some allocated memory is never released. This is not harmful immediately as it would not crash the app, but starts building the app for a later crash depending on how much memory is being leaked. But when the app does crash it's really hard to debug the cause, since the crash could be unpredictable.

```objc
-(void)leakyMethod
{
  NSArray *arr = [[NSArray alloc] initWithObjects:@"hello", @"world", nil];
  // the arr was never released
  NSLog(@"arr: %lu",[arr count]);
}
```

**Memory corruption** happens when we call `release` on already released memory. Or if the pointer is holding some memory address which is invalid. This could happen with a dangling pointer. It usually is an immediate runtime crash and one of the more frequent reason for app crashes.

```objc
-(void)crashyMethod
{
  NSArray *arr = [[NSArray alloc] initWithObjects:@"hello", @"world", nil];
  [arr release];
  // arr is now dangling
  NSLog(@"arr: %lu",[arr count]);
}
```

Remember than in Objc it is always safe to send a message to `nil`. So above crash can be fixed by setting `arr` to `nil`.

```objc
-(void)nonCrashyMethod
{
  NSArray *arr = [[NSArray alloc] initWithObjects:@"hello", @"world", nil];
  [arr release];
  arr = nil;
  // arr is now safe
  NSLog(@"arr: %lu",[arr count]);
}
```

### Autorelease pool

Another important concept is of autorelease pool. Let's say we need to create an `NSString` and return it back to the caller and we implement it like this:

```objc
-(NSString *)getIdentifier
{
  NSString *str = [[NSString alloc] initWithData:_someData
                                        encoding:NSUTF8StringEncoding];
  [str release];
  return str;
}
```

Since the `str` is released before returning, by the time caller gets access to the `NSString` it is already pointing to a deallocated memory. Looks a memory corruption, guaranteed crash at runtime.

Easy fix could be to return an allocated `NSString` hoping the caller would release it after use

```objc
-(NSString *)makeIdentifier
{
  NSString *str = [[NSString alloc] initWithData:_someData
                                        encoding:NSUTF8StringEncoding];
  return str;
}
```

But this implies that every caller has to know about the ownership responsibility. And what if the caller has to again forward this `NSString` to its caller? We need a solution where the caller if free to retain if they need the object for a longer period, otherwise it just gets deallocated. This is where `NSAutoreleasePool` shines.

```objc
-(NSString *)getIdentifier
{
  NSString *str = [[NSString alloc] initWithData:_someData
                                        encoding:NSUTF8StringEncoding];
  return [str autorelease];
}
```

We mark the instance to be kept alive for a very short span by transferring the ownership to the autorelease pool. Next, the caller can directly use it if they know for sure that they only need access before it gets deallocated which happens at next event cycle. Or they can retain it.

**Myth**: We can not get reference cycles with MRR

**Fact**: It is true that with MRR every reference is `weak` unless explicitly retained, but still we can run into situations where `Foo` retains `Bar` while `Bar` retains `Foo`.

```objc
// Forward declaration
@class Bar;

@interface Foo : NSObject
{
  Bar *_bar;
}
@end

@implementation Foo

- (instancetype)init
{
  self = [super init];
  if (self) {
    NSLog(@"Init Foo");
    _bar = [[Bar alloc] init];
    _bar.foo = self;
  }
  return self;
}

- (void)dealloc
{
  NSLog(@"Deinit Foo");
  [_bar release];
  [super dealloc];
}

@end

@interface Bar : NSObject
@property (nonatomic, retain) Foo *foo;
@end

@implementation Bar

- (instancetype)init
{
  self = [super init];
  NSLog(@"Init Bar");
  return self;
}

- (void)dealloc
{
  NSLog(@"Deinit Bar");
  [super dealloc];
}

@end
```

When called as:

```objc
-(void)referenceCycle
{
  Foo *foo = [[Foo alloc] init];
  [foo release];
}
```

Prints:

```
Init Foo
Init Bar
```

Just as with ARC the fix is to simply make the backward reference as `weak` or `assign` in MRR.

```objc
@interface Bar : NSObject
@property (nonatomic, assign) Foo *foo;
@end
```

Prints:

```
Init Foo
Init Bar
Deinit Foo
Deinit Bar
```

## Working with MRR in real life

With some basic concepts under our belt, lets take a look at some real world considerations when working with MRR.

### Avoid `property` when possible

Try hard reasoning to avoid `self` when it comes to using class data. We don't need every private data to be a `property` which is a norm when working with ARC. Think of `property` as ivar with convenience getter and setter, if there is a good reason for a getter or setter use `property` otherwise stick with ivar. In other words don't use `self.someProperty` when `_someProperty` will suffice.

```objc
- (NSInteger)collectionView:(UICollectionView *)collectionView numberOfItemsInSection:(NSInteger)section
{
  return [_delegate totalItemsListViewController:self];
}
```

Whenever in doubt mentally replace the fancy `.` with the full message passing syntax.

```objc
self.image = other.image;
```

is same as 
```objc
[self setImage:[other image]];
```

### Always use ivars from `init` and `dealloc`

```objc
- (instancetype)init
{
  self = [super init];
  if (!self) {
    return nil;
  }

  _viewModel = [[PAListViewModel alloc] init];

  return self;
}

- (void)dealloc
{
  [_viewModel release];
  [super dealloc];
}
```

The reason is that we don't usually want to invoke dynamic dispatch from these 2 special methods as the instance is partially initialized. Said that, it is not as problematic as in C++ or as in Swift where it is impossible (for good reasons).

### Reuse retain setter

A good understanding of how the setter for `retain` is synthesized never hurts. The way to learn is by implementing one ourself even if just for the kicks.

```objc
- (void)setCount:(NSNumber *)newCount 
{
    // acquire
    [newCount retain];
    // dispose
    [_count release];
    // exchange
    _count = newCount;
}
```

Then try to always use this setter, even when rolling back to some default value or `nil`. This is probably one of the best cases where using a `property` is the best choice.

```objc
@property (nonatomic, retain) NSNumber *count;
```

### Provide convenience initializers. 

Convenience does not mean just for the sake of readability. Think more from the memory ownership perspective. Lets take a look at 2 similar looking `NSString` initializers:

```objc
+ (instancetype)stringWithContentsOfURL:(NSURL *)url 
                               encoding:(NSStringEncoding)enc 
                                  error:(NSError * _Nullable *)error;

- (instancetype)initWithContentsOfURL:(NSURL *)url 
                             encoding:(NSStringEncoding)enc 
                                error:(NSError * _Nullable *)error;
```

The class methods implies we do not own the instance, while the instance method means we own the ownership and would have to manage the memory ourself.

```objc
- (void)loadContentWithURL:(NSURL *)url
{
  NSString *s1 = [NSString stringWithContentsOfURL:url 
                                          encoding:NSUTF8StringEncoding
                                             error:NULL];
  NSString *s2 = [[NSString alloc] initWithContentsOfFile:url                 
                                                 encoding:NSUTF8StringEncoding
                                                    error:NULL];

  // .. do stuff

  [s2 release];
}
```

### Understand ownership by external frameworks

We cannot manage the ownership everywhere. Check what the API looks like for the framework being used. Are they going to retain or simply keep a weak reference? For example this is what the interface for `UIWindow` looks like:

```objc
@interface UIWindow : UIView
@property(nonatomic, weak) UIWindowScene *windowScene;
@property(nonatomic, strong) UIViewController *rootViewController;
@end
```

This suggests that the `windowScene` has to retained by us somehow whereas the `rootViewController` can be released.

Some system frameworks use naming conventions to imply ownership. For example, an `add` implies ownership while `set` does not.

### Blocks live on stack by default

The biggest change is in understanding how blocks behave with MRR. Blocks are stored on stack by default. In Swift terms every block is `non-escaping` and every block needs to be explicitly moved to heap if they're expected to run in some later time, or in other words if they are `escaping`. 

The way to do this is by calling `Block_copy` and `Block_release`.

```objc
- (void)getPhotoWithURL:(NSURL *)photoURL
             completion:(void (^)(UIImage *))completion
{
  Block_copy(completion);
  [self getDataWithRequest:[PAImageRequest requestWithURL:photoURL]
                completion:^(NSData *data) {
    if (data != nil) {
      completion([UIImage imageWithData:data]);
    } else {
      completion(nil);
    }
    Block_release(completion);
  }];
}
```

## Resources

There are probably a billion articles still out there in the wild that discuss these topic in details. I would not link them here, any article before 2011 on ObjC is probably going to discuss MRR at some level. But I would link some official resources on working outside the comfort zone of ARC, or understanding on how ARC works.

ARC is good and all but like every other thing in computer science it is not a silver bullet that is going to solve all our problems. Most likely with ARC you would not have as many memory corruptions as everything is retained by default, but you might end up with more memory leak issues like with reference cycles.

Compared to C or classic C++ where we have to use very discipline in managing memory, I think ObjC with its reference counting and autorelease pool is already good enough to not run into a lot of issues C/C++ have.

I love C++ and ObjC, probably my all time favorite languages, but in their own pure ways. The more I think about it, it seems to me that ARC was designed by people who either never really understood or appreciated the core philosophies of ObjC. ARC feels like a language patch from people who love C++, and wanted the memory runtime behavior to be more in line of `std::shared_ptr` which MRR already did better.

1. [Advanced Memory Management Guide](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/MemoryMgmt/Articles/MemoryMgmt.html)
1. [Objective-C Automatic Reference Counting (ARC)](https://clang.llvm.org/docs/AutomaticReferenceCounting.html)
1. [Coding Guidelines for Cocoa](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/CodingGuidelines/CodingGuidelines.html#//apple_ref/doc/uid/10000146-SW1)