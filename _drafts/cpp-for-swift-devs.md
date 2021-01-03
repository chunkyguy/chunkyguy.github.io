---
layout: post
title:  "C++ for Swift developers"
date:   2021-01-03 00:10:00 +0200
categories: swift cpp
published: true
---

C++ is very much like Swift, and when I say C++ I mean C++11 and above. Here's a minimalistic C++ code

```cpp
#include <iostream> // 1

using namespace std; // 2

int main() { // 3
  cout << "Hello world!"; // 4
  return 0; // 5
}
```

-------

```cpp
#include <iostream> // 1
```

This is equivalent to `import iostream` in Swift. Frameworks are usually bundled into a single file, so you need to add only 1 include statement per framework. But unlike Swift, we need to add include statements for every file we want to use from our own code.

```cpp
#include <someframework> // external framework
#include "path/to/file.h" // local file
```

-------

```cpp
using namespace std; // 2
```

Unlike Swift, C++ has a proper support for namespaces. So if two types in different frameworks have same name they can be resolved using their namespaces. This above example could also be written as:

```cpp
int main() {
  std::cout << "Hello world!";
  return 0;
}
```

-------

```cpp
int main() { // 3
  // ...
}
```

This is how one writes functions in C++. Equivalent Swift code would be:

```swift
func main() -> Int {
}
```

-------

```cpp
cout << "Hello world!"; // 4
```

Like Swift we can also override operators in C++. In this case the `iostream` library provides overrides for `operator <<` for many common types like for `string` that we're using above.

-------

```cpp
return 0; // 5
```

Since our function requires a return type `int` we have to end our function with a `return` statement.

-------

## Variables

Using variables in C++ is pretty close to Swift.

| **Swift**             | **C++**                       |
|-----------------------|-------------------------------|
| `var myVariable = 42` | `auto myVariable = 42;`       |
| `myVariable = 50`     | `myVariable = 50;`            |
| `let myConstant = 42` | `const auto myConstant = 42;` |

Just like in Swift types can be deduced by the compiler using `auto`, but if required we can always also provide the type information explicitly.

| **Swift**                   | **C++**                 |
|-----------------------------|-------------------------|
| `let myFloat: Float = 3.14` | `float myFloat = 3.14;` |

Type conversion happens implicitly in C++ and that might be a bit unexpected for many cases or might feel less verbose in some compared to Swift.

| **Swift**                                | **C++**                                       |
|------------------------------------------|-----------------------------------------------|
| `let label = "The width is "`            | `auto label = "The width is ";`               |
| `let width = 94`                         | `auto width = 94;`                            |
| `let widthLabel = label + String(width)` | `auto widthLabel = label + to_string(width);` |


## String

Coming to string interpolation, C++ isn't as good as Swift but nonetheless we can construct `string` with either explicit string conversion, like

| **Swift**                                                          | **C++**                                                                              |
|--------------------------------------------------------------------|--------------------------------------------------------------------------------------|
| `let apples = 3`                                                   | `auto apples = 3;`                                                                   |
| `let oranges = 5`                                                  | `auto oranges = 5;`                                                                  |
| `let appleSummary = "I have \(apples) apples."`                    | `auto appleSummary = "I have " + to_string(apples) + " apples.";`                    |
| `let fruitSummary = "I have \(apples + oranges) pieces of fruit."` | `auto fruitSummary = "I have " + to_string(apples + oranges) + " pieces of fruit.";` |

Or using `ostringstream`

```cpp
auto apples = 3;
ostringstream ss;
ss << "I have " << apples << " apples.";
auto str = ss.str();
```

which can further be reduced to:

```cpp
auto str = (ostringstream()<<"I have "<<apples<<" apples.").str();
```

| **Swift**                                                          | **C++**                                                                                            |
|--------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
|                                                                    | `ostringstream ss;`                                                                                |
| `let apples = 3`                                                   | `auto apples = 3;`                                                                                 |
| `let oranges = 5`                                                  | `auto oranges = 5;`                                                                                |
| `let appleSummary = "I have \(apples) apples."`                    | `auto appleSummary = (ostringstream()<< "I have "<<apples<<" apples.").str();`                     |
| `let fruitSummary = "I have \(apples + oranges) pieces of fruit."` | `auto fruitSummary = (ostringstream()<<"I have "<<(apples + oranges)<<" pieces of fruit.").str();` |

## Array

Swift `Array` equivalent in C++ is `vector`. Here's how you use it

| **Swift**                                           | **C++**                                                             |
|-----------------------------------------------------|---------------------------------------------------------------------|
| `var shoppingList = ["catfish", "water", "tulips"]` | `auto shoppingList = vector<string>{"catfish", "water", "tulips"};` |
| `shoppingList[1] = "bottle of water"`               | `shoppingList[1] = "bottle of water";`                              |
| `shoppingList.append("blue paint")`                 | `shoppingList.push_back("blue paint");`                             |

And this is how you iterate a `vector`

| **Swift**                    | **C++**                              |
|------------------------------|--------------------------------------|
| `for item in shoppingList {` | `for (auto & item : shoppingList) {` |
| `  print(item)`              | `  cout << item << endl;`            |
| `}`                          | `}`                                  |

## Dictionary

Swift `Dictionary` equivalent in C++ is `map`. Here's how you use it

| **Swift**                                   | **C++**                                      |
|---------------------------------------------|----------------------------------------------|
| `var occupations = [`                       | `auto occupations = map<string, string> {`   |
| `  "Malcolm": "Captain",`                   | `  {"Malcolm", "Captain"},`                  |
| `  "Kaylee": "Mechanic",`                   | `  {"Kaylee", "Mechanic"},`                  |
| `]`                                         | `};`                                         |
| `occupations["Jayne"] = "Public Relations"` | `occupations["Jayne"] = "Public Relations";` |

And this is how you iterate a `map`

| **Swift**                           | **C++**                                    |
|-------------------------------------|--------------------------------------------|
| `for (key, value) in occupations {` | `for (auto& [key, value]: occupations) {`  |
| `  print("\(key) : \(value)")`      | `  cout << key << " : " << value << endl;` |
| `}`                                 | `}`                                        |

## Optional

Swift developers love `Optional` types. Fortunately C++ also got `optional` types starting c++17. Although not as fancy as with Swift, but the idea is the same

```cpp
auto greeting(bool required) {
  return required ? optional<string>{"Hello"} : nullopt;
}

int main() {
  auto s = greeting(false).value_or("Hi");
  cout << s << endl; // Hi

  if (auto str = greeting(true)) {
    cout << *str << endl; // Hello
  }

  return 0;
}
```

## Function

C++ functions do not have argument labels, so the way Swift functions would look in C++ is not as good:

```swift
// swift
func greet(_ person: String, on day: String) -> String {
    return "Hello \(person), today is \(day)."
}
greet("John", on: "Wednesday")
```

```cpp
// cpp
string greetPersonOnDay(string person, string day) {
  ostringstream ss;
  ss << "Hello " << person << ", today is " << day << ".";
  return ss.str();
}
greetPersonOnDay("John", "Wednesday")
```

But the function can also be written to look more Swift like:

```cpp
auto greetPersonOnDay(string person, string day) -> string
```

## Closures

C++ has a good support for closures. Here's an example:

| **Swift**                                                             | **C++**                                                                         |
|-----------------------------------------------------------------------|---------------------------------------------------------------------------------|
| `func hasAnyMatches(list: [Int], condition: (Int) -> Bool) -> Bool {` | `auto hasAnyMatches(vector<int> list, function<bool(int)> condition) -> bool {` |
| `  for item in list {`                                                | `  for (auto item : list) {`                                                    |
| `    if condition(item) {`                                            | `    if (condition(item)) {`                                                    |
| `      return true`                                                   | `      return true;`                                                            |
| `    }`                                                               | `    }`                                                                         |
| `  }`                                                                 | `  }`                                                                           |
| `  return false`                                                      | `  return false;`                                                               |
| `}`                                                                   | `}`                                                                             |
|                                                                       |                                                                                 |
| `func lessThanTen(number: Int) -> Bool {`                             | `auto lessThanTen(int number) -> bool {`                                        |
| `  return number < 10`                                                | `  return number < 10;`                                                         |
| `}`                                                                   | `}`                                                                             |
|                                                                       |                                                                                 |
| `var numbers = [20, 19, 7, 12]`                                       | `auto numbers = vector<int> {20, 19, 7, 12};`                                   |
| `hasAnyMatches(list: numbers, condition: lessThanTen)`                | `hasAnyMatches(numbers, lessThanTen);`                                          |

That said C++ still doesn't has all the fancy functional operations we get with swift, but they're coming soon with C++20. For now we can use `transform` as an equivalent to Swift `map`

| **Swift**                               | **C++**                                                                                |
|-----------------------------------------|----------------------------------------------------------------------------------------|
| `numbers.map({ (number: Int) -> Int in` | `transform(numbers.begin(), numbers.end(), numbers.begin(), [](auto number) -> auto {` |
| `  let result = 3 * number`             | `  auto result = 3 * number;`                                                          |
| `  return result`                       | `  return result;`                                                                     |
| `})`                                    | `});`                                                                                  |

The C++ code here would actually mutate the `numbers`. If we want the truly Swift `map` equivalent we would have to create an empty `vector` and append data to it

```cpp
auto mappedNumbers = vector<int> {};
transform(numbers.begin(), numbers.end(), back_inserter(mappedNumbers), [](auto n) { return 3 * n; });
```

## Classes

Unlike Swift in C++ there isn't much difference between a `struct` and a `class` with the only difference being that a C++ struct has all members `public` by default whereas a `class` has all members as `private` by default. Whether the instance is mutable or not depends on how it is declared.


| **Swift**                                           | **C++**                                                                          |
|-----------------------------------------------------|----------------------------------------------------------------------------------|
| `class Shape {`                                     | `class Shape {`                                                                  |
|                                                     | `public:`                                                                        |
| `  func simpleDescription() -> String {`            | `  auto simpleDescription() -> string {`                                         |
| `    return "A shape with \(numberOfSides) sides."` | `    return (ostringstream()<<"A shape with "<<numberOfSides<<" sides.").str();` |
| `  }`                                               | `  }`                                                                            |
|                                                     |                                                                                  |
| `  var numberOfSides = 0`                           | `  int numberOfSides = 0;`                                                       |
| `}`                                                 | `};`                                                                             |
|                                                     |                                                                                  |
| `var shape = Shape()`                               | `auto shape = Shape();`                                                          |
| `shape.numberOfSides = 7`                           | `shape.numberOfSides = 7;`                                                       |
| `var shapeDescription = shape.simpleDescription()`  | `auto shapeDescription = shape.simpleDescription();`                             |

Initializer or constructor as they are called in C++ have a slightly different syntax

| **Swift**                                           | **C++**                                                                          |
|-----------------------------------------------------|----------------------------------------------------------------------------------|
| `class Shape {`                                     | `class Shape {`                                                                  |
|                                                     | `public:`                                                                        |
| `  init(name: String) {`                            | `  Shape(string name)`                                                           |
| `    self.name = name`                              | `    : name(name) {`                                                             |
| `  }`                                               | `  }`                                                                            |
|                                                     |                                                                                  |
| `  func simpleDescription() -> String {`            | `  auto simpleDescription() -> string {`                                         |
| `    return "A shape with \(numberOfSides) sides."` | `    return (ostringstream()<<"A shape with "<<numberOfSides<<" sides.").str();` |
| `  }`                                               | `  }`                                                                            |
|                                                     |                                                                                  |
| `  var numberOfSides = 0`                           | `  int numberOfSides = 0;`                                                       |
| `  var name: String`                                | `  string name;`                                                                 |
| `}`                                                 | `};`                                                                             |

In C++ subclasses need to be more explicit that Swift. Consider this

 | **Swift**                                                   | **C++**                                                                             |
 |-------------------------------------------------------------|-------------------------------------------------------------------------------------|
 | `class Square: Shape {`                                     | `class Square: public Shape {`                                                      |
 |                                                             | `public:`                                                                           |
 | `  init(sideLength: Double, name: String) {`                | `  Square(double sideLength, string name)`                                          |
 | `    self.sideLength = sideLength`                          | `    : sideLength(sideLength),`                                                     |
 | `    super.init(name: name)`                                | `      Shape(name) {`                                                               |
 | `    numberOfSides = 4`                                     | `      numberOfSides = 4;`                                                          |
 | `  }`                                                       | `  }`                                                                               |
 |                                                             |                                                                                     |
 | `  func area() -> Double {`                                 | `  auto area() -> double {`                                                         |
 | `    return sideLength * sideLength`                        | `    return sideLength * sideLength;`                                               |
 | `  }`                                                       | `  }`                                                                               |
 |                                                             |                                                                                     |
 | `  override func simpleDescription() -> String {`           | `  auto simpleDescription() -> string {`                                            |
 | `    return "A square with sides of length \(sideLength)."` | `    return (ostringstream()<<"A square with sides of length "<<sideLength).str();` |
 | `  }`                                                       | `  }`                                                                               |
 |                                                             | `private:`                                                                          |
 | `  var sideLength: Double`                                  | `  double sideLength;`                                                              |
 | `}`                                                         | `};`                                                                                |
 |                                                             |                                                                                     |
 | `let test = Square(sideLength: 5.2, name: "My square")`     | `auto shape = Square(5.2, "My Square");`                                            |

 Unlike Swift in C++ there's concept of *function overloading* vs *function overriding*. To get the same behavior as Swift we would have to mark the base class `simpleDescription()` as virtual, so that if we assign a `Square` instance to `Shape` it would dispatch the `Square.simpleDescription` at runtime. To elaborate this is what would happen with the current implementation:

 ```cpp
auto shape = make_unique<Shape>("My Shape");
cout << shape->simpleDescription() << endl; // A shape with 0 sides.
shape.reset(new Square(5.2, "My Square"));
cout << shape->simpleDescription() << endl; // A shape with 4 sides.
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

auto shape = make_shared<Shape>("My Shape");
cout << shape->simpleDescription() << endl; // A shape with 0 sides.
shape.reset(new Square(5.2, "My Square"));
cout << shape->simpleDescription() << endl; // A square with sides of length 5.2
```

## Reference counting

Swift has reference counting mechanism builtin. C++ also provides reference counting, but it isn't builtin. In order to get the same behavior as Swift, we need to make of `shared_ptr` and `weak_ptr`. Here's how it looks in C++

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

Error handling in C++ is a bit more complicated than Swift. So if we want to see the C++ exception handling from the perspective of Swift error handling, it might look something like:

| **Swift**                                                               | **C++**                                                          |
|-------------------------------------------------------------------------|------------------------------------------------------------------|
| `func send(job: Int, toPrinter printerName: String) throws -> String {` | `auto sendJobToPrinter(int job, string printerName) -> string {` |
| `  if printerName == "BadPrinterr" {`                                   | `  if (printerName == "BadPrinter") {`                           |
| `    throw PrinterError.noToner`                                        | `    throw PrinterError::noToner();`                             |
| `  }`                                                                   | `  }`                                                            |
| `  return "Job sent"`                                                   | `  return "Job sent";`                                           |
| `}`                                                                     | `}`                                                              |
|                                                                         |                                                                  |
| `do {`                                                                  | `try {`                                                          |
| `  let printerResponse = try send(job: 1040, toPrinter: "BadPrinter")`  | `  auto printerResponse = sendJobToPrinter(1040, "BadPrinter");` |
| `  print(printerResponse)`                                              | `  cout << printerResponse << endl;`                             |
| `} catch {`                                                             | `} catch (const exception &e) {`                                 |
| `  print(error)`                                                        | `  cout << e.what() << endl;`                                    |
| `}`                                                                     | `}`                                                              |

Unlike Swift, C++ functions are assumed to always `throw`. If they're guaranteed to not throw at all then we can mark them as `noexcept`, in that case if an exception is thrown the app just crashes.

Just as in Swift our `PrinterError` has to conform to `Error`, in C++ our custom exception has to subclass `exception` class.

```swift
enum PrinterError: Error {
    case outOfPaper
    case noToner
    case onFire
}
```

Implementing the `PrinterError` type of `exception` is a bit more involved since C++ enums are not as awesome as Swift, but we can to improvise.

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

One good thing about generics in C++ is that it isn't as verbose as Swift. So this example in Swift

```swift
func anyCommonElements<T: Sequence, U: Sequence>(_ lhs: T, _ rhs: U) -> Bool
    where T.Element: Equatable, T.Element == U.Element
{
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
```

can be simply written in C++ as

```cpp
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
```

Notice no type constraints to `Sequence` or `Equatable`, although if you like that level of control c++20 has a proposal for [constraints and concepts](https://en.cppreference.com/w/cpp/language/constraints) which is very close to Swift.