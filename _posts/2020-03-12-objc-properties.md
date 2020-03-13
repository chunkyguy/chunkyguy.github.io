---
layout: post
title:  "Objective-C Properties Problems"
date:   2020-03-12 23:00:00 +0200
categories: objc
published: true
---

[Objective-C 2.0 introduced](https://web.archive.org/web/20100724195423/http://developer.apple.com:80/leopard/overview/objectivec2.html) properties as syntactic sugar over getter and setter a very long time back. Properties are very convenient for most of the things but there are quite a few issues that I feel always with them. Today I would like to document them.

## Dot syntax

In C there idea of `->` and `.` to access `struct` properties and it is simple to understand. For a given `struct` as:

```c
typedef struct _Person {
  char name[1024];
  int age;
} Person;
```

We have `.` for values

```c
Person setAge(Person p, int age)
{
  p.age = age;
  return p;
}
```

And we have `->` for references

```c
void setName(Person *p, const char *name)
{
  strcpy(p->name, name);
}
```

With Objective-C, all `NSObject` subclasses can only be used as reference types, like `NSString *`, so using the `.` on reference types breaks my brain every single time. 

```objc
@interface Person : NSObject
@property (nonatomic, copy) NSString *name;
@end

void setName(Person *p, NSString *name)
{
  p.name = name;
}
```

I believe the intention was good, but they could be picked some clearly non-ambiguous operator, like say `~>`. I know it would look unnatural to C programmers, that was one of the core design principle of Objective-C, to have a syntax that is clearly distinguishable from C. That is why we have things like `[` `]`, `@interface` and `-(void)foo:(Bar *)bar` because they don't look anything like C.

I've a conspiracy theory, that the `.` was used for property because Apple wanted the developers to adapt Swift syntax gradually, where `.` is the only way to access properties.

## Error prone

Let's say we want to implement a property chaining where the only storage is `NSDate` and everything is just a computed getters and setters on top of it

```objc
@interface DateWrapper : NSObject
@property (nonatomic, strong) NSDate *date;
@property (nonatomic, strong) NSDateComponents *dateComps;
@property (nonatomic, assign) NSInteger hour;
@end
```
```objc
@implementation DateWrapper

- (instancetype)init
{
  self = [super init];
  if (self) {
    _date = [NSDate dateWithTimeIntervalSince1970:0];
  }
  return self;
}

- (NSInteger)hour
{
  return self.dateComps.hour;
}

- (void)setHour:(NSInteger)hour
{
  self.dateComps.hour = hour;
}

- (NSDateComponents *)dateComps
{
  return [[NSCalendar currentCalendar] components:NSCalendarUnitHour 
                                         fromDate:self.date];
}

- (void)setDateComps:(NSDateComponents *)dateComps
{
  self.date = [[NSCalendar currentCalendar] dateFromComponents:dateComps];
}

@end
```

One could accidentally use the interface as:

```objc
- (void)run
{
  DateWrapper *dw = [[DateWrapper alloc] init];
  NSLog(@"BEF: %ld", dw.hour); // 1
  dw.hour += 1;
  NSLog(@"AFT: %ld", dw.hour); // 1 ❌
}
```

These bugs could happen again because of the C like dot syntax. Because we know this works in C

```c
CGRect frame = CGRectZero;
frame.size.width += 10; // ✅
```

We might apply the same principles to Objective-C properties

```objc
self.dateComps.hour += hour; // ❌
```

If you haven't figured it out yet. The bug is because `self.dateComps` call the getter and we update the `hour` on a temporary object. The right way would be

```objc
NSDateComponents *tmp = self.dateComps;
tmp.hour += hour;
self.dateComps = tmp;
```

The bug becomes more visible by switching to getters and setters because every operation is clear in terms of where are we reading the data and where are we writing it.

```objc
@interface DateWrapper : NSObject
{
  NSDate *_date;
}
@end
```
```objc
@implementation DateWrapper

- (instancetype)init
{
  self = [super init];
  if (self) {
    _date = [NSDate dateWithTimeIntervalSince1970:0];
  }
  return self;
}

- (NSInteger)hour
{
  return [[self dateComps] hour]; // read
}

- (void)setHour:(NSInteger)hour
{
  NSDateComponents *comps = [self dateComps]; // read
  [comps setHour:hour]; // write
  [self setDateComps:comps]; // write
}

- (NSDateComponents *)dateComps
{
  return [[NSCalendar currentCalendar] components:NSCalendarUnitHour 
                                         fromDate:_date]; // read
}

- (void)setDateComps:(NSDateComponents *)dateComps
{
  _date = [[NSCalendar currentCalendar] dateFromComponents:dateComps]; // write
}

@end
```

```objc
- (void)run
{
  DateWrapper *dw = [[DateWrapper alloc] init];
  NSLog(@"BEF: %ld", [dw hour]); // 1
  [dw setHour:[dw hour] + 1];
  NSLog(@"AFT: %ld", [dw hour]); // 2
}
```

## Chaining Setters

One limitation of property is that the setter has to return a `void` which could be problematic if you want to design your interface to be chainable. Take this usage for example

```objc
- (Person *)updatePerson:(Person *)person
{
  person.firstName = @"Barney";
  person.lastName = @"Gumble";
  person.age = 40;
  return person;
}
```

If we drop the whole property requirement, we could return `instancetype` from every setter which would make the setters as chainable.

```objc
- (Person *)updatePerson:(Person *)person
{
    return [[[person setFirstName:@"Barney" ]
                      setLastName:@"Gumble" ]
                           setAge:40        ];
}
```

## Ambiguous Attributes

Properties are ambiguous when we're writing custom getter and setters. If we've a C function that encodes and decodes `string` to `int`, and we wish to provide a Objective-C wrapper.

```c
int my_encrypt(const char *str);
const char *my_decrypt(int hash);
```

```objc
@interface StringWrapper : NSObject
{
  int _hash;
}
@end

@implementation StringWrapper
- (void)setString:(NSString *)string
{
  _hash = my_encrypt([string cStringUsingEncoding:NSUTF8StringEncoding]);
}

- (NSString *)string
{
  return [NSString stringWithCString:my_decrypt(_hash) 
                            encoding:NSUTF8StringEncoding];
}
@end
```

Next if we wish to expose `string` as a property, what should the attributes look like? We might use `copy` or `retain`, but that would be a lie, since we know we're not copying or retaining anything.

```objc
@property (nonatomic, copy) NSString *string;
```

We might simply leave it unspecified, but again unspecified does mean the default attributes `atomic`, `readwrite`, which might again cause confusion.

```objc
@property NSString *string;
```

With plain getter and setter, the memory storage or threading intention is never leaked outside.

```objc
@interface StringWrapper : NSObject
- (NSString *)string;
- (void)setString:(NSString *)string;
@end
```

## Good parts: MRR

The only good use of a property that I can imagine is when we're working on a **Manual Retain Release** memory model, the `retain` and `release` calls could be error prone. 

Take a look at the following example with a bug:

```objc
@interface Foo : NSObject
{
  Bar *_bar;
}
@end
```
```objc
@implementation Foo
- (instancetype)init
{
  self = [super init];
  if (self) {
    _bar = [[Bar alloc] init];
  }
  return self;
}

- (void)dealloc
{
  [_bar release];
  [super dealloc];
}

- (Bar *)bar
{
  return _bar;
}

- (void)setBar:(Bar *)bar
{
  [_bar release]; // ❌ might also release bar
  _bar = [bar retain];
}

@end
```

This might introduce a bug if the `bar` is same as `_bar` in `setBar:`. The first `release` would deallocated the object immediately and the `retain` in the next line would have no effect. The right way to implement a setter would be to call `retain` before `release`.

```objc
- (void)setBar:(Bar *)bar
{
  [bar retain];
  [_bar release];
  _bar = bar;
}
```

This is one of many consideration one has to look out for when using MRR model. So with MRR it's usually better to just use property for `retain` or `copy` attributes and let the compiler generate the correct code

```objc
@interface Foo : NSObject
@property (nonatomic, retain) Bar *bar;
@end

@implementation Foo
- (instancetype)init
{
  self = [super init];
  if (self) {
    _bar = [[Bar alloc] init];
  }
  return self;
}

- (void)dealloc
{
  [_bar release];
  [super dealloc];
}
@end
```

## Parting Thoughts

I do like the convenience of properties as a short hand for plain getter and setter. That means no property chaining and the intention is very clear. I also like using properties when working MRR memory model. Or when implementing a `readonly` property. But there are these few things I don't like about properties that sometimes I deliberately avoid using them for sake of accidental bugs.

I would file Objective-C properties under the **Apple Fuckups** section. It could've been really brilliant if they had consulted the Objective-C language designers and not C/C++, and hopefully not Swift.

I'm probably not the first one to dislike Objective-C properties, and I know I'm not the last. I'm actually a bit glad that Apple has now shifted the gears to Swift, so finally I can write Objective-C that way I always wanted to.