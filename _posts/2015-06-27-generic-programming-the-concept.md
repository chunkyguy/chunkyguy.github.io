---
layout: post
title:  "Generic Programming: The Concept"
date:   2015-06-27 23:28:54 +0530
categories: programming generic
---

Generic Programming: The Concept
================================

Say, we have a type `Animal` which is at this time just an idea, so a
protocol.

``` {.brush: .cpp; .title: .; .notranslate title=""}
protocol Animal {

    func say()

}
```

Next, say we want a generic function that works with `Animal`.

``` {.brush: .cpp; .title: .; .notranslate title=""}
func greet<T: Animal>(animal: T) {

    animal.say()

}
```

Looks good. Now let’s create our concrete types.

``` {.brush: .cpp; .title: .; .notranslate title=""}
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

``` {.brush: .cpp; .title: .; .notranslate title=""}
let m = Mammal()

let r = Reptile()



let animals: [Animal] = [m, r]



for animal in animals {

    greet(animal)

}
```

Oh o! Unfortunately this code won’t compile. This is the error the
compiler might throw on your machine.

``` {.brush: .cpp; .title: .; .notranslate title=""}
error: generic parameter 'T' cannot be bound to non-@objc protocol type 'Animal'
```

Let’s rewrite our implementation in C++ and see what sort of error we
get.

``` {.brush: .cpp; .title: .; .notranslate title=""}
class Animal {

public:

    virtual void Say() const = 0;

};



template <typename T>

void Greet(const T &amp;t) {

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



    std::for_each(animals.begin(), animals.end(), [](const Animal &amp;animal) {

        animal.Say();

    });



    return 0;

}
```

If you compile this with your C++11 compiler
`clang++ -std=c++11 -stdlib=libc++ code.cpp`, you might get an error
such as

``` {.brush: .cpp; .title: .; .notranslate title=""}
error: allocating an object of abstract class type 'Animal'
```

But, we are never allocating the instance of type `Animal` right? This
is time to get our understanding of Generic Programming better.

Generic Programming works around these main terminologies: The
**concept**, **model**, **refinement** and **type constraint**.

Concept
-------

Concept is the core design. Model is what conforms to the concept, the
implementation of Concept. Refinement is like inheriting a Concept and
adding more things to it. And Type Constraint deals with specifying what
and what not a Type is expected to do

In C++, we can have a `pure virtual base class`, where the base class
provides no implementation for any member function. This idea is
implemented with a `protocol` in Swift.\
Along with that we have something in Swift that C++ programmers are not
used to. It’s the use of `protocol` for type constraints. With C++, the
compiler implicitly adds the type constraints for us.

For example, in the following example,

``` {.brush: .cpp; .title: .; .notranslate title=""}
protocol Equatable {

    func ==(lhs: Self, rhs: Self) -> Bool

}



struct Student {

    let studentId: Int

}



func ==(lhs: Student, rhs: Student) -> Bool {

    return lhs.studentId == rhs.studentId

}



extension Student: Equatable {}
```

the `Equatable` protocol is an explicit type constraint. But since we’re
using the term `protocol` with `Equatable`, it could be a source of
confusion to many.\
The point to remember when working with Swift is that in swift
`protocol` is used both for **concept** and **type constraint**. Whereas
in C++ we don’t have any way to design **type constraint**.

Let’s explore this theory with an example of a generic stack
implementation:

``` {.brush: .cpp; .title: .; .notranslate title=""}
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

What could be the core **concept** of a stack? If we take a look at it,
we might say a stack is a data structure where you push items and later
on pop them back. So, we can say the concept of stack can be written as

``` {.brush: .cpp; .title: .; .notranslate title=""}
protocol Stack {

    typealias Item

    func push(item: Item)

    func pop() -> Item?

}
```

Now, we can model an Array based stack on this concept as:

``` {.brush: .cpp; .title: .; .notranslate title=""}
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

Or, we can go crazy and write a weird stack, where you pop data filtered
by some epoch timestamp that we provide at runtime.

``` {.brush: .cpp; .title: .; .notranslate title=""}
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

You get the idea. A **concept** is just the bare minimum abstraction of
the idea that has to modeled by some other implementation for use.

Now, coming back to the original problem. What was wrong with our
`Animal` example at the top?

The problem is with both the implementations. In the Swift
implementation, we’ve designed our `Animal` as a **type constraint** and
with our c++ implementation, as our dear clang compiler was trying to
inform us, the `Animal` is actually a **concept**. And hence both the
usages are invalid.

That’s all for today. We shall explore the other Generic Programming
terminologies later someday.\
Have a nice day!