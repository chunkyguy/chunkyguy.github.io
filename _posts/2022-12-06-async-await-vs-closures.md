---
layout: post
title:  "Swift async-await vs closures"
date:   2022-12-06 11:09:00 +0200
categories: swift async await concurrency
published: true
---
Swift `await` works by capturing the context and suspending the execution until the called `async` method returns. Another thing that works similarly by capturing the surrounding context is an escaping closure. So `async-await` calls can be imagined as equivalent to escaping closure. Whenever you see a method with `async` mentally replace that method with a escaping completion handler.

```swift
func run() async
func run(_ completion: @escaping () -> Void)
```

## Examples
A simple scenario with equivalent code

```swift
class Foo {
  func before() {}
  func after() {}

  func doStuff() async {}
  func doStuff(_ completion: @escaping () -> Void) {}
}
```

```swift
extension Foo {
  func run() async {
    before()
    await doStuff()
    after()
  }
}
```

```swift
extension Foo {
  func run(_ completion: @escaping () -> Void) {
    before()
    doStuff {
      self.after()
      completion()
    }
  }
}
```

And then it only makes sense how the return values work

```swift
func doStuff() async -> Int

let ret = await doStuff()
print("\(ret)")
```

```swift
func doStuff(_ completion: @escaping (Int) -> Void)

doStuff { ret in
    print("\(ret)")
}

```

Or the error propagation

```swift
func div(_ a: Int, _ b: Int) async throws -> Double {
  if b == 0 {
    throw FooError.divideByZero
  }
  return Double(a) / Double(b)
}

do {
  let ret = try await div(1, 0)
  print("\(ret)")
} catch {
  print("\(error)")
}
```

```swift
func div(_ a: Int, _ b: Int, _ completion: @escaping (Double) -> Void) throws {
  if b == 0 {
    throw FooError.divideByZero
  }
  completion(Double(a) / Double(b))
}

do {
  try div(1, 0) { ret in
    print("\(ret)")
  }
} catch {
  print("\(error)")
}
```

## Converting async to sync
Notice how the `async` call naturally propagates to the callee. So a call to `doStuff() async` makes `run() async` as well. This also makes sense for completion handers in most cases. And if we wish to not propagate the asynchronous behavior, or in other words, we wish to convert an `async` method into a `sync` method, we need to use a `Task` which takes in a completion handler and is equivalent to wrapping within `DispatchQueue.async { ... }`

```swift
func run() {
  before()
  Task {
    await doStuff()
    after()
  }
}
```
```swift
func run() {
  before()
  DispatchQueue.global().async {
    self.doStuff {
      self.after()
    }
  }
}
```

## Chaining calls
If we do wish to call async methods one after the other, the mental model remains the same

```swift
func doStuff() async {}
func doMoreStuff() async {}

before()
await doStuff()
await doMoreStuff()
after()
```

```swift
func doStuff(_ completion: @escaping () -> Void) {}
func doMoreStuff(_ completion: @escaping () -> Void) {}

before()
doStuff {
  self.doMoreStuff {
    self.after()
  }
}
```

But if the tasks need to be run in parallel the `async-await` provides a construct with `async let`

```swift
func run() async {
  before()
  async let task1: Void = doStuff()
  async let task2: Void = doMoreStuff()
  _ = await [task1, task2]
  after()
}
```

This is then equivalent to using `DispatchGroup`

```swift
func run(_ completion: @escaping () -> Void) {
  before()
  let group = DispatchGroup()
  group.enter()
  doStuff { group.leave() }
  group.enter()
  doMoreStuff { group.leave() }
  group.notify(queue: DispatchQueue.global()) {
    self.after()
    completion()
  }
}
```

## Conclusion
Having a better mental model for async-await helps with appreciating what sort of pitfalls it saves us from, and also how to migrate from completion handlers to async await.

## Further Reading
1. [Swift Programming Language - Concurrency](https://docs.swift.org/swift-book/LanguageGuide/Concurrency.html)
1. [WWDC 2021 - Swift concurrency: Update a sample app](https://developer.apple.com/wwdc21/10194)
1. [WWDC 2021 - Swift Concurrency: Behind the Scenes](https://developer.apple.com/wwdc21/10254)