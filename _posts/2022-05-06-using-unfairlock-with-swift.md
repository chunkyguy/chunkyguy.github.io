---
layout: post
title:  "Using UnfairLock with Swift"
date:   2022-05-16 15:00:00 +0200
categories: swift concurrency
published: true
---

`UnfairLock` seems to be causing a lot of confusion for Swift developers. Every once in a while I run across some *incorrect* implementation that reads like:

```swift
var lock = os_unfair_lock_s()
os_unfair_lock_lock(&lock)

// code here ...

os_unfair_lock_unlock(&lock)
```

The confusion seems to arise from the fact that `&` is used to take address of a variable in C while it is used for in-out operation in Swift, which seem similar but are not. To make `os_unfair_lock_lock` work the `lock` location in memory needs to not move, which isn't guaranteed by Swift. So the above code might or might not work as expected. To understand better we need to take a look at equivalent C code:

```c
os_unfair_lock lock = OS_UNFAIR_LOCK_INIT;
os_unfair_lock_lock(&lock);

// code here ...

os_unfair_lock_unlock(&lock);
```

In C API `os_unfair_lock` is a `struct` defined as:

```c
typedef struct os_unfair_lock_s { 
    // ...
} os_unfair_lock, *os_unfair_lock_t;
```

Since Swift does not provide a `OS_UNFAIR_LOCK_INIT` equivalent, we can not allocated `os_unfair_lock` in stack memory. Another alternative is to allocate in the heap memory

```c
os_unfair_lock_t lock;
lock = malloc(sizeof(os_unfair_lock));
os_unfair_lock_lock(lock);

// code here ...

os_unfair_lock_unlock(lock);
free(lock);
lock = NULL;
```

The equivalent in Swift would be:

```swift
public typealias os_unfair_lock_t = UnsafeMutablePointer<os_unfair_lock_s>

var lock: os_unfair_lock_t // os_unfair_lock_t lock;
lock = UnsafeMutablePointer<os_unfair_lock_s>.allocate(capacity: 1) // lock = malloc(sizeof(os_unfair_lock));
lock.initialize(to: os_unfair_lock()) // memset(lock, 0, sizeof(os_unfair_lock));
os_unfair_lock_lock(lock)

// code here ...

os_unfair_lock_unlock(lock)
lock.deallocate() // free(lock);
```

### Further Reading

* [The Peril of the Ampersand](https://developer.apple.com/forums/thread/674633)
* [Interacting with C Pointers](https://developer.apple.com/swift/blog/?id=6)
* [Swift Ownership Manifesto](https://github.com/apple/swift/blob/main/docs/OwnershipManifesto.md)