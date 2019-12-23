---
layout: post
title:  "Service Locator pattern with GameplayKit"
date:   2019-10-23 00:18:00 +0200
categories: architecture ios
published: true
---

- [Service Locator Pattern](#service-locator-pattern)
- [Entity Component System](#entity-component-system)
- [Hello GameplayKit](#hello-gameplaykit)
- [Encapsulation](#encapsulation)
- [And more](#and-more)

### Service Locator Pattern

In the beginning everything seems fine. You start a new project. Things talk to each other, and there are so few participants and so less information that has to be shared among them. Next arrives a situation where a resource has to be shared among several objects. The shared resource could be a service that stores some app data, or could be some collection of utility functions wrapped under a single blanket. We can think of these shared resources as dependencies that need to be injected from outside into every one of those objects as their lifespan is controlled on a different level. 

As the code size increases, so does the problem of dependency injection. `Bob` needs to pass a dependency to `Alice`, but has no direct access to `Alice`. So, ends up handing over the dependencies to every thing in the chain of communication. And very soon a lot of objects has access to dependencies that do not need themselves. 

A very common solution that I often come across is to use the **Singleton Pattern**. Implemented usually by providing a shared instance that can then easily be accessed from anywhere in your code. This pattern is [very well covered by Apple](https://developer.apple.com/documentation/swift/cocoa_design_patterns/managing_a_shared_resource_using_a_singleton) too.

This indeed does the job but not without creating a mess. The mess is felt the most when trying to unit test a functionality where there is a need to mock a service, but can't since there no clear way to inject the mock instance. Another place where the **Singleton Pattern** hurts the most is with initialization order, where 2 singleton instances can accidentally become dependent on each other and end up in a infinite loop.

```swift
class ServiceA {
     static let sharedInstance: ServiceA = {
        let instance = ServiceA(that: ServiceB.sharedInstance.that)
           return instance
       }()

    var this: String { return "this" }

    init(that: String) {}
}

class ServiceB {
    static let sharedInstance: ServiceB = {
        let instance = ServiceB(this: ServiceA.sharedInstance.this)
           return instance
       }()

    var that: String { return "that" }

    init(this: String) {}
}
```

Another good alternative to this pattern is to use what is commonly called as [**Service Locator Pattern**](https://en.wikipedia.org/wiki/Service_locator_pattern). The basic idea is that there is this one instance that manages all the global services. When the system starts, we register all the services with this service locator, preferably during the initialization. Later, anyone who wishes to get the shared resource simply asks for the service from this service locator.

It's not a golden solution to all your problems, but at least helps with unit testing where we can register our mock services before running the tests. And also since the service locator owns the dependencies, they are easier to manage. Including releasing a service when done. Which is hard (if not impossible) with **Singleton Pattern**.

### Entity Component System

And now for something completely different, let's talk about the **Entity Component System**. A design pattern held in high regards by many game developers big and small.

Most of the games need at least these three kinds of object types at some point of development cycle:

1. **Render objects**: Think of the background image that only needs to be rendered on screen.
1. **Player objects**: Every gameplay character. Needs to be rendered on screen and also interact with each other or the environment; So also need a physics body.
1. **Trigger objects**: Something that does not need any rendering, but still interacts with the rest of the physics bodies. Like the invisible bonus or when character drops out of screen.

One way to implement this could be have a classic object hierarchy that all inherit from something like a `GameObject` all the way down:

```swift
class GameObject { }

class RenderObject: GameObject {
    func draw() { }
}

class PhysicsObject: RenderObject {
    func update() { }
}

class PlayerObject: PhysicsObject {
    func collectCoins() { }
}
```

Then we create our instances as:

```swift
let background = RenderObject()
let hiddenBonus = PhysicsObject()
let mario = PlayerObject()

background.draw()

hiddenBonus.update()

mario.collectCoins()
mario.update()
mario.draw()
```

This would work fine except the fact that now `hiddenBonus` also has a `draw()` function, which does nothing. Also notice that every time we introduce another behavior, like say user interaction, we would have to come up with some strategy to update the inheritance hierarchy so that all the subclasses can get the behavior. This is a hard problem to solve. Harder if the game engine and the game live in entirely different layers.

A good flexible solution for this problem is [Entity Component System](https://en.wikipedia.org/wiki/Entity_component_system). The basic idea is to wrap every possible reusable behavior in isolation and let's call it a `Component`.

So, in our case above we could have 2 components:

```swift
class Component {}

class RenderComponent: Component {
    func draw() { }
}

class PhysicsComponent: Component {
    func update() { }
}
```
And next we can have something called as `Entity`. An `Entity` can be composed with several behavior components.

```swift
class Entity {
    private var components: [Component]

    init(components: [Component]) {
        self.components = components
    }

    func add(component: Component) { }

    func getComponent(ofType: Component.Type) -> Component? { }
}
```

And now we can create all the desired instances as:

```swift
let background = Entity(components: [RenderComponent()])
let hiddenBonus = Entity(components: [PhysicsComponent()])
let mario = Entity(components: [RenderComponent(), PhysicsComponent()])
```

### Hello GameplayKit

Okay that sounds good. But how does it fit with our topic? 

If you squint your eyes enough you would realize that this **Entity Component System** is a **Service Locator Pattern** in disguise. Where a service locator is `Entity` and the services are the `Component` registered with the `Entity`.

What's even better, Apple already provides us a nice framework which implements the **Entity Component System** it's called [GameplayKit](https://developer.apple.com/documentation/gameplaykit). `GameplayKit` provides all the boilerplate code we need. Although probably not originally designed for this task, but we can definitely build our **Service Locator Pattern** with this framework. 

Let's say we want to have a service that manages all the feature flags stored on the device. We can implement it as a `GKComponent`:

```swift
import GameplayKit

class FeatureFlagService: GKComponent {
    let store: UserDefaults

    init(userDefaults: UserDefaults = .standard) {
        self.store = userDefaults
        super.init()
    }

    func set(value: Bool, key: String) {
        store.set(value, forKey: key)
    }

    func value(key: String) -> Bool {
        return store.bool(forKey: key)
    }
}
```

Next we can implement our `ServiceLocator` wrapped around a `GKEntity`

```swift
class ServiceLocator {
    private let registry = GKEntity()

    func register(service: GKComponent) {
        registry.addComponent(service)
    }

    var featureFlagService: FeatureFlagService? {
        return registry.component(ofType: FeatureFlagService.self)
    }
}
```

And then we can simply use the `ServiceLocator` to find dependencies

```swift
// at booting time
let serviceLocator = ServiceLocator()
serviceLocator.register(service: FeatureFlagService())

// a few seconds later
serviceLocator.featureFlagService?.set(value: true, key: "show-alt-login")
print(String(describing: serviceLocator.featureFlagService?.value(key: "show-alt-login")))
```

Another `GameplayKit` functionality that we can exploit is for scenarios where we want to perform some action whenever the service is actually registered with the service locator, rather than at the allocation/initialization time. For example, we want to fetch the feature flag data from our server whenever the `FeatureFlagService` is actually registered with the `ServiceLocator` for a faster launch time. With `GameplayKit` we can achieve these tasks by overriding the `didAddToEntity` method of `GKComponent`

```swift
extension FeatureFlagService {
    override func didAddToEntity() {
        // TODO: fetch data and synchronize with local store
        store.synchronize()
    }
}
```

### Encapsulation

If we do not want to expose the `GameplayKit` dependency outside. Or if you want to use "pure" swift classes, structs to represent a Component. We can add another level of indirection to hide the abstraction.

We can have a generic wrapper that wraps any service

```swift
private class ServiceWrapper<T>: GKComponent {
    let service: T

    init(service: T) {
        self.service = service
        super.init()
    }

    required init?(coder: NSCoder) {
        fatalError("init(coder:) has not been implemented")
    }
}
```

And then our `ServiceLocator` can perform one extra boxing/unboxing:

```swift
class ServiceLocator {
    private let registry = GKEntity()

    func register<T>(service: T) {
        registry.addComponent(ServiceWrapper(service: service))
    }

    func get<T>() -> T? {
        registry.component(ofType: ServiceWrapper<T>.self)?.service
    }

    var featureFlagService: FeatureFlagService? { return get() }
    var networkService: NetworkService? { return get() }
    var accessibilityService: AccessibilityService?  { return get() }
}

```

### And more

And that is not all. There is much more available in `GameplayKit` that can be useful when implementing a `Service Locator Pattern`. Like the `GKEntity.removeComponent(ofType:)` or `GKComponent.willRemoveFromEntity()`. There is also a section in the [GameplayKit programming guide](https://developer.apple.com/library/archive/documentation/General/Conceptual/GameplayKit_Guide/EntityComponent.html#//apple_ref/doc/uid/TP40015172-CH6) that covers the Entity Component System I was talking about earlier.

Although initially `GameplayKit` does sounds like an unusual fit for the **Service Locator** problem. But rather than writing your own solution from scratch, or using anu external dependency, I find `GameplayKit` a better for the job. For one, it is one less thing to maintain, and second it is already there ready for use. So, the next time you are looking for a Service Locator Pattern, try `GameplayKit`.