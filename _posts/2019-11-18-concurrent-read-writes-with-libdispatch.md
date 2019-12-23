---
layout: post
title: "Concurrent read writes with libdispatch"
date: 2019-11-18 18:00:08
categories: ios libdispatch
published: true
---

Lets write a thread-safe collection. Say a cache that provide a way to load and store values. And for simplicity, we only care about storing `Int` for a `String` key.

### Basic Unsafe Read Writes

The simplest thing that comes to mind is a simple wrapper around mutating dictionary to provide a limiting interface.

```swift
class Cache {
    private var store: [String: Int] = [:]

    func get(key: String) -> Int? {
        return store[key]
    }

    func set(value: Int, key: String) {
        store[key] = value
    }
}
```

We need some driver code to use this cache from multiple threads. It's very hard to test for concurrency bugs, so I'm trying to read write to same key multiple times from many different threads. This is my driver code:

```swift
func taskRunner() {
    let tasks = 20
    let jobs = 100000
    let cache = Cache()

    let startTime = CACurrentMediaTime()
    DispatchQueue.concurrentPerform(iterations: tasks) { task in
        for job in 0..<jobs {
            let currValue = cache.get(key: "shared-key") ?? 0
            cache.set(value: currValue + 1, key: "shared-key")
        }
    }
    let endTime = CACurrentMediaTime()
    print("finish \(endTime - startTime)")
}
```

When I run this code, I get a runtime exception at some point. Since, I'm running the code with **Thread Sanitizer**, the error is more readable:

```
WARNING: ThreadSanitizer: Swift access race (pid=34398)
  Read of size 8 at 0x7b080003aa50 by thread T2:
  Previous modifying access of Swift variable at 0x7b080003aa50 by thread T1:
  Location is heap block of size 24 at 0x7b080003aa40 allocated by main thread:
  Thread T2 (tid=455657, running) is a GCD worker thread
  Thread T1 (tid=455656, running) is a GCD worker thread
SUMMARY: ThreadSanitizer: Swift access race
```

Looks like we have a classic race condition, as expected. Let's fix it.

### Blocking Reads and Writes

An improvement could be to wrap all the reads and writes to the shared memory location in a serial queue.

```swift
class Cache {
    private let queue = DispatchQueue(label: "serial-queue")
    private var store: [String: Int] = [:]

    func get(key: String) -> Int? {
        return queue.sync {
            return store[key]
        }
    }

    func set(value: Int, key: String) {
        queue.sync {
            store[key] = value
        }
    }
}
```

Here's the output, to confirm that this code does indeed work:
```
finish 26.025777226001082
```

It works, but it has a few problems. First, `DispatchQueue.sync` is deadlock prone. It's very easy to dispatch sync on the same queue causing a deadlock. Second, since we are using a serial queue to synchronize all the read/writes, we are not actually achieving any parallelism. We can definitely improve the performance if we making the reads and writes in parallel.

### Asynchronous Reads and Writes

The first solution could be to use a concurrent queue and dispatch every read write operation to that queue.

```swift
class Cache {
    private let queue = DispatchQueue(label: "concurrent-queue", attributes: .concurrent)
    private var store: [String: Int] = [:]

    func get(key: String, completion: @escaping (Int?) -> Void) {
        queue.async { [weak self] in
            completion(self?.store[key])
        }
    }

    func set(value: Int, key: String) {
        queue.async { [weak self] in
            self?.store[key] = value
        }
    }
}
```

But this has the same problems as our original solution. It is prone to race conditions when writing to `store`. Since the `queue` is concurrent, so multiple writes can be happening in parallel.

An improvement could be to use memory barriers to synchronize reads and writes.

```swift
class Cache {
    private let queue = DispatchQueue(label: "concurrent-queue", attributes: .concurrent)
    private var store: [String: Int] = [:]

    func get(key: String, completion: @escaping (Int?) -> Void) {
        queue.async(flags: .barrier) { [weak self] in
            completion(self?.store[key])
        }
    }

    func set(value: Int, key: String) {
        queue.async(flags: .barrier) { [weak self] in
            self?.store[key] = value
        }
    }
}
```

Although now our memory access should be again race free, but it might be making things works worse in reality. Due to the fact that barrier blocks the entire concurrent queue we might get even worse time than using a serial queue as we are using a concurrent queue as a serial queue.

### Synchronous Reads Asynchronous Writes

An observation could be made from our above solution that maybe we do not need barrier for reading tasks. And which brings to another solution which might the best so far.

```swift
class Cache {
    private let queue = DispatchQueue(label: "concurrent-queue", attributes: .concurrent)
    private var store: [String: Int] = [:]

    func get(key: String) -> Int? {
        return queue.sync { 
            return store[key]
        }
    }

    func set(value: Int, key: String) {
        queue.async(flags: .barrier) { [weak self] in
            self?.store[key] = value
        }
    }
}
```

### Beyond libdispatch

libdispatch is not a low level library. It is probably a bit lower level than `NSOperation`, but it's not designed for performance, rather usage simplicity. If performance is the issue we can always go a bit more further by using few more tools provided by `Foundation`, like say `NSLock` to get more control over things.

```swift
class Cache {
    private let lock = NSLock()
    private var store: [String: Int] = [:]

    func get(key: String) -> Int? {
        lock.lock()
        let value = store[key]
        lock.unlock()
        return value
    }

    func set(value: Int, key: String) {
        lock.lock()
        store[key] = value
        lock.unlock()
    }
}
```

One of the best resources on GCD are:

1. [Official Using Grand Central Dispatch tutorial](https://apple.github.io/swift-corelibs-libdispatch/tutorial/)
1. [Mastering Grand Central Dispatch - WWDC 2011 video](https://developer.apple.com/videos/play/wwdc2011/210/)
