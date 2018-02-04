---
layout: post
title:  "Generic Programming: The Concept"
date:   2015-06-27 23:28:54 +0530
categories: programming generic
---

### Problem

Say, we have a type `Animal` which is at this time just an idea, so a protocol.

```
protocol Animal {
    func say()
}
```

Next, say we want a generic function that works with `Animal`.

```
func greet<T: Animal>(animal: T) {
    animal.say()
}
```

Looks good. Now let’s create our concrete types.

```
class Mammal: Animal {
    func say() {
        println("Hello! I'm a mammal!")
    }
}

class Reptile: Animal {
    func say() {
        println("Hello! I'm a replite")
    }
}
```

And bring it to use

```
let m = Mammal()
let r = Reptile()
let animals: [Animal] = [m, r]

for animal in animals {
    greet(animal)

}
```

Oh o! Unfortunately this code won’t compile. This is the error the compiler might throw on your machine.

```
error: generic parameter 'T' cannot be bound to non-@objc protocol type 'Animal'
```

Let’s rewrite our implementation in C++ and see what sort of error we get.

```
class Animal {
public:
    virtual void Say() const = 0;
};

template <typename T>
void Greet(const T &t) {
    t.Say();
}

class Mammal: public Animal {
public:
    void Say() const {
        std::cout << "Hello! I'm a mammal." << std::endl;
    }
};

class Reptile: public Animal {
public:
    void Say() const {
        std::cout << "Hello! I'm a reptile." << std::endl;
    }
};

int main() {
    Mammal m;
    Reptile r;
    std::vector<Animal> animals = {m, r};
    std::for_each(animals.begin(), animals.end(), [](const Animal &animal) {
        animal.Say();
    });
    return 0;
}
```

If you compile this with your C++11 compiler `clang++ -std=c++11 -stdlib=libc++ code.cpp`, you might get an error such as:

```
error: allocating an object of abstract class type 'Animal'
```

But, we are never allocating the instance of type `Animal` right? This is time to get our understanding of Generic Programming better.

### Terminology

Generic Programming works around these main terminologies:

1. `Concept`
2. `Model`
3. `Refinement`
4. `Type Constraint`

`Concept` is the core design or interface. `Model` is what conforms to the `Concept` and provides a implementation of the `Concept`. `Refinement` is like inheriting a `Concept` and adding more things to it. And finally `Type Constraint` deals with specifying what and what not a `Type` that wishes to use the `Concept` is allowed to do.

In C++, we can have a `pure virtual base class`, where the base class provides no implementation for any member function. Swift hipster might know this idea as a `protocol`.

Along with that we have something in Swift that C++ programmers are not used to. It’s the use of `protocol` for `Type Constraint`. With C++, the compiler implicitly adds the `Type Constraint` for us.

Let's take a look at some examples.

This is a `Concept`:

```
class Animal {
    public:
    virtual void Say() const = 0;
};
```

This is a `Model`:
```
class Mammal: public Animal { ... }
```

This is `Refinement`:
```
class WalkingAnimal: public Animal {
    public:
    virtual void Walk() const = 0;
};
```

This is a `Type Constraint`:
```
protocol Equatable {
    func ==(lhs: Self, rhs: Self) -> Bool
}
```

In Swift we use the term `protocol` for both `Concept` and `Type Constraint`, and this could be a source of confusion to many. Whereas, in C++ we don’t have any way to design `Type Constraint`, which helps when writing code, but when the code fails to compile for some reaons it might get hard to understand why.

First let us try to understand the relationship between a `Model` and a `Type Constraint`. Let's say we have a  `Model` named `Student` as:

```
struct Student: Equatable {
    let studentId: Int
}

func ==(lhs: Student, rhs: Student) -> Bool {
    return lhs.studentId == rhs.studentId
}
```
Here `Student` conforms to the `Type Constraint`. So any function that is designed to with types `Equatable` can work with `Student`. But, we have to keep in mind that the `Student` is **not** a `Model` of `Concept` named `Equatable`.


Next let us explore the relationship between a `Model` and a `Concept`. If we have a generic stack implementation:

```
class Stack<T> {
    private var store: [T] = []

    func push(item: T) {
        store.append(item)
    }

    func pop() -> T? {
        if store.isEmpty {
            return nil
        }
        return store.removeLast()
    }
}
```

What could be the core `Concept` of a stack? If we take a look at it, we might say a stack is a data structure where you push items and later on pop them back. So, we can say the `Concept` of stack can be written as:

```
protocol Stack {
    typealias Item
    func push(item: Item)
    func pop() -> Item?
}
```

Now, we can `Model` an `Array` backed stack on this `Concept` as:

```
class ArrayStack<T>: Stack {
    private var store: [T] = []

    func push(item: T) {
        store.append(item)
    }

    func pop() -> T? {
        if store.isEmpty {
            return nil
        }
        return store.removeLast()
    }
}
```

Or, we can go crazy and write a weird stack, where you pop data filtered by some epoch timestamp that we provide at runtime.

```
class TimeStack<T>: Stack {
    private var store: [CFTimeInterval: T] = [:]
    var epoch = CACurrentMediaTime()

    func push(item: T) {
        store[CACurrentMediaTime()] = item
    }

    func pop() -> T? {
        for (time, value) in store {
            if time < epoch {
                return value
            }
        }

        return nil
    }
}
```

### Solution

With our basics now clear, we can take a look again at the original problem. What was wrong with our `Animal` example at the top?

The problem is with both the implementations.

In the Swift implementation, we’ve designed our `Animal` as a `Type Constraint`, as can be observed with the `func greet<T: Animal>`, but then later we are trying to use it as `Concept` by doing things like `class Mammal: Animal`. And this is what compiler is complaining about.

With our C++ implementation, as our dear clang compiler was trying to inform us, the `Animal` is actually a `Concept`, but we are trying to use it as `Type Constraint` by doing things like `std::vector<Animal>`.

And hence both the usages are invalid.

That’s all for today. We shall explore the other Generic Programming terminologies later someday.

Have a nice day!
