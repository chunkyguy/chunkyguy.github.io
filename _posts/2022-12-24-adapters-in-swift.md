---
layout: post
title:  "Adapters in Swift"
date:   2022-12-24 15:10:00 +0200
categories: architecture combine swift ios
published: true
---
Any professional software developer knows that the simplest solution in building software is sometimes just to wrap an existing solution with the expected client interface and be done with it. This pattern is so common they even gave it a name, [**Adapter Pattern**](https://en.wikipedia.org/wiki/Adapter_pattern).

In such a pattern there are mostly three pieces in play, the *Adapter*, *Adaptee* and the *Client*. With *Adaptee* being the existing solution out there that we wish to wrap and reuse, and *Client* being the user of the API, the only moving part in the equation is then the *Adapter*, which is the only thing in focus.

![Adapter Pattern]({{ site.url}}/assets/swift-adapter-pattern/01.png)

If you would like to do it the hard way, you can obviously painfully implement every piece of the interface in terms of the internal implementation details provided by the *Adaptee*.

To illustrate, if we wanted to wrap the `CurrentValueSubject` from `Combine` framework into something which we would like to call as `Variable` with the point being that we don't want to bother our client with any `Error` type. 

We could simply create our *Adapter* as:

```swift
class Variable<T> {
    private let subject: CurrentValueSubject<T, Never>
    
    var value: T {
        get { subject.value }
        set { subject.value = newValue }
    }
    
    var stream: AnyPublisher<T, Never> {
        return subject.eraseToAnyPublisher()
    }
    
    init(_ initValue: T) {
        self.subject = CurrentValueSubject(initValue)
    }
}
```

Here's one usage of this *Adapter*

```swift
class Client {
    var cancellable: AnyCancellable?
    let variable = Variable(0)
    
    init() {
        cancellable = variable.stream.sink { 
            print("recv: \($0)") 
        }
    }

    func run() {
        print("before: \(variable.value)")
        variable.value = 10
        print("after: \(variable.value)")
    }
}
```

But this usage becomes verbose very easily when we start using more complex data structures, like:

```swift
struct User {
    var fullName: String {
        [firstName, lastName].joined(separator: " ")
    }
    var firstName = "Steve"
    var lastName = "Jobs"
}
```

```swift
class Client {
    var cancellable: AnyCancellable?
    let user = Variable(User())
    
    init() {
        cancellable = user.stream.sink { 
            print("recv: \($0.fullName)") 
        }
    }

    func run() {
        print("before: \(user.value.fullName)")
        user.value.lastName = "Carell"
        print("after: \(user.value.fullName)")
    }
}
```

Notice the verbosity with `user.value.xxx` usage. Wouldn't it be nice if we could simply access the property with `user.xxx`

In Objective-C this pattern could be simplified by implementing with built-in runtime behavior like *Message Forwarding*. But even in Swift there's another way to implement this pattern, using the `@dynamicMemberLookup`. 

With `@dynamicMemberLookup` compiler can provide direct access to the properties of an object via `KeyPath<Root, Value>`, where `Root` is the receiver and `Value` is the property. So we can rewrite our *Adapter* as:

```swift
@dynamicMemberLookup
class Variable<T> {
    subscript<U>(dynamicMember keyPath: KeyPath<T, U>) -> U {
        return subject.value[keyPath: keyPath]
    }

    subscript<U>(dynamicMember keyPath: WritableKeyPath<T, U>) -> U {
        get { subject.value[keyPath: keyPath] }
        set { subject.value[keyPath: keyPath] = newValue }
    }
    
    var stream: AnyPublisher<T, Never> {
        return subject.eraseToAnyPublisher()
    }

    private let subject: CurrentValueSubject<T, Never>

    init(_ startValue: T) {
        self.subject = CurrentValueSubject(startValue)
    }
}
```

And with this our call site becomes a lot cleaner

```swift
class Client {
    var cancellable: AnyCancellable?
    let user = Variable(User())
    
    init() {
        cancellable = user.stream.sink { 
            print("recv: \($0.fullName)") 
        }
    }

    func run() {
        print("before: \(user.fullName)")
        user.lastName = "Carell"
        print("after: \(user.fullName)")
    }
}
```

If we further wish to make our call site even fancier, we can add support for `@propertyWrapper`

```swift
@dynamicMemberLookup @propertyWrapper
class Variable<T> {
    var wrappedValue: T {
        get { subject.value }
        set { subject.value = newValue }
    }
    
    subscript<U>(dynamicMember keyPath: KeyPath<T, U>) -> U {
        return subject.value[keyPath: keyPath]
    }

    subscript<U>(dynamicMember keyPath: WritableKeyPath<T, U>) -> U {
        get { subject.value[keyPath: keyPath] }
        set { subject.value[keyPath: keyPath] = newValue }
    }
    
    var stream: AnyPublisher<T, Never> {
        return subject.eraseToAnyPublisher()
    }
    
    private let subject: CurrentValueSubject<T, Never>
    
    init(wrappedValue value: T) {
        self.subject = CurrentValueSubject(value)
    }
}
```

There's is lot of hidden magic here that only happens when you know the secret code. The code here is to provide a property called `wrappedValue` and an initializer called `init(wrappedValue:)`. With that in place our call site looks even prettier.

```swift
class Client {
    @Variable var user: User = User()

    func run() {
        print("before: \(user.fullName)")
        user.lastName = "Carell"
        print("after: \(user.fullName)")
    }
}

```

To add back the support to the `stream`, we can use another secret code, that is providing a property called `projectedValue`

```swift
@dynamicMemberLookup @propertyWrapper
class Variable<T> {
    var projectedValue: AnyPublisher<T, Never> {
        return subject.eraseToAnyPublisher()
    }
    
    var wrappedValue: T {
        get { subject.value }
        set { subject.value = newValue }
    }
    
    subscript<U>(dynamicMember keyPath: KeyPath<T, U>) -> U {
        return subject.value[keyPath: keyPath]
    }

    subscript<U>(dynamicMember keyPath: WritableKeyPath<T, U>) -> U {
        get { subject.value[keyPath: keyPath] }
        set { subject.value[keyPath: keyPath] = newValue }
    }
    
    private let subject: CurrentValueSubject<T, Never>
    
    init(wrappedValue value: T) {
        self.subject = CurrentValueSubject(value)
    }
}
```

This can add another super power to our *Adapter* that can be accessed via the magical `$` sign

```swift
class Client {
    var cancellable: AnyCancellable?
    @Variable var user: User = User()

    init() {
        cancellable = $user.sink { 
            print("recv: \($0.fullName)") 
        }
    }

    func run() {
        print("before: \(user.fullName)")
        user.lastName = "Carell"
        print("after: \(user.fullName)")
    }
}
```

## Conclusion
As we just saw Swift has evolved long since the days of being strict compiled language with all rules being embeded within some contracts bounded by `Protocol`. There is a lot of magical stuff that can be achieved with things that you to must first learn from elsewhere, like the `$` access via `projectedValue` or the magical `KeyPath` based access. But we can use this to simplify our APIs and make everyone a bit more happier.