---
layout: post
title:  "State of Meta Programming in 2020"
date:   2020-02-10 23:28:54 +0200
categories: programming generic
published: true
---

We all know what meta programming is. We all have tasted that medicine at some point in our lives. The basic idea is that first there is a level of coding we all are familiar with. Usually it is full of fun and excitement. Other times it becomes a bit repetitive, usually when we find ourselves copy pasting some code over and over again and we wish there were an another higher level of code that take over this job. 

To help with meta programs we have designed a plethora of tools, both standalone and integrated deep within our every day text editors. Some programming languages even went a step further and integrated support for meta programming deep inside the language itself, others integrated meta programming support in the runtime.

Today I want to take a look at all the options available in programming languages I'm most familiar with. Hoping this would cover most of the programming languages out there.

## Problem

Let's say we had a simple clean `sum` function which was originally designed to take an array of integers and always returned a integer. Next, we want to have a almost the same function but for many other types, like floating points, string or even color which would sum the red, blue and green components such that an array of red, green and blue would return same value as of the color white.

## External tools

Before we even go into any language specific solution, I think it is wise to mention that there [are](https://stencil.fuller.li/en/latest/) [several](https://swagger.io/tools/swagger-codegen/) [templating](https://github.com/krzysztofzablocki/Sourcery) [tools](http://mustache.github.io) [that](https://github.com/apple/swift/blob/master/utils/gyb.py) would love help with this job by simply generating source code from a template.

Here's an sample of how a template might look like:
```
%{
supported_types = ['Color', 'String', 'Double', 'Float',
                 'Int', 'Int8', 'Int16', 'Int32', 'Int64',
                 'UInt', 'UInt8', 'UInt16', 'UInt32', 'UInt64']
}%

% for type in supported_types:
func sum(elements: [${type}]) -> Int {
  var s = 0
  for element in elements {
    s += element
  }
  return s
}
% end
```

### Pros

- These tools get the job done with minimal overhead.
- Have a pretty good community support.

### Cons

- Hard to validate the output
- There is a learning curve involved to learn the syntax expected by the templating engine.
- The tools have to be constantly upgraded as the target language evolves.
- Every change has to be done externally.

## C

Here's a very simple implementation of the `sum` function in C.

```c
int sum(int *elements, int count) {
  int s = 0;
  for (int i = 0; i < count; ++i) {
    s += elements[i];
  }
  return s;
}

printf("%d\n", sum((int []){1,2,3,4}, 4)); // 10
```

C is pretty straightforward as far as meta programming is concerned. It doesn't support any meta programming in the traditional sense. There is a bit of preprocessing one can do to achieve a bit of template like solution.

```c
#define gen_sum(T) int sum_##T(T *elements, int count) { \
  int s = 0; \
  for (int i = 0; i < count; ++i) { \
    s += elements[i]; \
  } \
  return s; \
}

gen_sum(int)
gen_sum(float)

printf("%d\n", sum_int((int []){1,2,3,4}, 4)); // 10
printf("%d\n", sum_float((float []){1.0,2.0,3.0,4.0}, 4)); // 10
```

Things start getting a bit uglier for types that can not be automatically converted to `int`, like say a string.

```c
typedef char * str;
gen_sum(str)
``` 

If we take a look inside, this is what the preprocessor generates:

```c
int sum_str(char ** elements, int count) { 
  int s = 0; 
  for (int i = 0; i < count; ++i) { 
    s += elements[i]; // int + char*
  } 
  return s; 
}
```

The solution is to delegate the `s += element[i]` out of here. We want to make this behave as 

```c
s += atoi((char *)element[i])
```

The idea is to use a function pointer to serve as callback that takes in `T` and returns `int`. For basic fundamental types we can pass `NULL` which would then fallback to whatever we had.

```c
#define gen_sum(T) int sum_##T(T *elements, int count, int(*f)(T)) { \
  int s = 0; \
  for (int i = 0; i < count; ++i) { \
    if (f == NULL) { s += elements[i]; } \
    else { s += f(elements[i]); } \
  } \
  return s; \
}

gen_sum(int)
gen_sum(float)

typedef char * str;
gen_sum(str)

printf("%d\n", sum_int((int []){1,2,3,4}, 4, NULL));
printf("%d\n", sum_float((float []){1.0,2.0,3.0,4.0}, 4, NULL));
printf("%d\n", sum_str((str []){"1", "2", "3", "4"}, 4, atoi));
```

Sweet! How about custom types?

```c
typedef struct color_t {
  int r, g, b;
} color;

int color_value(color c) {
  return (c.r << 16) + (c.g << 8) + c.b;
}

gen_sum(color)

printf("%d\n", sum_color((color []){ {1, 0, 0}, {0, 1, 0}, {0, 0, 1} }, 3, color_value));
```

The compiler will complain about `s += elements[i];` even though that line of code should never even execute at runtime as `f` is not `NULL`. The problem is because the compiler can not figure out how to generate code for `int + color`. The solution is to drop the fallback case and always pass in a function pointer.

```
#define gen_sum(T) int sum_##T(T *elements, int count, int(*f)(T)) { \
  int s = 0; \
  for (int i = 0; i < count; ++i) { \
    s += f(elements[i]); \
  } \
  return s; \
}
```

Now, for fundamental types we can create another template that just generates a direct mapping. Something like:

```c
#define gen_convert(T) int to_##T(T x) { return x; }

gen_convert(int); // generates: int to_int(int x) { return x; };
```

With that we can call all sort of types:

```c
printf("%d\n", sum_int((int []){1, 2, 3, 4}, 4, to_int));
printf("%d\n", sum_float((float []){1.0, 2.0, 3.0, 4.0}, 4, to_float));
printf("%d\n", sum_str((str []){"1", "2", "3", "4"}, 4, atoi));
printf("%d\n", sum_color((color []){ {1, 0, 0}, {0, 1, 0}, {0, 0, 1} }, 3, color_value));
```

This is the entire code:

```c
#define gen_convert(T) int to_##T(T x) { return x; }

#define gen_sum(T) int sum_##T(T *elements, int count, int(*f)(T)) { \
  int s = 0; \
  for (int i = 0; i < count; ++i) { \
    s += f(elements[i]); \
  } \
  return s; \
}

gen_sum(int)
gen_convert(int)

gen_sum(float)
gen_convert(float)

typedef const char * str;
gen_sum(str)

typedef struct color_t {
  int r, g, b;
} color;

gen_sum(color)

int color_value(color c) {
  return (c.r << 16) + (c.g << 8) + c.b;
}

void test_sum() {
  printf("%d\n", sum_int((int []){1, 2, 3, 4}, 4, to_int));
  printf("%d\n", sum_float((float []){1.0, 2.0, 3.0, 4.0}, 4, to_float));
  printf("%d\n", sum_str((str []){"1", "2", "3", "4"}, 4, atoi));
  printf("%d\n", sum_color((color []){ {1, 0, 0}, {0, 1, 0}, {0, 0, 1} }, 3, color_value));
}
```

### Pros

- No need for an external build system.

### Cons
- Hard to read code

## C++

C++ comes with all sort of meta programming goodies built in. People have been known to go crazy over how awesome the meta programming actually is. I've seen people making amazing thing with just the meta programming, for example a [Tetris game at compile time](https://blog.mattbierner.com/stupid-template-tricks-super-template-tetris/)!

So coming from C, building the solution in C++ should be a lot easier.

```cpp
template <typename T>
int sum(const std::vector<T> &v) {
  int s = 0;
  for (const T element : v) {
    s += element;
  }
  return s;
}

std::vector<int> v1 = {1,2,3,4};
std::cout << "int: " << sum(v1) << std::endl; // 10

std::vector<float> v2 = {1.0,2.0,3.0,4.0};
std::cout << "float: " << sum(v2) << std::endl; // 10
```

The C++ solution is surprisingly very flexible. It might even inconveniently work for types we didn't expect it to.

```cpp
std::vector<char> v3 = {'a', 'b', 'c'};
std::cout << "char: " << sum(v3) << std::endl; // 294

std::vector<bool> v4 = {true, false};
std::cout << "bool: " << sum(v4) << std::endl; // 1
```

For thing where this might fail, like `string`, the C++ solution is to provide an overload

```cpp
int sum(const std::vector<std::string> &v) {
  int s = 0;
  for (const std::string & element : v) {
    s += std::stoi(element);
  }
  return s;
}

std::vector<std::string> v5 = {"1", "2", "3", "4"};
std::cout << "string: " << sum(v5) << std::endl;
```

This exact solution also applies to any of our custom types

```cpp
struct color {
  int r, g, b;

  int value() const {
    return (r << 16) + (g << 8) + b;
  }
};

int sum(const std::vector<color> &v) {
  int s = 0;
  for (const color & element : v) {
    s += element.value();
  }
  return s;
}
```

### Pros

- Flexibility
- Amazing compiler support

### Cons

- Might accidentally work for non-supported types

## Swift

I feel Swift is very heavily influenced by C++. So it shouldn't come as any surprise that Swift supports very good meta programming with generics. If our simple `sum` looks something lie:

```swift
func sum(elements: [Int]) -> Int {
  var s = 0;
  for element in elements {
    s += element
  }
  return s
}

print("\(sum(elements: [1, 2, 3, 4]))") // 10
```

The generic version is not very different:

```swift
func sum<T>(elements: [T]) -> Int {
  var s = 0;
  for element in elements {
    s += element
  }
  return s
}
```

But this is where things start diverging away. Swift is very strict with it's type checking. So strict that a few times it might even come out as annoying for simple use cases, at least simple inside our head. If we try to call this with `Int` we get an error:

```swift
print("\(sum(elements: [1, 2, 3, 4]))")
```
> Cannot convert value of type 'T' to expected argument type 'Int'

The problem above is with `s += element`. The compiler can not perform `+` between `Int` and `T` because unlike C++, Swift assumes nothing about `T`. So we need to provide exact information to the compiler of what `T` is capable of. 

Simplest solution here could be to get rid of `Int` and provide a `T: AdditiveArithmetic` conformance. Since both `Int` and `Double` already conform to `AdditiveArithmetic` we won't have to do anything more. 

Another problem might be that `s = 0` is not available for `T`, but we can workaround by providing that as an additional param.

```swift
func sum<T>(initial: T, elements: [T]) -> T where T: AdditiveArithmetic {
  var s = initial;
  for element in elements {
    s += element
  }
  return s
}

print("\(sum(initial: 0, elements: [1, 2, 3, 4]))") // 10
print("\(sum(initial: 0, elements: [1.0, 2.0, 3.0, 4.0]))") // 10
```

This is looking like close to the standard `reduce` function, and now we know why.

When we need to expand our `sum` function to non-trivial types, like say the `String`, we face another error:

```swift
print("\(sum(initial: "0", elements: ["1", "2", "3", "4"]))") 
```
> Error: Argument type 'String' does not conform to expected type 'AdditiveArithmetic'

We could continue and explicitly provide a conformance for `String` for `AdditiveArithmetic`. This could be a way to go if we knew that the `sum` function only needs to handle a limited types. Or if modifying `sum` was outside our control. Or we can simply rollback to our previous implementation and make that work.

Remember, our original solution was as simple as `func sum<T>(elements: [T]) -> Int`. How can we make this work for `String`? In the earlier solution the problem was with `+` not working for `Int` and `T`. How about providing a protocol that converts any type to `Int`?

```swift
protocol IntegerConvertible {
  var intValue: Int? { get }
}
```

Then our `sum` can be simplified back:

```swift
func sum<T>(elements: [T]) -> Int where T: IntegerConvertible {
  var s = 0;
  for element in elements {
    if let v = element.intValue {
      s += v
    }
  }
  return s
}
```

We need every type to provide their own way to implement `intValue`. Which is trivial for types we are covering so far

```swift
extension Int: IntegerConvertible {
  var intValue: Int? { return self }
}

extension Double: IntegerConvertible {
  var intValue: Int? { return Int(self) }
}

extension String: IntegerConvertible {
  var intValue: Int? { return Int(self) }
}
```

The beauty of this easy and flexible solution is that it's really easy to extend to any new type.

```swift
extension CGColor: IntegerConvertible {
  var intValue: Int? {
    guard let comps = self.components else {
      return nil
    }
    // probably unsafe access!!
    let r = comps[0]
    let g = comps[1]
    let b = comps[2]
    return (Int(r * 255.0) << 16) + (Int(g * 255.0) << 8) + (Int(b * 255.0));
  }
}
```

### Pros

- Type checking from compiler

### Cons

- Very verbose

## Objective-C

With Objective-C meta programming is actually more fun than with Swift or C++. Personally I feel like it's a very different dimension of meta programming not available in other programming languages I know of.

Remember the basic idea behind meta programming is to first write a simple solution and later extend it to work any type. With that in mind, this is our basic `Adder` class for `NSNumber` types:

```objc
@interface Adder : NSObject
@property (nonatomic, copy) NSArray *elements;
@property (nonatomic, readonly) NSInteger sum;
+ (instancetype)adderWithElements:(NSArray *)elements;
@end

@implementation Adder

+ (instancetype)adderWithElements:(NSArray *)elements
{
  Adder *addr = [[Adder alloc] init];
  addr.elements = elements;
  return addr;
}

- (NSInteger)sum
{
  NSInteger s = 0;
  for (NSNumber *element in self.elements) {
    s += [element integerValue];
  }
  return s;
}
@end

NSLog(@"ints: %ld", [[Adder adderWithElements:@[@1, @2, @3, @4]] sum]);
```

Since Objective-C is not so strict on compile time, we need to make sure that every element in `NSArray` can has implemented the `integerValue` method. One way to achieve this is

```objc
- (NSInteger)sum
{
  NSInteger s = 0;
  for (id element in self.elements) {
    if ([element respondsToSelector:@selector(integerValue)]) {
      s += [element integerValue];
    }
  }
  return s;
}
```

And that's it! We are done! This same thing works for every type you can imagine!

```objc
NSLog(@"ints: %ld", [[Adder adderWithElements:@[@1, @2, @3, @4]] sum]);
NSLog(@"floats: %ld", [[Adder adderWithElements:@[@1.0, @2.0, @3.0, @4.0]] sum]);
NSLog(@"strings: %ld", [[Adder adderWithElements:@[@"1", @"2", @"3", @"4"]] sum]);
```

But what for custom types that do not provide `integerValue` out of the box? The good news is that it won't crash at runtime, but the bad news is that it won't return anything meaningful either.

```objc
NSLog(@"colors: %ld",
       [[Adder adderWithElements:@[
        [UIColor redColor],
        [UIColor greenColor],
        [UIColor blueColor]]] sum]); // colors: 0
```

But it's relatively easy to extend support for any custom type.

```objc
@interface UIColor (AdderSupport)
@end

@implementation UIColor (AdderSupport)
- (NSInteger)integerValue
{
  CGFloat r,g,b;
  [self getRed:&r green:&g blue:&b alpha:nil];
  return ((NSInteger)(r * 255.0) << 16) + ((NSInteger)(g * 255.0) << 8) + ((NSInteger)(b * 255.0));
}
@end
```

### Pros

- Very easy to work with.

### Cons

- Easy to run into unexpected situations at runtime

## Final words

Every meta programming exercise starts out with a pretty simple problem, at least in our head. *"I've this thing that works only for one specific case. How can I make this thing work for this other type as well?"*. Several hours later we find ourselves in this labyrinth of generic types where there are things with the sole purpose of describing the constraint environment. Then there are few things that are not required to actually solve the problem but might serve as a guideline on how things are supposed to work, particularly useful if the solution ever needs an expansion, say a note for the future maintainer written in pure code and conveniently type checked by the compiler. And then finally there are things that actually solve the original problem.

This is a more true with Swift than with C++ where so many requirements we have to explicitly specified otherwise the compiler will get mad. On the other hand with C++, the compiler asks for nothing beforehand, but there are a few things one is somehow expected to know, otherwise the compiler will get mad.

I think the best strategy when going down the meta programming rabbit hole is to follow these steps:

- The first draft should be very short, clean, concise and to the point. No wondering about cases that do not yet exists.

- When expanding for another type avoid overthinking. Implement the algorithm that provides a solution for just the subset required. So no over abstract protocols, no complex types. Focus is the key here.