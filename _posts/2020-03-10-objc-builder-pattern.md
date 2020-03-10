---
layout: post
title:  "Objective-C builder pattern"
date:   2020-03-09 23:30:54 +0200
categories: objc design-pattern
published: true
---

It's not an secret that creating instances in Objective-C can be very verbose. It doesn't help that we don't have default values and by default everything is initialized to zero. It gets even more verbose when you want to write a robust code and don't want to expose properties as `readwrite` where they shouldn't be. That said, a lot of times keeping them `readwrite` is better than wiring things around it. Remember that properties in Objective-C, unlike say C++, are just syntactic sugar over getters and setters.

Often times we need an object with all properties as `readonly` as they are not expected to be modified after the initialization.

```objc
@interface Foo : NSObject
@property (nonatomic, readonly) NSString *title;
@property (nonatomic, readonly) Bar *bar;
@end
```

Clean. But then comes the problem of initializing this class. There are a few options here.

## Simple alloc/init

Let the clients deal with it by calling `alloc`, `init` and setting the property directly. This would mean that we need to keep the properties as `readwrite`

```objc
@interface Foo : NSObject
@property (nonatomic, copy) NSString *title;
@property (nonatomic, strong) Bar *bar;
@end
```

```objc
- (void)useFoo
{
  Foo *foo = [[Foo alloc] init];
  foo.title = @"hello";
  foo.bar = [[Bar alloc] init];

  NSLog(@"%@", foo.title);
  NSLog(@"%@", foo.bar);
}
```

This is probably the simplest solution out there, needs no extra care. But this would mean that anyone with access to the instance can update the properties anytime they like.

## Custom Initializers

This is probably the most common way of creating instances with Objective-C. The idea is pretty simple, expose an initializer where clients can fill in all the properties, and all the properties at the interface are declared to be `readonly`

```objc
// Foo.h
@interface Foo : NSObject
- (instancetype)initWithTitle:(NSString *)title
                         bar:(Bar *)bar;
@property (nonatomic, readonly) NSString *title;
@property (nonatomic, readonly) Bar *bar;
@end
```

And provide a `readwrite` override in the implementation details.

```objc
// Foo.m
@interface Foo ()
@property (nonatomic, copy) NSString *title;
@property (nonatomic, strong) Bar *bar;
@end

@implementation Foo
- (instancetype)initWithTitle:(NSString *)title
                          bar:(Bar *)bar
{
  self = [super init];
  if (self) {
    _title = [title copy];
    _bar = bar;
  }
  return self;
}
@end
```

This solution is good for as long as not dealing with properties with default values. Imagine a initializer with a lot of properties and every time we have to call it, we have to provide the default values to properties. One standard solution is to provide telescope initializers, where an convenience initializer calls another initializer with some default value filled in until a designated initializer is invoked at the end of the pipeline which does the actual initialization


```objc
- (instancetype)initWithBar:(Bar *)bar;
{
  return [self initWithTitle:@"title" bar:bar];
}

- (instancetype)initWithTitle:(NSString *)title;
{
  return [self initWithTitle:title bar:[Bar bar]];
}

- (instancetype)initWithTitle:(NSString *)title
                          bar:(Bar *)bar 
{ 
  // usual stuff here ...
}
```

Again, works fine when the number of properties is not huge. These convenience initializers grow in size for every added property. And also sometimes it does not make clear sense which initializer to invoke, specially true if we need a selected subset of optional properties to be populated, we would need total initializers for every permutation. 

Here's chart of number of initializers we would need for a given number of properties

```
properties: 2, initializers: 3
properties: 3, initializers: 7
properties: 4, initializers: 15
properties: 5, initializers: 31
properties: 6, initializers: 63
properties: 7, initializers: 127
properties: 8, initializers: 255
properties: 9, initializers: 511
properties: 10, initializers: 1023
```

Woah! At first glance this does not really a scalable solution for sure! The good news is that usually we don't need all of these initializers, since some properties only make sense together. Mostly we care to provide 2 or 3 initializers that are often used and let the client use the designated verbose one if case they need to have more control.

## Subclass Factory Pattern

Going back to the first solution, I did like it a lot for the reason that it was pretty straightforward. The instance can set default values in the `init` and we explicitly override any properties we like

```objc
@interface Foo : NSObject
@property (nonatomic, copy) NSString *title;
@property (nonatomic, strong) Bar *bar;
@end

@implementation Foo
- (instancetype)init
{
  self = [super init];
  if (self) {
    _title = @"Default";
    _bar = [[Bar alloc] init];
  }
  return self;
}
@end
```

The only problem with this approach is that the properties remain `readwrite` forever. The technique I use is to have a factory subclass that has `readwrite` properties and the actual class only exposes the properties as `readonly`

```objc
@interface Foo : NSObject
@property (nonatomic, readonly) NSString *title;
@property (nonatomic, readonly) Bar *bar;
@end

@implementation Foo
@dynamic title;
@dynamic bar;
@end
```

The reason we set the properties to be `dynamic` because we don't want the compiler to synthesize the properties.

```objc
// FooBuilder.h
@interface FooBuilder : Foo
@property (nonatomic, copy) NSString *title;
@property (nonatomic, strong) Bar *bar;
@end

// FooBuilder.m
@implementation FooBuilder 
{
  NSString *_title;
  Bar *_bar;
}

@synthesize title = _title;
@synthesize bar = _bar;

- (instancetype)init
{
  self = [super init];
  if (self) {
    _title = @"default";
    _bar = [[Bar alloc] init];
  }
  return self;
}
@end
```

We need to explicitly synthesize the properties otherwise the compiler will complain

> Auto property synthesis will not synthesize property 'title' because it is 'readwrite' but it will be synthesized 'readonly' via another property

Finally, when creating the instance, we need to create the builder class first and override any properties we require.

```objc
FooBuilder *fooBuilder = [[FooBuilder alloc] init];
fooBuilder.title = @"hello";
// override properties here ...
```

Next we can implement the `NSCopying` for `FooBuilder` and provide a `generate` method to make things a little simpler:

```objc
@interface FooBuilder : Foo <NSCopying>
- (Foo *)generate;
@end
```

```objc
- (Foo *)generate
{
  return [self copy];
}

- (id)copyWithZone:(NSZone *)zone 
{
  FooBuilder *other = [[FooBuilder alloc] init];
  other.title = self.title;
  return other;
}
```

Here's an usage example
```objc
- (void)useFoo
{
  FooBuilder *fooBuilder = [[FooBuilder alloc] init];
  fooBuilder.title = @"hello";

  Foo *foo = [fooBuilder generate];
  NSLog(@"%@", foo.title);
  NSLog(@"%@", foo.bar);
}
```

The important point to remember here is that any property that needs to be deep copied has to explicitly stated in the `copyWithZone`. For example, if we leave our implementation like:

```objc
- (id)copyWithZone:(nullable NSZone *)zone 
{
    return [[FooBuilder alloc] init];
}
```

We might get the default values for every `generate`

```objc
- (void)useFoo
{
  FooBuilder *fooBuilder = [[FooBuilder alloc] init];

  Foo *foo1 = [fooBuilder generate];
  NSLog(@"%@", foo1.title); // "default"
  NSLog(@"%@", foo1.bar); // <Bar: 0x6000019382b0>

  fooBuilder.title = @"hello";
  Foo *foo2 = [fooBuilder generate];
  NSLog(@"%@", foo2.title); // "default" <-- not "hello"
  NSLog(@"%@", foo2.bar); // <Bar: 0x6000019045c0>
}
```

Another thing to look out for is that `Foo` is now more like an abstract class with no implementation for properties, so we might crash at runtime if we try to initialize object directly like

```objc
Foo *foo = [[Foo alloc] init];
NSLog(@"%@", foo.title);
```

> Terminating app due to uncaught exception 'NSInvalidArgumentException', reason: '-[Foo title]: unrecognized selector sent to instance 0x600000a941f0'

First solution that comes to mind is marking the `init` as `NS_UNAVAILABLE` to throw error **'init' is unavailable**, but this would mean our factory subclass is also unavailable. This is another reason I do not the `NS_UNAVAILABLE` attribute which I believe was introduced without much thought but just for sake of making Objective-C interop with Swift.

The better solution would be to throw an exception

```objc
// Foo.m
- (instancetype)init;
{
  [NSException raise:NSInternalInconsistencyException format:@"Use [FooBuilder generate] instead"];
  return nil;
}
```

Although this would mean that the `init` is also not available to `FooBuilder`. One simple workaround could be provide an alternative designated initializer that is then only invoked from `FooBuilder`

```objc
// Foo.m
- (instancetype)initFromFactory
{
  return [super init];
}
```

```objc
// FooBuilder.m
- (instancetype)init
{
  self = [super initFromFactory];
  if (self) {
    _title = @"default";
    _bar = [[Bar alloc] init];
  }
  return self;
}
```

But this over engineering is only strictly optional, it saves nothing and makes code a little more complicated. The runtime should already raise an exception when trying to access any property from an instance directly created.