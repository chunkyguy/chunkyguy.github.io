---
layout: post
title:  "Enums are anti-pattern"
date:   2020-02-16 23:28:54 +0200
categories: architecture objc swift
published: false
---

Enums are a convoluted construct. As far as Object Oriented programming is concerned, they are **anti pattern**. They have this sort of magnetic nature to converge all the control flow to a single place, making a *God class* like behavior which only grows in complexity over time. In languages like C this is really helpful as there is no better mechanism to envelop the type system. So we need to invent an independent meta type system with enums. 

Search for any [parser implemented in C](https://github.com/google/gumbo-parser/blob/master/src/parser.c#L579) and at the core you would find a giant enough switch case. No wonder language designers and compiler writers love enums, this is how they solve problems.

## Problem

To illustrate, lets write a parser in Swift. Say we need a system to handle deeplinks where commands are encoded in a `URL` that needs to be parsed. This is the format we've designed:

```
scheme://hostname/command?argument=key,...
```

Here are few examples of how the URLs might look like:

```
https://example.com/home
https://example.com/checkout?transaction_id=234234
```

Step 1, we implement the `Command`

```swift
protocol Command {
  func execute()
}

struct HomeCommand: Command {
  func execute() {
    print("Home")
  }
}

struct CheckoutCommand: Command {
  let transactionId: String

  func execute() {
    print("Transaction \(transactionId)")
  }
}
```

Step 2, we implement the parser

```swift
struct DeeplinkParser {

  private enum CommandName: String {
    case home
    case checkout
  }

  private struct CommandArgument {
    let name: String
    let value: String
  }

  func parse(url: URL) -> Command? {
    guard
      let urlComps = URLComponents(url: url, resolvingAgainstBaseURL: false),
      let commandName = urlComps.path.components(separatedBy: "/")
        .last.flatMap(CommandName.init) else { return nil }

    let arguments = (urlComps.queryItems ?? []).compactMap { query in
      query.value.map { CommandArgument(name: query.name, value: $0) }
    }

    switch commandName {
    case .home:
      return HomeCommand()

    case .checkout:
      guard let transactionId = arguments.first(where: {
        $0.name == "transaction_id" })?.value else {
          return nil
      }
      return CheckoutCommand(transactionId: transactionId)

    // Many more to come ...
    }
  }
}
```

And finally put it to test:
```swift
func testParser() {
  let parser = DeeplinkParser()
  URL(string: "https://example.com/home")
    .flatMap(parser.parse)?.execute()
  URL(string: "https://example.com/checkout?transaction_id=234234")
    .flatMap(parser.parse)?.execute()
}
```

Works fine! But this feels like very close to how one would do it in C as well. As the list of commands grows so does the code complexity in the `DeeplinkParser`. Code complexity can be simply measured by the number of branches available in a function, [this article](https://blog.codacy.com/an-in-depth-explanation-of-code-complexity/) describes it very nicely. In case of a simple switch statement it's just the number of cases handled including default.

## Solution

If we were to plot the code control distribution of our above code as a heat map, we would see a lot of activity in the `DeeplinkParser` and very less in the `Command`. The kind of heat map we should be aiming for is the one where the control is spread out evenly as possible. This is one of the promises made by Object Oriented Programming, but failed at implementation by languages like C++ and Java.

Let's see how we could implement this parser in Objective-C and get rid of all the control flow congestion out of a single *God class*.

Step 1: Define a base class `PLCommand` that owns the logic for every command we care about, current and future. And them every subclass only needs to take care of their own implementations.

```objc
@interface PLCommand : NSObject

@property (nonatomic, copy) NSString * name;
@property (nonatomic, copy) NSArray * arguments;

- (void)execute;
- (NSString *)argumentForKey:(NSString *)key;
@end

@implementation PLCommand

- (void)execute
{
  [NSException raise:@"Not Implemented"
              format:@"Should be implemented by subclasses"];
}

- (NSString *)argumentForKey:(NSString *)key;
{
  for (NSDictionary *arg in self.arguments) {
    if ([[arg objectForKey:@"name"] isEqualToString:key]) {
      return [arg objectForKey:@"value"];
    }
  }
  return nil;
}

@end
```

```objc 
@interface PLHomeCommand : PLCommand
@end

@implementation PLHomeCommand
- (void)execute 
{
  NSLog(@"Home");
}
@end
```

```objc 
@interface PLCheckoutCommand : PLCommand
@end

@implementation PLCheckoutCommand

- (void)execute 
{
  NSLog(@"Checkout: TransactionId: %@", [self argumentForKey:@"transaction_id"]);
}

@end
```

Step 2: Next implement the `PLDeeplinkParser`

```objc
@interface PLDeeplinkParser : NSObject
- (PLCommand *)parseURL:(NSURL *)url;
@end

@implementation PLDeeplinkParser

- (PLCommand *)parseURL:(NSURL *)url
{
  NSURLComponents *urlComps = [NSURLComponents componentsWithURL:url
                                         resolvingAgainstBaseURL:NO];
  NSString *commandName = [NSString stringWithFormat:@"PL%@Command",
                           [[[urlComps path] lastPathComponent] capitalizedString]];
  Class CommandClass = NSClassFromString(commandName);
  NSAssert(CommandClass != nil, @"Unable to find command: %@", commandName, nil);
  PLCommand *command = [[CommandClass alloc] init];
  command.name = commandName;
  NSMutableArray *args = [NSMutableArray array];
  for (NSURLQueryItem *q in [urlComps queryItems]) {
    [args addObject:[NSDictionary dictionaryWithObjectsAndKeys:
                     q.name, @"name",
                     q.value, @"value", nil]];
  }
  command.arguments = args;
  return command;
}

@end
```

And finally time to test the code:

```objc
- (void)testParser
{
  PLDeeplinkParser *parser = [PLDeeplinkParser new];
  [[parser parseURL:
    [NSURL URLWithString:@"https://example.com/home"]] execute];
  [[parser parseURL:
    [NSURL URLWithString:@"https://example.com/checkout?transaction_id=234234"]] execute];
}
```

So where did we actually push the switch case? The answer is: **Objective-C Runtime**, with that call to `NSClassFromString()`. That's the place where we map parsed `NSString` to a `PLCommand`. It is probably not as fast a direct alloc/init, but the runtime is designed to be very efficient for this lookup. There has to be a good justification for any superfluous code complexity. Like, if we are working on a game or an app that needs 60 FPS rendering and this lookup is the bottleneck. Then sure we can roll back to the enum based approach, like the compiler writers do. But for everyone else there are always alternate solutions which might be better.

## Read more

1. [Enums as configuration: the anti-pattern **• jessesquires.com**](https://www.jessesquires.com/blog/enums-as-configs/)
2. [The Evils of the switch Statement **• lostechies.com**](https://lostechies.com/johnteague/2013/03/03/polymorphism-part-2-refactoring-to-polymorphic-behavior/)
3. [Could enums be considered an anti-pattern? **• users.rust-lang.org**](https://users.rust-lang.org/t/could-enums-be-considered-an-anti-pattern/10068)
