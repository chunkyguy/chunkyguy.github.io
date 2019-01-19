---
layout: post
title:  "Curious case of Generics Specialization with Swift"
date:   2015-01-20 11:54:54 +0530
categories: swift
---

Today, I want to talk about a curious case I discovered while playing
with generic programming with Swift.

To illustrate, let’s start by writing a simple function.

``` swift
func Min(x:Int, y:Int) -> Int {
    println("Using Min<Int>")
    return (x < y) ? x : y
}

let a = Min(-1, 1) // Using Min<Int>
```

And now let’s make it generic.

``` swift
func Min<T:Comparable>(x:T, y:T) -> T {
    println("Using Min<T>")
    return (x < y) ? x : y
}

let a = Min(-1, 1) // Using Min<T>
```

What if we’ve both the implementations? The compiler automatically picks
the specialized version.

``` swift
func Min<T:Comparable>(x:T, y:T) -> T {
    println("Using Min<T>")
    return (x < y) ? x : y
}

func Min(x:Int, y:Int) -> Int {
    println("Using Min<Int>")
    return (x < y) ? x : y
}

let a = Min(-1, 1) // Using Min<Int>
```

The swift standard library already provides a min and max functions.

``` swift
func min<T : Comparable>(x: T, y: T) -> T
```

Suppose we use that as the generic version and override our specialized
one for Int? The compiler still picks the specialized one.

``` swift
func min(x:Int, y:Int) -> Int {
    println("Using Min<Int>")
    return (x < y) ? x : y
}

let a = min(-1, 1) // Using min<Int>
```

This is really convenient, isn’t it? Let’s expand our example to
something you might face in real life. Let’s work with a Vector2 type.

``` swift
struct Vector2 {
    var x : Float = 0.0
    var y : Float = 0.0

    init(x:Float = 0,y:Float = 0) {
        self.x = x
        self.y = y
    }
}
```

How about making use of standard min and max functions with Vector2?

``` swift
let lowerBound = Vector2(x: -1, y: -1)
let upperBound = Vector2(x: 1, y: 1)

let a = min(lowerBound, upperBound)
```

This is going to throw an error, as the standard min and max functions
need the type to conform to the comparable protocol. So let’s do that
first.

``` swift
func ==(lhs: Vector2, rhs: Vector2) -> Bool {
   return (lhs.x == rhs.x) && (lhs.y == rhs.y)
}

func < (lhs: Vector2, rhs: Vector2) -> Bool {
    return (lhs.x < rhs.x) && (lhs.y < rhs.y)
}

extension Vector2 : Comparable {}

let lowerBound = Vector2(x: -1, y: -1)
let upperBound = Vector2(x: 1, y: 1)

let a = min(lowerBound, upperBound)
```

This works as expected. This is a great feature, in my opinion, one of
the best things to switch from Objective-C to Swift. Moving forward,
let’s write a generic clamp function.

``` swift

func clamp<T:Comparable>(value:T, lowerBound:T, upperBound:T) -> T {
    return min(max(lowerBound, value), upperBound)
}

let b = clamp(10, -1, 1) // prints 1
```

This is awesome! Let’s try our clamp function with Vector2.

``` swift
let lowerBound = Vector2(x: -100, y: -100)
let upperBound = Vector2(x: 100, y: 100)

let test1 = clamp(Vector2(x: 200, y: 200), lowerBound, upperBound) // {x:100, y:100}
let test2 = clamp(Vector2(x: -200, y: -200), lowerBound, upperBound) // {x:-100, y:-100}
let test3 = clamp(Vector2(x: -10, y: -134), lowerBound, upperBound) // {x:-10, y:-134}
let test4 = clamp(Vector2(x: 10, y: 134), lowerBound, upperBound) // {x:10, y:134}
```

At first this might look bad, because the test3 and test4 are not
correct. But, this is not the compiler’s fault. The standard min and max
use the overloaded comparison operators and they are not correct. We can
test this with

``` swift
let test5 = min(Vector2(x: -10, y: -134), lowerBound) // {x:-10, y:-134}
let test6 = max(Vector2(x: 10, y: 134), upperBound) // {x:10, y:134}
```

Let’s fix them by providing our own specialized versions.

``` swift
func min(lhs: Vector2, rhs: Vector2) -> Vector2 {
    return Vector2(x: min(lhs.x, rhs.x), y: min(lhs.y, rhs.y))
}

func max(lhs: Vector2, rhs: Vector2) -> Vector2 {
    return Vector2(x: max(lhs.x, rhs.x), y: max(lhs.y, rhs.y))
}

let test5 = min(Vector2(x: -10, y: -134), lowerBound) // {x:-100, y:-134}
let test6 = max(Vector2(x: 10, y: 134), upperBound) // {x:100, y:134}
```

This looks better in the sense that the min and max is calculated per
component. But, something is still wrong with our clamp tests, as they
still print the same old value. Turns out the min and max within clamp
function still use the standard generic function, rather than our
provided specialized version.

The only way to make this work is if I provide a specialized clamp
function for Vector2.

``` swift
func clamp(value:Vector2, lowerBound:Vector2, upperBound:Vector2) -> Vector2 {
    return min(max(lowerBound, value), upperBound)
}

let test1 = clamp(Vector2(x: 200, y: 200), lowerBound, upperBound) // {x:100, y:100}
let test2 = clamp(Vector2(x: -200, y: -200), lowerBound, upperBound) // {x:-100, y:-100}
let test3 = clamp(Vector2(x: -10, y: -134), lowerBound, upperBound) // {x:-10, y:-100}
let test4 = clamp(Vector2(x: 10, y: 134), lowerBound, upperBound) // {x:10, y:100}
```

Note that the specialized clamp implementation for Vector2 is exactly
the same as the generic one. And this is not good, as now if we want the
compiler to automatically pick the right version, we have to implement
the entire chain down to every function. So, it comes down to either
using the generic functions all the way up or implementing the entire
chain.

To compare, here’s a C++ version of the same functionality that works
great with a single clamp generic implementation.

``` cpp
#include <iostream>

struct Vector2 {
    float x;
    float y;

    Vector2(float x = 0, float y = 0) {
        this->x = x;
        this->y = y;
    }
};

template <typename T>
T min(T a, T b) {
    return a < b ? a : b;
}

template <>
Vector2 min(Vector2 a, Vector2 b) {
    return Vector2(min(a.x, b.x), min(a.y, b.y));
}

template <typename T>
T max(T a, T b) {
    return a > b ? a : b;
}

template <>
Vector2 max(Vector2 a, Vector2 b) {
    return Vector2(max(a.x, b.x), max(a.y, b.y));
}

template <typename T>
T clamp(T value, T lowerBound, T upperBound) {
    return min(max(lowerBound, value), upperBound);
}

std::ostream &operator<<(std::ostream &os, const Vector2 &v) {
    os << v.x << ", " << v.y;
    return os;
}

int main() {
    Vector2 lowerBound(-100, -100);
    Vector2 upperBound(100, 100);

    std::cout << clamp(Vector2(200, 200), lowerBound, upperBound) << std::endl;
    std::cout <<  clamp(Vector2(-200, -200), lowerBound, upperBound) << std::endl;
    std::cout << clamp(Vector2(-10, -134), lowerBound, upperBound) << std::endl;
    std::cout <<  clamp(Vector2(10, 134), lowerBound, upperBound) << std::endl;

    std::cout <<  min(Vector2(-10, -134), lowerBound) << std::endl;
    std::cout <<  max(Vector2(10, 134), upperBound) << std::endl;
}
```

In conclusion, generic programming with Swift is great and definitely a
step forward than Objective-C, but somebody coming from C++ would be a
little disappointed. The bright side is that the Swift language is
rapidly evolving and maybe this would get improved in the future
version.

The entire code for this rant is available at
[C++](https://gist.github.com/chunkyguy/5c11fbe4b52b634dade7) and
[Swift](https://gist.github.com/chunkyguy/d3aa8b27ace2abd44f4b).

