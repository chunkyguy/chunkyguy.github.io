---
layout: post
title: "Policy based design in Swift"
date: 2019-10-23 11:26:19
categories: architecture ios
published: false
---

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

let printerEnglish = Printer(output: OutputPolicyToConsole(), language: LanguagePolicyEnglish())
let printerEmoji = Printer(output: OutputPolicyToConsole(), language: LanguagePolicyEmoji())

printerEnglish.run()
printerEmoji.run()
```
