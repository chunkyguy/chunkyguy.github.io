---
layout: post
title: Immutability - Swift vs C++
date: 2019-11-28 17:12:00
categories: swift cpp
published: true
---

Immutability awareness is on the rise. Especially when the codebase grows in size and complexity, the benefits of immutability for stability, readability and maintenance reasons is imminent. 

If you're dealing a function that receives immutable data as arguments, you don't have to bother checking for any side effects. Or worrying about accidentally mutating the received data.
```cpp
void process(const Page & page) {
    // ...
}
```

Similar benefits can be observed from the other side. If you don't expect data to be mutated, you might prefer marking the entire method as immutable.

```cpp
struct Page {
    bool print() const {
        // ...
        return true;
    }
};

void process(const Page &page) {
    bool status = page.print();
    // ...
}
```

But things get a bit complicated when you're not dealing with the first hand data, rather a reference to the data. Since the reference is just a data holding the data. So depending on the intended design, either the data or the reference could be mutable, or both, or none. 

As we can see, we are dealing with 2 things here each of which can then be in 2 states. So overall we are dealing with 4 cases:

- [Everything is mutable](#everything-is-mutable)
- [Reference is mutable](#reference-is-mutable)
- [Data is mutable](#data-is-mutable)
- [Everything is immutable](#everything-is-immutable)


### Everything is mutable

Let's consider a simple C++ data structure

```cpp
struct Foo {
    int x;
};
```

Marking everything as mutable is very easy. Do nothing special.

```cpp
void update_all(Foo *f) {
    f->x = 10;
    f = nullptr;
}
```

With Swift we need to make the decision about mutability while designing the data structures. Since the data within a `struct` is always immutable and within a `class` may or may not be mutable. The mutability of reference depends on whether is a `let` or a `var`.

So, in case of a `struct` we would have to explicitly set the data and the reference to be `var`

```swift
struct A {
    var x = 0
}

func updateAllStruct() {
    var f = A()
    f.x = 10
    f = A()
}
```

Note that since `A` is `struct` type, so every mutating operation internally creates a new copy of the entire data structure.

The `class` behaves just like the C++ counterpart as long as keep both data and reference as `var`

```swift
class B {
    var x = 0
}

func updateAllClass() {
    var f = B()
    f.x = 10
    f = B()
}
```

### Reference is mutable

If we only need a mutable reference but the data should remain immutable. With C++ we simply need to mark the pointed data as `const`. If we then accidentally try to mutate the data, the compiler would throw an error.

```cpp
void update_ref(Foo const * f) {
    f->x = 10; // error: Cannot assign to variable 'f' with const-qualified type 'const Foo *'
    f = nullptr; // good
}
```

In Swift, if we wish the data to be immutable, we have to mark it as `let` while keeping the reference as `var`

```swift
struct C {
    let x = 0
}

class D {
    let x = 0
}
```

And then, if we accidentally try to mutate the via an mutable reference, the compiler would throw an error

```swift
func updateRefStruct() {
    var f = C()
    f.x = 10 // error: Cannot assign to property: 'x' is a 'let' constant
    f = C() // good
}

func updateRefClass() {
    var f = D()
    f.x = 10 // error: Cannot assign to property: 'x' is a 'let' constant
    f = D() // good
}
```

### Data is mutable

With C++, marking the data as mutable while the reference remains immutable also seems quite straightforward. Simply mark the reference as `const`

```cpp
void update_data(Foo * const f) {
    f->x = 10; // good
    f = nullptr; // error: Cannot assign to variable 'f' with const-qualified type 'Foo *const'
}
```

In Swift however, having a `struct` where reference is immutable while data as mutable is not possible. Because, as soon as you mark the reference as `let`, the internal storage becomes immutable as well, even if it was marked as `var`

```swift
func updateDataStruct() {
    let f = A()
    f.x = 10 // error: Cannot assign to property: 'f' is a 'let' constant
    f = A() // error: Cannot assign to value: 'f' is a 'let' constant
}
```

But, again we can use `class` type to get the same behavior as C++

```swift
func updateDataClass() {
    let f = B()
    f.x = 10 // good
    f = B() // error: Cannot assign to value: 'f' is a 'let' constant
}
```

### Everything is immutable

Marking everything as immutable with C++ is definitely possible, but requires a lot of extra typing. You need to mark both the reference and the pointed data as `const`

```cpp
void update_none(Foo const * const f) {
    f->x = 10; // error: Cannot assign to variable 'f' with const-qualified type 'const Foo *const'
    f = nullptr; // error: Cannot assign to variable 'f' with const-qualified type 'const Foo *const'
}
```

With Swift, if we mark both the reference and the data as `let` everything becomes immutable

```swift
func updateNoneStruct() {
    let f = C()
    f.x = 10 // error: Cannot assign to property: 'x' is a 'let' constant
    f = C() // error: Cannot assign to value: 'f' is a 'let' constant
}

func updateNoneClass() {
    let f = D()
    f.x = 10 // error: Cannot assign to property: 'x' is a 'let' constant
    f = D() // error: Cannot assign to value: 'f' is a 'let' constant
}
```

### Opinions?

I personally like C++ way of dealing with mutability. It seems simpler to understand and offers more control to the user of the API. Compared to Swift where you need to look the implementation of the library to see if the type is a `class` or a `struct`.

Also from the author's perspective, they only have to design the class intrinsic behavior for 'const correctness' by marking appropriate methods as only available when accessed via as `const`. 

What's even more interesting is the fact that the authors can even override methods for `const` to give even more flexible class interface

```cpp
struct Foo {
    int increment() { return ++x; }
    int increment() const { return x + 1; }
    int x = 0;
};

void f() {
    const Foo f;
    int y = f.increment();
    printf("%d %d\n", y, f.x); // 1 0
}

void g() {
    Foo f;
    int y = f.increment();
    printf("%d %d\n", y, f.x); // 1 1
}
```

In fact, this technique is used by the `std::map operator[]`. It's called [const-overloading](https://isocpp.org/wiki/faq/const-correctness#const-overloading)

The thing I like about Swift though is the **immutable by default** behavior in Swift, which makes code safer by default, unlike C++ where you have to discipline yourself to it. And some cases where having things as immutable is even [frowned upon](https://stackoverflow.com/questions/6954906/does-c11-allow-vectorconst-t).