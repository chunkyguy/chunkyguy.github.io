---
layout: post
title: "Policy based design in Swift"
date: 2019-10-25 10:26:19 +0200
categories: architecture ios
published: true
---

How about [Policy based design](https://en.wikipedia.org/wiki/Modern_C%2B%2B_Design#Policy-based_design) in Swift? 

It's quite an interesting pattern in C++. I first read about it in the **Modern C++ Design by Andrei Alexandrescu** years ago. The idea is to replace inheritance madness by composition. If we could wrap every reusable functionality in a `Policy` then we can compose objects from those policies. One might think this is nothing new, but that is where things get interesting. The composition happens at compile time! Which obviously requires the language to have some generic programming support.

In C++ this magic works beautifully since the language allows multiple inheritance. So as long as the policies are free of any ambiguities, one can have as many permutations of policies as they want. In swift though, it's a little challenge to implement as we do not have multiple inheritance, but we can workaround by having allocations per policy and can still achieve similar effect.

So, here's my implementation of the Policy pattern in Swift:

```swift
import Foundation

protocol OutputPolicy {
    associatedtype Message
    func print(message: Message)
}

protocol LanguagePolicy {
    associatedtype Message
    var message: Message { get }
}

struct Printer<O: OutputPolicy, L: LanguagePolicy> where O.Message == L.Message {
    let output: O
    let language: L

    func run() {
        output.print(message: language.message)
    }
}

class OutputPolicyToConsole: OutputPolicy {
    func print(message: String) {
        Swift.print(message)
    }
}

class LanguagePolicyEnglish: LanguagePolicy {
    let message = "Hello World!"
}

class LanguagePolicyEmoji: LanguagePolicy {
    let message: String = "👋 🌏❗️"
}

let printerEnglish = Printer(
    output: OutputPolicyToConsole(), 
    language: LanguagePolicyEnglish()
)
let printerEmoji = Printer(
    output: OutputPolicyToConsole(), 
    language: LanguagePolicyEmoji()
)

printerEnglish.run()
printerEmoji.run()
```

For reference, this is based on the sample example in the wiki article for C++:

```cpp
template <typename OutputPolicy, typename LanguagePolicy>
class Printer : private OutputPolicy, private LanguagePolicy
{
 public:
  void run() const {
    print(message());
  }

 private:
  using LanguagePolicy::message;
  using OutputPolicy::print;
};

class OutputPolicyWriteToCout
{
 protected:
  template <typename MessageType>
  void print(MessageType&& message) const {
    std::cout << message << std::endl;
  }
};

class LanguagePolicyEnglish
{
 protected:
  std::string message() const { return "Hello, World!"; }
};

class LanguagePolicyEmoji
{
 protected:
  std::string message() const { return "👋🌏!"; }
};

int main() {
    Printer<OutputPolicyWriteToCout, LanguagePolicyEnglish> printerEnglish;
    Printer<OutputPolicyWriteToCout, LanguagePolicyEmoji> printerEmoji;

    printerEnglish.run();
    printerEmoji.run();
}
```