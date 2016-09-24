
---
layout: post
title:  "Changing the gears to Generic Programming"
date:   2015-06-20 11:32:54 +0530
categories: jekyll update
---

Practical usage of generic programming can be narrowed down to two
forms:

1\. Writing top level generic functions\
2. Writing generic custom types.

**Generic Functions**

Writing top level generic functions is easier. Its more of like you
write a normal function and later on you realize that this function is
not restricted to any type, rather it is just an abstract algorithm that
can be used by many types, you upgrade it to generic type. For example,

``` {.brush: .cpp; .title: .; .notranslate title=""}

bool isEqual(int x, int y) {

    return x == y;

}
```

can be easily abstracted out to a generic form such as

``` {.brush: .cpp; .title: .; .notranslate title=""}

template <typename T>

bool isEqual(T x, T y) {

    return x == y;

}
```

This generic form would work as long as `T` supports a `==` operation.
Which brings us to the term you will often find with generic
programming, type constraints.

So, the above example would work fine with all types that support ==
operation. Such as:

``` {.brush: .cpp; .title: .; .notranslate title=""}

    std::cout << (isEqual(10, 10) ? "Y" : "N") << std::endl;

    std::cout << (isEqual(10.5, 10.5) ? "Y" : "N") << std::endl;

    std::cout << (isEqual("hello", "hello") ? "Y" : "N") << std::endl;
```

But would easily break down when used like:

``` {.brush: .cpp; .title: .; .notranslate title=""}

struct Vector2 {

    float x, y;

    Vector2(float xx, float yy) : x(xx), y(yy) {}

};



std::cout << (isEqual(Vector2(10, 10), Vector2(10, 10)) ? "Y" : "N") << std::endl; // output compilation failure
```

And the reason being, that the type constraint has been violated. The
type `Vector2` does not provides an `==` operation. The fix is simple,
satisfy the constraints

``` {.brush: .cpp; .title: .; .notranslate title=""}

bool operator == (const Vector2 &lhs, const Vector2 &rhs) {

    return (lhs.x == rhs.x) && (lhs.y == rhs.y);

}
```

Another type of error that could arise with generic functions is, where
the constraints are not violated syntactically, by are logically broken.
For example:

``` {.brush: .cpp; .title: .; .notranslate title=""}

    const char w1[] = "world";

    const char w2[] = "world";

    std::cout << (isEqual(w1, w2) ? "Y" : "N") << std::endl; // output N
```

Here, our `isEqual()` sees that the types are `const char *` and the
compiler knows how to compare two pointers. But the result might or
might not be what we were expecting. If you were expecting a string
comparison, you’ve to tell the compiler how to do that.

For our specific case, it can be achieved by providing an overloaded
implementation that just takes `const char *`

``` {.brush: .cpp; .title: .; .notranslate title=""}

bool isEqual(const char *x, const char *y) {

    return strcmp(x, y) == 0;

}
```

 

**Generic Types**

The main power of generic programming lies when working with generic
types. Just as generic functions abstract the algorithm to a type-free
level, generic types abstract our design to type free level.

Think of generic types as if us providing the type information for the
compiler, and passing the responsibility of creating the actual types to
the compiler.

To get an quick overview, the `Vector2` type that we wrote above for
`float` types can easily be rewritten as a generic type

``` {.brush: .cpp; .title: .; .notranslate title=""}

template <typename T>

struct Vector2 {

    T x, y;

    Vector2(T xx, T yy) : x(xx), y(yy) {}

};
```

And out `==` implementation should just fit with the change:

``` {.brush: .cpp; .title: .; .notranslate title=""}

template <typename T>

bool operator == (const Vector2<T> &lhs, const Vector2<T> &rhs) {

    return (lhs.x == rhs.x) && (lhs.y == rhs.y);

}
```

Now, where should we actually use the generic types in real world? To
address this question, you’ve to switch you mindset from run-time
polymorphism to compile-time polymorphism. Let’s consider an example
where you have designed your types in the classical object-oriented
fashion, that is using inheritence

``` {.brush: .cpp; .title: .; .notranslate title=""}

class Shape {

public:

    virtual ~Shape() {}



    void PrintArea() {

        std::cout << "area = " << Area() << std::endl;

    }



private:

    virtual float Area() const = 0;

};



class Rectangle: public Shape {

public:

    Rectangle(float w, float h) : width(w), height(h) {}



private:

    float Area() const {

        return width * height;

    }



    float width, height;

};





class Circle: public Shape {

public:

    Circle(float r) : radius(r) {}



private:

    float Area() const {

        return 3.14 * (radius * radius);

    }



    float radius;

};



int main() {

    Rectangle *r = new Rectangle(10, 20);

    Circle *c = new Circle(5);



    std::vector<Shape *> shapes;

    shapes.push_back(r);

    shapes.push_back(c);



    std::for_each(shapes.begin(), shapes.end(), [](Shape *s) {

        s->PrintArea();

    });



    delete r;

    delete c;



    return 0;

}
```

Here, we have a base `Shape` class that does provides the public
interface and leaves the actual implementation details to the subclass,
in this case the actual calculation of the area. Internally, we’re
calling the `Area()` function on `Shape` and let the run-time do the
actual look-up for the implementation by looking at the type
information.

In another words, what we are actually doing is that delegating some
part of the implementation of our `Shape` class to subclasses. Another
way of achieving the same results is by not depending on subclasses for
overrides, but delegating the implementation to some external classes.

``` {.brush: .cpp; .title: .; .notranslate title=""}

struct ShapeImpl {

    virtual float Area() const = 0;

};



class RectangleImpl: public ShapeImpl {

public:

    RectangleImpl(float w, float h) : width(w), height(h) {}



private:

    float Area() const {

        return width * height;

    }



    float width, height;

};



class CircleImpl: public ShapeImpl {

public:

    CircleImpl(float r) : radius(r) {}



private:

    float Area() const {

        return 3.14 * (radius * radius);

    }



    float radius;

};



class Shape {

public:

    Shape(const ShapeImpl *impl) : implementation(impl) {}



    void PrintArea() const {

        std::cout << "area = " << implementation->Area() << std::endl;

    }



private:

    const ShapeImpl *implementation;

};



int main() {

    RectangleImpl *r = new RectangleImpl(10, 20);

    CircleImpl *c = new CircleImpl(5);



    std::vector<Shape> shapes;

    shapes.push_back(Shape(r));

    shapes.push_back(Shape(c));



    std::for_each(shapes.begin(), shapes.end(), [](const Shape &s) {

        s.PrintArea();

    });



    delete r;

    delete c;



    return 0;

}
```

I don’t know if we’ve achieved any actual improvement over the earlier
implementation with this or not. But, this gives you an idea that the
implementation can be dragged out of the core class while keeping the
visible interface of the `Shape` intact.

If we can achieve this, we can even go a step further and actually
provide the implementation at compile time. In other words, we can
simply make the `Shape` class generic, and let the compiler plug-in the
implementation details at compile-time.

``` {.brush: .cpp; .title: .; .notranslate title=""}

template <typename Impl>

class Shape {

public:

    Shape(const Impl impl) : implementation(impl) {}



    void PrintArea() const {

        std::cout << "area = " << implementation.Area() << std::endl;

    }



private:

    const Impl implementation;

};
```

Now, we can create and use our generic shape instances as:

``` {.brush: .cpp; .title: .; .notranslate title=""}

    Shape<RectangleImpl> rect(RectangleImpl(10, 20));

    Shape<CircleImpl> circle(CircleImpl(5));



    rect.PrintArea();

    circle.PrintArea();
```

The only type constraints on the `RectangleImpl` and `CircleImpl` is
that they need to have a `Area()` function.

``` {.brush: .cpp; .title: .; .notranslate title=""}

class RectangleImpl {

public:

    RectangleImpl(float w, float h) : width(w), height(h) {}



    float Area() const {

        return width * height;

    }



private:

    float width, height;

};



class CircleImpl {

public:

    CircleImpl(float r) : radius(r) {}



    float Area() const {

        return 3.14 * (radius * radius);

    }



private:

    float radius;

};
```

The c++ compiler is smart enough to see the constraints are not violated
by any new delegate class. For example if we introduce a `TriangleImpl`
as:

``` {.brush: .cpp; .title: .; .notranslate title=""}

class TriangleImpl {

public:

    TriangleImpl(float b, float h) : base(b), height(h) {}



private:

    float base, height;

};



   Shape<TriangleImpl> triangle(TriangleImpl(10, 5));

   triangle.PrintArea();


```

The compiler will immediately throw error messages, unless you fix your
implementation by providing the `Area()` function.

``` {.brush: .cpp; .title: .; .notranslate title=""}

class TriangleImpl {

public:

    TriangleImpl(float b, float h) : base(b), height(h) {}



    float Area() const {

        return 0.5 * base * height;

    }



private:

    float base, height;

};
```

Now, if you notice that we are not using an `std::vector` anymore, that
is because we don’t have a `Shape` type anymore. And if you find
yourself thinking in this direction, you need to get out of the run-time
polymorphic mindset. When using compile-time polymorphism you’ve to keep
in mind the fact that now you’re not in control of creating the custom
types, rather you’ve passed on that authority to the compiler.

So, `Shape<TriangleImpl>` and `Shape<CircleImpl>` are entirely different
types with no common ancestor or anything.

When designing with compile time polymorphism you need to keep in mind
that your generic type should be a leaf node or a final class in the
entire type system. You should not have to subclass your generic type,
rather plug-in a delegate class that provides the implementation that
you would rather provide by sub-classing.

To summarize, here’s a quick list of good and bad of generic type
system:

**Pro**\
1. Robust: No run time exceptions.\
2. Efficient: No run time lookups

**Cons**\
1. Not a good candidate for base class\
2. Hard to read and debug.

Now, moving on to using generics with Swift. Swift provides the same
good old c++ way of writing generic code with some extra type constraint
system. Where in c++ the compiler would do the type constraint checking
when you actually compile the code, Swift actually makes the constraint
system more explicit, such that you have to provide the constraint
information with a `protocol`.

``` {.brush: .cpp; .title: .; .notranslate title=""}

protocol ShapeImplementable {

    var area: Double { get }

}



struct Shape<Impl: ShapeImplementable> {

    private let implementation: ShapeImplementable



    init(impl: ShapeImplementable) {

        implementation = impl

    }



    func printArea() {

        println("area = \(implementation.area)")

    }

}
```

And then you can implement your delegate class as

``` {.brush: .cpp; .title: .; .notranslate title=""}

struct RectangleImpl: ShapeImplementable {

    var area: Double {

        return width * height

    }



    init(w: Double, h: Double) {

        width = w

        height = h

    }



    private let width: Double

    private let height: Double

}
```

And finally use them the same way you would with c++.

``` {.brush: .cpp; .title: .; .notranslate title=""}

let rect = Shape<RectangleImpl>(impl: RectangleImpl(w: 10, h: 20))
```

The entire code for this article is also available at
<https://github.com/chunkyguy/GenericsDemo>

Goodbye and have a nice day!
