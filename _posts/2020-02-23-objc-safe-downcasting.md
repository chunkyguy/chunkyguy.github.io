---
layout: post
title:  "Objective-C safe downcasting"
date:   2020-02-23 14:34:00 +0200
categories: objc swift
published: true
---

Swift has this nice concept of optional chaining which is used in a lot of places. One really good use case is when down casting a type the operation becomes a no-op if casting to an incorrect sibling. To illustrate take a look at this Swift example code:

```swift
class FruitCompany {
  static var apple: FruitCompany { Apple() }
  static var orange: FruitCompany { Orange() }
}

class Apple: FruitCompany {
  func makeMoney() { print("💰") }
}

class Orange: FruitCompany {}

func makeMoney(with fruitCompany: FruitCompany) {
  (fruitCompany as? Apple)?.makeMoney()
}

func testOptional() {
  makeMoney(with: FruitCompany.apple) // Prints
  makeMoney(with: FruitCompany.orange) // Does nothing
}
```

Let's see how this behaves in Objective-C

```objc
@interface Apple : FruitCompany
- (void)makeMoney;
@end

@interface Orange : FruitCompany
@end

@interface FruitCompany : NSObject
+ (instancetype)apple;
+ (instancetype)orange;
@end

@implementation FruitCompany
+ (instancetype)apple { return [Apple new]; }
+ (instancetype)orange { return [Orange new]; }
@end

@implementation Apple
- (void)makeMoney { NSLog(@"💰"); }
@end

@implementation Orange
@end
```

```objc
- (void)makeMoneyWithFruitCompany:(FruitCompany *)company
{
  [(Apple *)company makeMoney];
}

- (void)testOptional
{
  [self makeMoneyWithFruitCompany:[FruitCompany apple]]; // Prints
  [self makeMoneyWithFruitCompany:[FruitCompany orange]]; // Crash !!
}
```

A crash! This is the crash log

```
*** Terminating app due to uncaught exception 'NSInvalidArgumentException', 
    reason: '-[Orange makeMoney]: unrecognized selector sent to instance 0x6000024f8110'
```

Since Objective-C supports passing message to `nil`, so if we explicitly do a down casting to `nil` it works

```objc
- (void)makeMoneyWithFruitCompany:(FruitCompany *)company
{
  Apple *apple = [company isKindOfClass:[Apple class]] ? (Apple *)company : nil;
  [apple makeMoney];
}
```

So, how can we make this more aligned with Swift like. The first step would be to implement a *Category* on `NSObject` that does the casting magic.

```objc
@interface NSObject (WLSafeCast)
- (id)wl_safeCastToClass:(Class)aClass;
@end

@implementation NSObject (WLSafeCast)

- (id)wl_safeCastToClass:(Class)aClass;
{
  return [self isKindOfClass:aClass] ? self : nil;
}

@end
```

With this one can already start using it like

```objc
- (void)makeMoneyWithFruitCompany:(FruitCompany *)company
{
  Apple *apple = [company wl_safeCastToClass:[Apple class]];
  [apple makeMoney];
}
```

Next step we can implement the C MACRO

```c
#define CAST_OR_NIL(v, T) [v wl_safeCastToClass:[T class]]
```

With that we can start using the Swift like down casting

```objc
- (void)makeMoneyWithFruitCompany:(FruitCompany *)company
{
  Apple *apple = CAST_OR_NIL(company, Apple);
  [apple makeMoney];
}
```

Or can even be reduced to one line

```objc
- (void)makeMoneyWithFruitCompany:(FruitCompany *)company
{
  [CAST_OR_NIL(company, Apple) makeMoney];
}
```

Do you know a better way? Please [share your thought](https://www.reddit.com/r/ObjectiveC/comments/f89xf5/objectivec_safe_downcasting/) or let me know on [twitter](https://twitter.com/chunkyguy)