---
layout: post
title:  "C++ for Swift developers"
date:   2021-01-03 15:30:00 +0200
categories: swift cpp languages
published: true
---

Swift in a sense is very much like C++, and when I say C++ I mean C++11 and beyond. One could also say that Swift is *cleaner* C++, or C++ without the backwards compatibility baggage from the 80s. To give an idea here's a minimal *modern* C++ code:

```cpp
#include <iostream>       // 1
using namespace std;      // 2
auto main() -> int {      // 3
  cout << "Hello world!"; // 4
  return 0;               // 5
}
```

Now lets break them down from Swift perspective.
```cpp
#include <iostream> // 1
```

Swift equivalent would be `import iostream`. Frameworks usually come with an umbrella header as a single file, so typically we need to add only one include statement per framework like in Swift. But unlike Swift, we need to add include statements for every file we want to use from our own code.

```cpp
#include <framework>      // external framework
#include "path/to/file.h" // local file
```

```cpp
using namespace std; // 2
```

Unlike Swift, C++ has a proper support for namespaces. So if two types in different frameworks have same type name they can be resolved using their namespaces. For example `fwkA::JSON` and `fwkB::JSON`. To get a more Swift like behavior we can add `using` to indicate that the symbols do not need their namespace for every single usage.

```cpp
auto main() -> int { // 3
  // ...
}
```

This is the modern way of writing functions in C++. In classic fashion this function could also be written as:

```cpp
int main() {
  std::cout << "Hello world!";
  return 0;
}
```

```cpp
cout << "Hello world!"; // 4
```

Like Swift we can also override operators in C++. In this case the `iostream` library provides `cout` which is an object of type `ostream` that outputs to *standard output stream*. `iostream` also provides overrides for `operator <<` for many common types like for `string` that we're using above. But it can also work for our custom types if we provide the `operator <<` for our type. The Swift equivalent to this would be the `CustomStringConvertible` protocol.

```cpp
return 0; // 5
```

And finally since our function requires a return type `int` we have to end our function with a `return` statement.

With a basic introduction, now let us take a look at C++ in a bit more detail from Swift perspective.

- [Variables](#variables)
- [String](#string)
- [Array](#array)
- [Dictionary](#dictionary)
- [Optional](#optional)
- [Function](#function)
- [Closures](#closures)
- [Classes](#classes)
- [Error handling](#error-handling)
- [Generics](#generics)

## Variables

<table>
<thead><tr><th>Swift</th><th>C++</th></tr></thead>
<tbody><tr>
<td><pre><code class="language-swift">
var myVariable = 42
myVariable = 50 
let myConstant = 42
</code></pre></td>
<td><pre><code class="language-cpp">
auto myVariable = 42;
myVariable = 50;
const auto myConstant = 42;
</code></pre></td>
</tr></tbody>
</table>

Just like in Swift types can be deduced by the compiler using `auto`, but if required we can always also provide the type information explicitly.

<table>
<thead><tr><th>Swift</th><th>C++</th></tr></thead>
<tbody><tr>
<td><pre><code class="language-swift">
let myFloat: Float = 3.14
</code></pre></td>
<td><pre><code class="language-cpp">
float myFloat = 3.14;
</code></pre></td>
</tr></tbody>
</table>

Unlike Swift, type conversions can also occur implicitly in C++. This needs to be carefully watched out for to avoid hard to find bugs. For other cases we need to explicitly convert data from a type to another. For example `int` to `string`

<table>
<thead><tr><th>Swift</th><th>C++</th></tr></thead>
<tbody><tr>
<td><pre><code class="language-swift">
let label = "The width is "
let width = 94
let widthLabel = label + String(width)
</code></pre></td>
<td><pre><code class="language-cpp">
auto label = "The width is ";
auto width = 94;
auto widthLabel = label + to_string(width);
</code></pre></td>
</tr></tbody>
</table>

## String

String interpolation in C++ isn't as good as with Swift. But nonetheless we can construct `string` with either explicit string conversion, like we saw above

<table>
<thead><tr><th>Swift</th><th>C++</th></tr></thead>
<tbody><tr>
<td><pre><code class="language-swift">
let apples = 3
let oranges = 5
let appleSummary = "I have \(apples) apples."
let fruitSummary = "I have \(apples + oranges) pieces of fruit."
</code></pre></td>
<td><pre><code class="language-cpp">
auto apples = 3;
auto oranges = 5;
auto appleSummary = "I have " + to_string(apples) + " apples.";
auto fruitSummary = "I have " + to_string(apples + oranges) + " pieces of fruit.";
</code></pre></td>
</tr></tbody>
</table>

Or using `ostringstream` which works just like `cout` and makes use of `operator <<`

```cpp
auto apples = 3;
ostringstream ss;
ss << "I have " << apples << " apples.";
auto str = ss.str();
```

which can then further be reduced to:

```cpp
auto str = (ostringstream()<<"I have "<<apples<<" apples.").str();
```

<table>
<thead><tr><th>Swift</th><th>C++</th></tr></thead>
<tbody><tr>
<td><pre><code class="language-swift">
let apples = 3
let oranges = 5
let appleSummary = "I have \(apples) apples."
let fruitSummary = "I have \(apples + oranges) pieces of fruit."
</code></pre></td>
<td><pre><code class="language-cpp">
auto apples = 3;
auto oranges = 5;
auto appleSummary = (ostringstream()<< "I have "<<apples<<" apples.").str();
auto fruitSummary = (ostringstream()<<"I have "<<(apples + oranges)<<" pieces of fruit.").str();
</code></pre></td>
</tr></tbody>
</table>

## Array

Swift `Array` equivalent in C++ is `vector`. Here's how you use it

<table>
<thead><tr><th>Swift</th><th>C++</th></tr></thead>
<tbody><tr>
<td><pre><code class="language-swift">
var shoppingList = ["catfish", "water", "tulips"]
shoppingList[1] = "bottle of water"
shoppingList.append("blue paint")
</code></pre></td>
<td><pre><code class="language-cpp">
auto shoppingList = vector&lt;string&gt;{"catfish", "water", "tulips"};
shoppingList[1] = "bottle of water";
shoppingList.push_back("blue paint");
</code></pre></td>
</tr></tbody>
</table>

And this is how you iterate a `vector`

<table>
<thead><tr><th>Swift</th><th>C++</th></tr></thead>
<tbody><tr>
<td><pre><code class="language-swift">
for item in shoppingList {
  print(item)
}
</code></pre></td>
<td><pre><code class="language-cpp">
for (auto & item : shoppingList) {
  cout << item << endl;
}
</code></pre></td>
</tr></tbody>
</table>

The `&` here means that we wish to use `item` as a reference. If we were to use `auto item` it would always create a new copy.

## Dictionary

Swift `Dictionary` equivalent in C++ is `map`. Here's how you use it

<table>
<thead><tr><th>Swift</th><th>C++</th></tr></thead>
<tbody><tr>
<td><pre><code class="language-swift">
var occupations = [
  "Malcolm": "Captain",
  "Kaylee": "Mechanic",
]
occupations["Jayne"] = "Public Relations"
</code></pre></td>
<td><pre><code class="language-cpp">
auto occupations = map&lt;string, string&gt; {
  {"Malcolm", "Captain"},
  {"Kaylee", "Mechanic"},
};
occupations["Jayne"] = "Public Relations";
</code></pre></td>
</tr></tbody>
</table>

And this is how you iterate over a `map`

<table>
<thead><tr><th>Swift</th><th>C++</th></tr></thead>
<tbody><tr>
<td><pre><code class="language-swift">
for (key, value) in occupations {
  print("\(key) : \(value)")
}
</code></pre></td>
<td><pre><code class="language-cpp">
for (auto& [key, value]: occupations) {
  cout << key << " : " << value << endl;
}
</code></pre></td>
</tr></tbody>
</table>

## Optional

Swift developers love `Optional` types. Fortunately for us, C++ also got `optional` types starting c++17. Although not as fancy as with Swift, but the core idea is the same: A type that either has the value or not. Here's an example:

<table>
<thead><tr><th>Swift</th><th>C++</th></tr></thead>
<tbody><tr>
<td><pre><code class="language-swift">
func greeting(_ required: Bool) -> String? {
  return required ? Optional("Hello") : nil;
}

func main() -> Int {
  let s = greeting(false) ?? "Hi"
  print(s) // Hi

  if let str = greeting(true) {
    print(str) // Hello
  }

  return 0
}
</code></pre></td>
<td><pre><code class="language-cpp">
auto greeting(bool required) {
  return required ? optional{"Hello"} : nullopt;
}

auto main() -> int {
  auto s = greeting(false).value_or("Hi");
  cout << s << endl; // Hi

  if (auto str = greeting(true)) {
    cout << *str << endl; // Hello
  }

  return 0;
}
</code></pre></td>
</tr></tbody>
</table>

The `*str` can be thought of as force unwrapping in Swift, `str!`. And after *dereferencing* we can either use the `.` or the handy `->` operator to access the variable. 

```cpp
str->size();
(*str).size();
```

## Function

C++ functions do not have argument labels

<table>
<thead><tr><th>Swift</th><th>C++</th></tr></thead>
<tbody><tr>
<td><pre><code class="language-swift">
func greet(_ person: String, on day: String) -> String {
    return "Hello \(person), today is \(day)."
}

greet("John", on: "Wednesday")
</code></pre></td>
<td><pre><code class="language-cpp">
auto greetPersonOnDay(string person, string day) -> string {
  return (ostringstream()<<"Hello "<<person<<", today is "<<day<<".").str();
}

greetPersonOnDay("John", "Wednesday")l
</code></pre></td>
</tr></tbody>
</table>

But if you really miss having named arguments there are many tricks of all shapes and sizes out there. The simplest is to use a `struct` for arguments

```cpp
struct Args { string person, day; };
auto greet(Args args) -> string {
  return (ostringstream()<<"Hello "<<args.person<<", today is "<<args.day<<".").str();
}

greet({.person = "John", .day = "Wednesday"})
```

## Closures

C++ has a good support for closures. Here's an example:

<table>
<thead><tr><th>Swift</th><th>C++</th></tr></thead>
<tbody><tr>
<td><pre><code class="language-swift">
func hasAnyMatches(list: [Int], condition: (Int) -> Bool) -> Bool {
  for item in list {
    if condition(item) {
      return true
    }
  }
  return false
}

func lessThanTen(number: Int) -> Bool {
  return number < 10
}

var numbers = [20, 19, 7, 12]
hasAnyMatches(list: numbers, condition: lessThanTen)
</code></pre></td>
<td><pre><code class="language-cpp">
auto hasAnyMatches(vector&lt;int&gt; list, function&lt;bool(int)&gt; condition) -> bool {
  for (auto item : list) {
    if (condition(item)) {
      return true;
    }
  }
  return false;
}

auto lessThanTen(int number) -> bool {
  return number < 10;
}

auto numbers = vector&lt;int&gt; {20, 19, 7, 12};
hasAnyMatches(numbers, lessThanTen);
</code></pre></td>
</tr></tbody>
</table>

That said, C++ still doesn't has all the fancy functional operations like `map`, `filter`, `...` that we get with Swift, but they're coming soon with [C++20 ranges](https://en.cppreference.com/w/cpp/ranges). For now we can use `transform` as an equivalent to Swift `map`

<table>
<thead><tr><th>Swift</th><th>C++</th></tr></thead>
<tbody><tr>
<td><pre><code class="language-swift">
numbers.map({ (number: Int) -> Int in
  let result = 3 * number
  return result
})
</code></pre></td>
<td><pre><code class="language-cpp">
transform(numbers.begin(), numbers.end(), numbers.begin(), [](auto number) -> auto {
  auto result = 3 * number;
  return result;
});
</code></pre></td>
</tr></tbody>
</table>

The C++ code here would actually mutate the `numbers`. If we want the truly Swift `map` equivalent we would have to create an empty `vector` and append data to it

```cpp
auto mappedNumbers = vector<int> {};
transform(numbers.begin(), numbers.end(), back_inserter(mappedNumbers), [](auto n) { 
  return 3 * n; 
});
```

## Classes

Unlike Swift in C++ there isn't much difference between a `struct` and a `class` with the only difference being that a C++ struct has all members `public` by default whereas a `class` has all members as `private` by default. Whether the instance is mutable or not depends on how it is declared.

<table>
<thead><tr><th>Swift</th><th>C++</th></tr></thead>
<tbody><tr>
<td><pre><code class="language-swift">
class Shape {

  func simpleDescription() -> String {
    return "A shape with \(numberOfSides) sides."
  }

  var numberOfSides = 0
}

var shape = Shape()
shape.numberOfSides = 7
var shapeDescription = shape.simpleDescription()
</code></pre></td>
<td><pre><code class="language-cpp">
class Shape {
public:
  auto simpleDescription() -> string {
    return (ostringstream()<<"A shape with "<<numberOfSides<<" sides.").str();
  }

  int numberOfSides = 0;
};

auto shape = Shape();
shape.numberOfSides = 7;
auto shapeDescription = shape.simpleDescription();
</code></pre></td>
</tr></tbody>
</table>

Initializer (or constructor as they are called in C++) have a slightly different syntax

<table>
<thead><tr><th>Swift</th><th>C++</th></tr></thead>
<tbody><tr>
<td><pre><code class="language-swift">
class Shape {

  init(name: String) {
    self.name = name
  }

  func simpleDescription() -> String {
    return "A shape with \(numberOfSides) sides."
  }

  var numberOfSides = 0
  var name: String
}
</code></pre></td>
<td><pre><code class="language-cpp">
class Shape {
public:
  Shape(string name)
    : name(name) {
  }

  auto simpleDescription() -> string {
    return (ostringstream()<<"A shape with "<<numberOfSides<<" sides.").str();
  }

  int numberOfSides = 0;
  string name;
};
</code></pre></td>
</tr></tbody>
</table>

In C++ subclasses need to be more explicit than Swift.

<table>
<thead><tr><th>Swift</th><th>C++</th></tr></thead>
<tbody><tr>
<td><pre><code class="language-swift">
class Square: Shape {

  init(sideLength: Double, name: String) {
    self.sideLength = sideLength
    super.init(name: name)
    numberOfSides = 4
  }

  func area() -> Double {
    return sideLength * sideLength
  }

  override func simpleDescription() -> String {
    return "A square with sides of length \(sideLength)."
  }

  var sideLength: Double
}

let shape = Square(sideLength: 5.2, name: "My square")
</code></pre></td>
<td><pre><code class="language-cpp">
class Square: public Shape {
public:
  Square(double sideLength, string name)
    : sideLength(sideLength),
      Shape(name) {
      numberOfSides = 4;
  }

  auto area() -> double {
    return sideLength * sideLength;
  }

  auto simpleDescription() -> string {
    return (ostringstream()<<"A square with sides of length "<<sideLength).str();
  }
private:
  double sideLength;
};

auto shape = Square(5.2, "My Square");
</code></pre></td>
</tr></tbody>
</table>

 In C++ there's a clear concept of *function overloading* vs *function overriding*. To get the same behavior as Swift we would have to mark the base class `simpleDescription()` as `virtual`, so that if we assign a `Square` instance to `Shape` it would dispatch the `Square.simpleDescription` at runtime. To elaborate this is what would happen with the current implementation:

 ```cpp
auto shape = make_shared<Shape>("My Shape");  // create new Shape
cout << shape->simpleDescription() << endl;   // prints: A shape with 0 sides.
shape.reset(new Square(5.2, "My Square"));    // release Shape and create new Square
cout << shape->simpleDescription() << endl;   // prints: A shape with 4 sides.
```

But after we mark the method as `virtual` and `override` we get the dynamic dispatching as we expect.

```cpp
class Shape {
  virtual auto simpleDescription() -> string {
    return (ostringstream()<<"A shape with "<<numberOfSides<<" sides.").str();
  }
  // ...
};

class Square: public Shape {
  auto simpleDescription() -> string override {
    return (ostringstream()<<"A square with sides of length "<<sideLength).str();
  }
  // ...
};

auto shape = make_shared<Shape>("My Shape");  // create new Shape
cout << shape->simpleDescription() << endl;   // prints: A shape with 0 sides.
shape.reset(new Square(5.2, "My Square"));    // release Shape and create new Square
cout << shape->simpleDescription() << endl;   // prints: A square with sides of length 5.2
```

## Reference counting

Swift has reference counting mechanism builtin. C++ also provides reference counting, but it isn't builtin. In order to get the same behavior as Swift, we need to make use of `shared_ptr` and `weak_ptr`. Here's how it looks in C++

```cpp
class MyString {
public:
  MyString(string str) : str(str) {
    cout << "created" << endl;
  }

  ~MyString() {
    cout << "destroyed" << endl;
  }

  string str;
};

auto s0 = make_shared<MyString>("hello!");
cout << "first strong ref: " << s0->str << " refCount:" << s0.use_count() << endl;

auto w0 = weak_ptr<MyString>(s0);
cout << "first weak ref: " << s0->str << " refCount:" << s0.use_count() << endl;

auto s1 = s0;
cout << "second strong ref: " << s1->str << " refCount:" << s0.use_count() << endl;
```

And this is the output

```
created
first strong ref: hello! refCount:1
first weak ref: hello! refCount:1
second strong ref: hello! refCount:2
destroyed
```

No surprises here, as you can see only a single instance is ever created and both the instance refer to the same thing with just the reference count that gets incremented. Also weak reference doesn't increment the reference count. And then finally when we're done the instance gets automatically deallocated. Just like in swift.

## Error handling

Error handling in C++ is a bit more involved than Swift. So if we want to see the C++ exception handling from the perspective of Swift error handling, it might look something like:

<table>
<thead><tr><th>Swift</th><th>C++</th></tr></thead>
<tbody><tr>
<td><pre><code class="language-swift">
func send(job: Int, toPrinter printerName: String) throws -> String {
  if printerName == "BadPrinterr" {
    throw PrinterError.noToner
  }
  return "Job sent"
}

do {
  let printerResponse = try send(job: 1040, toPrinter: "BadPrinter")
  print(printerResponse)
} catch let error {
  print(error.description)
}
</code></pre></td>
<td><pre><code class="language-cpp">
auto sendJobToPrinter(int job, string printerName) -> string {
  if (printerName == "BadPrinter") {
    throw PrinterError::noToner();
  }
  return "Job sent";
}

try {
  auto printerResponse = sendJobToPrinter(1040, "BadPrinter");
  cout << printerResponse << endl;
} catch (const exception & error) {
  cout << error.what() << endl;
}
</code></pre></td>
</tr></tbody>
</table>

Unlike Swift, C++ functions are assumed to always `throw`. If they're guaranteed to not throw at all then we can mark them as `noexcept`, in that case if any exception is thrown the app just crashes.

In Swift our `PrinterError` has to conform to `Error`, similarly in C++ our custom exception `PrinterError` has to subclass `exception`.

```swift
enum PrinterError: Error {
    case outOfPaper
    case noToner
    case onFire
}
```

Implementing the `PrinterError` type of `exception` is a bit more involved since C++ enums are not as awesome as Swift, but we can improvise.

```cpp
class PrinterError: public exception {
public:
  static auto outOfPaper() -> PrinterError {
    return PrinterError(0, "out of paper");
  }
  static auto noToner() -> PrinterError {
    return PrinterError(1, "no toner");
  }
  static auto onFire() -> PrinterError {
    return PrinterError(2, "on fire");
  }

  PrinterError(int errorCode, const std::string &message) noexcept
  : errorCode(errorCode), message(message)
  {}

  auto what() const noexcept -> const  char * {
    return message.c_str();
  }

  int errorCode;
  string message;
};
```

## Generics

If there's one thing where C++ beats Swift hands down, it's Generics. Here's an example how a generic `class` might look in C++

```cpp
template <typename Element>
class Stack {
public:

  auto push(Element e) {
    storage.push_back(e);
  }

  auto pop() -> optional<Element> {
    if (storage.empty()) {
      return nullopt;
    }
    auto lastEle = storage.back();
    storage.pop_back();
    return lastEle;
  }

private:
  vector<Element> storage;
};
```

When it comes to generics, C++ isn't as as verbose as Swift.

<table>
<thead><tr><th>Swift</th><th>C++</th></tr></thead>
<tbody><tr>
<td><pre><code class="language-swift">
func anyCommonElements<T: Sequence, U: Sequence>(_ lhs: T, _ rhs: U) -> Bool
  where T.Element: Equatable, T.Element == U.Element {
  for lhsItem in lhs {
    for rhsItem in rhs {
      if lhsItem == rhsItem {
        return true
      }
    }
  }
  return false
}

anyCommonElements([1, 2, 3], [3])
</code></pre></td>
<td><pre><code class="language-cpp">
template <typename T, typename U> 
auto anyCommonElements(T lhs, U rhs) -> bool {
  for (auto lhsItem : lhs) {
    for (auto rhsItem : rhs) {
      if (lhsItem == rhsItem) {
        return true;
      }
    }
  }
  return false;
}

anyCommonElements(vector&lt;int&gt;{1, 2, 3}, vector&lt;int&gt;{3});
</code></pre></td>
</tr></tbody>
</table>

Notice no type constraints to `Sequence` or `Equatable` since the C++ compiler automatically takes care of that. In case the type constraints fail, for instance `T.Element` does not have `==` implemeted, the compiler throws errors that are generally hard to decipher. If you like that level of control c++20 has a proposal for [constraints and concepts](https://en.cppreference.com/w/cpp/language/constraints) which is very close to where Swift is already at.