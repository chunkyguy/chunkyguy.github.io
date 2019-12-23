---
layout: post
title: "Step by step guide towards type erasures in Swift"
date:   2019-12-13 11:15:00 +0200
categories: swift design-patterns
published: true
---

There are 2 kind of programmers that you encounter in the wild. Ones those who really like **types** and the others those really don't. Forget arguments over spaces vs tabs, this is the real debate. This is a big deal. Probably the first thing engineers think when starting on a new project. Perhaps even before they think about the real goal of the project itself. Like, "I need a weekend project where I can use python". Type is a serious thing.

So, naturally when a group of expert C++ programmers got together to write a new language they came up with a language which is even more complex than C++ as long as the type system is concerned. Swift is very strictly typed, undeniably the most strongly typed language I've ever used. I mean even natural looking type conversions, say between `Int` and `UInt`, have to be explicitly declared.

When you're dealing with such strongly typed system for solving real world problems there comes quite a few times a moment where you have a bunch of types that are not strictly the same but still you can group them by some shared functionality. And if you're a Swift developer youself, you must be thinking 'Ah `protocol`!'.

### Problem

In Swift `protocol` can be used in 2 different paradigms. 

First, abstract base class

```swift
protocol Beverage {
  var name: String { get }
}

struct Beer: Beverage {
  let name = "Duff"
}
```

Second, to represent a generic concept

```swift
protocol Alcoholic {
  associatedtype Drink: Beverage
  func gulp(drink: Drink)
}

struct Homer<B: Beverage>: Alcoholic {
  func gulp(drink: B) {
    print("Hmmm \(drink.name)")
  }
}

struct Barney<B: Beverage>: Alcoholic {
  func gulp(drink: B) {
    print("Yay \(drink.name)")
  }
}
```

If you've ever written any C++ you must be well aware of the fact that C++ does not provide any language construct for the second usage of `protocol`. So we end up having similar function signatures that are still checked by the compiler but not available as a documentation for users of the library. Like the `size()` member function of every collection type in the standard library. If you wish you write your own new collection type and want it to fit with the rest of the standard library, you just somehow have to know [what methods to implement](https://stackoverflow.com/a/22299553) to keep the compiler from screaming.

With Swift `protocol`s the most noticeable difference is when attempting to group various types under a common type.

```swift
// Abstract base class: Works!
func collect(beverages: [Beverage]) {}

// Concept: Fails!
// Protocol 'Alcoholic' can only be used as a generic constraint because it has Self or associated type requirements
func enterMoTavern(customers: [Alcoholic]) {}
```
The workaround for this second case is what type erasure is all about. This is my strategy.

### Step 1: Interface

Create a dumb class that only needs to serve as the interface for our concept, or in other words an abstract base class.

```swift
class Interface<B: Beverage>: Alcoholic {
  func gulp(drink: B) {
    fatalError()
  }
}
```

### Step 2: Implementation

Next, create a dumb implementation class. This class needs to be a subclass of the `Interface` as later we are going to use the famous [PIMPL](https://en.cppreference.com/w/cpp/language/pimpl) pattern

```swift
class Implementation: Interface<SomeAlcoholic.Drink> { }

let proxy: Interface = Implementation()
```

As far as the actual implementation goes, this class only needs to capture an concrete instance and forward all the calls to that. To make that happen we need to parametrize this class for a type conforming to the concept.

```swift
class Implementation<SomeAlcoholic: Alcoholic>: Interface<SomeAlcoholic.Drink> {

  private let concreteInstance: SomeAlcoholic

  init(_ concreteInstance: SomeAlcoholic) {
    self.concreteInstance = concreteInstance
  }

  override func gulp(drink: SomeAlcoholic.Drink) {
    concreteInstance.gulp(drink: drink)
  }
}
```

The rule to remember here is that `Interface` conforms to `Alcoholic` while `Implementation` does not. `Implementation` simply provides overridden implementations of `Alcoholic`.

### Step 3: Wrap it up

The final step is to glue both `Interface` and `Implementation` together. 

```swift
class AnyAlcoholic<B: Beverage>: Alcoholic {

  private let proxy: Interface<B>

  init<SomeAlcoholic: Alcoholic>(_ concreteInstance: SomeAlcoholic) where SomeAlcoholic.Drink == B {
    self.proxy = Implementation(concreteInstance)
  }

  func gulp(drink: B) {
    proxy.gulp(drink: drink)
  }
}
```

The point to note here is that we are templating on abstract base class `Beverage` and conforming to the concept `Alcoholic` magically. So that we can easily pass `AnyAlcoholic` as a complete type.

```swift
let homer: Homer<Beer> = Homer()
let barney: Barney<Beer> = Barney()
let customers: [AnyAlcoholic<Beer>] = [AnyAlcoholic(homer), AnyAlcoholic(barney)]
```

With this we can finally have a type that can be used to group every concept conforming type.

```swift
func enterMoesTavern(customers: [AnyAlcoholic<Beer>]) {
  let beer = Beer()
  customers.forEach { $0.gulp(drink: beer) }
}

enterMoesTavern(customers: [AnyAlcoholic(Homer()), AnyAlcoholic(Barney())])
```

### Step 4: Clean up

To hide all the ugliness, we can make all of our boilerplate code as nested private classes.

```swift
class AnyAlcoholic<B: Beverage>: Alcoholic {

  private class Interface<B: Beverage>: Alcoholic {
    func gulp(drink: B) {
      fatalError()
    }
  }

  private class Implementation<SomeAlcoholic: Alcoholic>: Interface<SomeAlcoholic.Drink> {

    private let concreteInstance: SomeAlcoholic

    init(_ concreteInstance: SomeAlcoholic) {
      self.concreteInstance = concreteInstance
    }

    override func gulp(drink: SomeAlcoholic.Drink) {
      concreteInstance.gulp(drink: drink)
    }
  }

  private let proxy: Interface<B>

  init<SomeAlcoholic: Alcoholic>(_ concreteInstance: SomeAlcoholic) where SomeAlcoholic.Drink == B {
    self.proxy = Implementation(concreteInstance)
  }

  func gulp(drink: B) {
    proxy.gulp(drink: drink)
  }
}
```

So our exposed interface is like:

```swift
class AnyAlcoholic<B: Beverage> : Alcoholic {

  init<SomeAlcoholic : Alcoholic>(_ concreteInstance: SomeAlcoholic) where SomeAlcoholic.Drink == B

  func gulp(drink: B)
}
```

### Closing notes

I think one of the reasons people love Swift is because it has a pretty nice exponential learning curve

![img](https://www.graphpad.com/guides/prism/7/curve-fitting/_bm91.png)

What this means is that it is very easy to learn the basics of the language, which is great for beginners. While, the language complexity keeps increasing the more you get into it, which keeps experienced Swift developers entertained. Of course untill you hit the roof. That's when you discover that the [feature you badly need](https://github.com/apple/swift/blob/master/docs/GenericsManifesto.md#variadic-generics) is still in proposal state of the evolution, which might help with keeping things interesting for a while. 