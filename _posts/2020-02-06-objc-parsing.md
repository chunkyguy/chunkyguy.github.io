---
layout: post
title:  "Objective-C and parsing the unknown"
date:   2020-02-06 23:28:54 +0200
categories: objc architecture
published: true
---

So there's this pattern I encounter every now and then. There is a service that returns some json data. The json data has some polymorphic behavior built-in. What I mean by that is that there is a root node that describes the behavior of the contained child node, which could be different for every child. To provide an example, lets say we are building a sort of a design system where the components are going to be provided by the server. The client then needs to parse the components and render them on screen. 

So there could be an `Image` component which looks like:

```json
{
    "type": "Image",
    "props": {
        "image": "https://via.placeholder.com/300/09f/fff.png",
        "width": 300,
        "height": 300
    }
}
```

Similarly there could be `Text` component which looks like:

```json
{
    "type": "Text",
    "props": {
        "text": "This is a text",
        "font-size": 12
    }
}
```

And many more components to come.

## Swift solution

With Swift, the solution I've seen implemented in most of the places is to use a giant `switch` somewhere most like with cyclomatic complexity disabled for the linter. Something like this:

```swift
typealias JSON = [String: Any]

func parse(json: JSON) -> Component? {
  guard
    let type = json["type"] as? String,
    let props = json["props"] as? JSON else {
    return nil
  }

  switch type {
  case "Image": return ImageComponent.parse(props: props)
  case "Text": return TextComponent.parse(props: props)
  default: return nil
  }
}
```

We can try making this less error prone by hiding raw `String` behind an enum like:

```swift
guard 
  let type = (json["type"] as? String).flatMap(ComponentType.init),
  let props = json["props"] as? JSON else {
  return nil
}
switch type {
  case .image: return ImageComponent.parse(props: props)
  case .text: return TextComponent.parse(props: props)
  default: return nil
}
```

But still there's no easy way to get rid of that switch statement.

## The problem

What's wrong with that switch statement anyways? 

Turns out everything. Behind the scene a switch statement is just an `if-else` like conditional branching with some syntactic sugar. This sort of `type` based conditional branching is against everything Object Oriented Programming advocates. 

One of the core ideas of OOP is to promote polymorphism over switch statements. In fact, most of the text books start out by an example where there's a giant switch statement somewhere and clean it up by writing an abstract implementation that is then implemented by every type in its own ways. 

You can find [many interesting articles](https://sourcemaking.com/refactoring/replace-conditional-with-polymorphism) that describe this in great details.

## Objective-C solution

With Objective-C this is an easy fix because we can parse the `type` and construct the class name at runtime. And then send the `parse` message to it.

```objc
- (id)parseComponentJSON:(NSDictionary *)componentDict
{
  NSString *type = [componentDict objectForKey:@"type"];
  NSString *componentName = [type stringByAppendingString:@"Component"];
  Class componentClass = NSClassFromString(componentName);
  if (componentClass == nil) {
    NSLog(@"%@ not found", componentName);
    return nil;
  }

  id component = [[componentClass alloc] init];
  if (![component respondsToSelector:@selector(parseJSON:)]) {
    return nil;
  }
  [component performSelector:@selector(parseJSON:)
                  withObject:[componentDict objectForKey:@"props"]];
  return component;
}
```

The components then can be implement as:

```objc
@interface ImageComponent : NSObject
@property (nonatomic, strong) NSURL* imageURL;
@property (nonatomic, assign) CGSize size;

- (void)parseJSON:(NSDictionary *)json;
@end

@implementation ImageComponent
- (void)parseJSON:(NSDictionary *)json;
{
  self.imageURL = [NSURL URLWithString:[json objectForKey:@"image"]];
  self.size = CGSizeMake([[json objectForKey:@"width"] doubleValue],
                         [[json objectForKey:@"height"] doubleValue]);
}

- (NSString *)description
{
  return [NSString stringWithFormat:@"%@, %@",
          self.imageURL,
          NSStringFromCGSize(self.size)];
}
@end
```

And another unrelated component

```objc
@interface TextComponent : NSObject
@property (nonatomic, strong) NSString* text;
@property (nonatomic, assign) double fontSize;

- (void)parseJSON:(NSDictionary *)json;
@end

@implementation TextComponent
- (void)parseJSON:(NSDictionary *)json
{
  self.text = [json objectForKey:@"text"];
  self.fontSize = [[json objectForKey:@"font-size"] doubleValue];
}

- (NSString *)description
{
  return [NSString stringWithFormat:@"%@, %.2f", self.text, self.fontSize];
}
@end
```

The beauty of this solution is that there is no giant `switch` statement anywhere. Whenever a new `Component` is introduced we only have to add another class that knows how to decode the `props` of that component.

If we want to make this more **protocol oriented**, we can have a proper `Component` type that provides an interface for how a `Component` should look like. 

```objc
@interface Component : NSObject
- (void)parseJSON:(NSDictionary *)json;
@end

@implementation Component
- (void)parseJSON:(NSDictionary *)json;
{
  [NSException raise:@"AbstractMethod" format:@"Not implemented"];
}
@end
```

And if we're that far, we can even make the signature is bit more clean to avoid having 2 steps; One for `[[class alloc] init]` and other for `[instance parseJSON: json]`

```objc
@interface Component : NSObject
+ (instancetype)componentWithJSON:(NSDictionary *)json;
@end

@implementation Component
+ (instancetype)componentWithJSON:(NSDictionary *)json;
{
  [NSException raise:@"AbstractMethod" format:@"Not implemented"];
  return nil;
}
@end
```

With that our parser simply looks like:

```objc
- (Component *)parseComponentJSON:(NSDictionary *)componentDict
{
  NSString *type = [componentDict objectForKey:@"type"];
  NSString *componentName = [type stringByAppendingString:@"Component"];
  Class componentClass = NSClassFromString(componentName);
  return [componentClass componentWithJSON:[componentDict objectForKey:@"props"]];
}
```

Now if you notice we're not checking of any `nil`. That is because sending messages to `nil` is safe and [very well documented](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/ObjectiveC/Chapters/ocObjectsClasses.html#//apple_ref/doc/uid/TP30001163-CH11-SW7) behavior. 

> If the method returns an object, then a message sent to nil returns 0 (nil).

This means that we don't have to check for intermediate `nil` if we our method is designed to return `nil` if parsing fails. So, we can even clean up further by removing all temporary variables in our parser

```objc
- (Component *)parseComponentJSON:(NSDictionary *)componentDict
{
  NSString *componentName = [[componentDict objectForKey:@"type"] stringByAppendingString:@"Component"];
  return [NSClassFromString(componentName) componentWithJSON:[componentDict objectForKey:@"props"]];
}
```

This is entire code:

```objc
@interface Component : NSObject
+ (instancetype)componentWithJSON:(NSDictionary *)json;
@end

@implementation Component
+ (instancetype)componentWithJSON:(NSDictionary *)json;
{
  [NSException raise:@"AbstractMethod" format:@"Not implemented"];
  return nil;
}
@end

@interface ImageComponent : Component
@property (nonatomic, strong) NSURL* imageURL;
@property (nonatomic, assign) CGSize size;

+ (instancetype)componentWithJSON:(NSDictionary *)json;
@end

@implementation ImageComponent
+ (instancetype)componentWithJSON:(NSDictionary *)json;
{
  ImageComponent *component = [[ImageComponent alloc] init];
  component.imageURL = [NSURL URLWithString:[json objectForKey:@"image"]];
  component.size = CGSizeMake([[json objectForKey:@"width"] doubleValue],
                         [[json objectForKey:@"height"] doubleValue]);
  return component;
}

- (NSString *)description
{
  return [NSString stringWithFormat:@"%@, %@",
          self.imageURL,
          NSStringFromCGSize(self.size)];
}
@end

@interface TextComponent : Component
@property (nonatomic, strong) NSString* text;
@property (nonatomic, assign) double fontSize;

+ (instancetype)componentWithJSON:(NSDictionary *)json;
@end

@implementation TextComponent
+ (instancetype)componentWithJSON:(NSDictionary *)json;
{
  TextComponent *component = [[TextComponent alloc] init];
  component.text = [json objectForKey:@"text"];
  component.fontSize = [[json objectForKey:@"font-size"] doubleValue];
  return component;
}

- (NSString *)description
{
  return [NSString stringWithFormat:@"%@, %.2f", self.text, self.fontSize];
}
@end

@interface ComponentParser : NSObject
- (NSArray *)parseComponentsWithData:(NSData *)data;
@end

@implementation ComponentParser

- (Component *)parseComponentJSON:(NSDictionary *)componentDict
{
  NSString *componentName = [[componentDict objectForKey:@"type"] stringByAppendingString:@"Component"];
  return [NSClassFromString(componentName) componentWithJSON:[componentDict objectForKey:@"props"]];
}

- (NSArray *)parseComponentsWithData:(NSData *)jsonData
{
  NSError *error = nil;
  NSDictionary *componentDict = [NSJSONSerialization JSONObjectWithData:jsonData
                                                            options:NSJSONReadingFragmentsAllowed
                                                              error:&error];
  NSAssert(error == nil, @"Parsing failure %@", [error localizedDescription]);

  NSArray *componentJSONs = [componentDict objectForKey:@"components"];
  NSMutableArray *components = [NSMutableArray arrayWithCapacity:[componentJSONs count]];
  for (NSDictionary *componentJSON in componentJSONs) {
    Component *component = [self parseComponentJSON:componentJSON];
    if (component != nil) {
      [components addObject:component];
    } else {
      NSLog(@"Unable to parse %@", componentJSON);
    }
  }
  return components;
}

@end

@interface Test : NSObject
- (void)run;
@end

@implementation Test

- (void)run
{
  NSError *error = nil;
  NSURL *path = [[NSBundle mainBundle] URLForResource:@"components" withExtension:@"json"];
  NSAssert(path != nil, @"No path");

  NSData *jsonData = [NSData dataWithContentsOfURL:path options:NSDataReadingMappedIfSafe error:&error];
  NSAssert(error == nil, @"File read failed %@ | %@", path, [error localizedDescription]);

  ComponentParser *parser = [[ComponentParser alloc] init];
  NSArray *components = [parser parseComponentsWithData:jsonData];

  NSLog(@"total components: %ld", [components count]);

  for (Component *component in components) {
    NSLog(@"%@", [component description]);
  }
}

@end
```

And a sample output with one component not implemented

```
Unable to parse {
    props =     {
        link = "https://whackylabs.com";
    };
    type = Web;
}
total components: 2
https://via.placeholder.com/300/09f/fff.png, {300, 300}
This is a text, 12.00
```

The more I think about it, I think very few languages provide a clean way to express OOP patterns. And in this case Objective-C seems to be cleaner implementation than Swift. 

If you know of a better way to solve this problem in Swift like languages, please let me know on [twitter](https://twitter.com/chunkyguy)

## References

1. [Object-Oriented Programming in Objective-C -- youtube.com](https://www.youtube.com/watch?v=_BbGxpiYFDg)
1. [Cocoa Design Patterns -- developer.apple.com](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/CocoaFundamentals/CocoaDesignPatterns/CocoaDesignPatterns.html#//apple_ref/doc/uid/TP40002974-CH6-SW6)