---
layout: post
title: objc async selectors
date: 2024-12-07 15:21 +0100
categories: swift objc async
published: true
---

I very recently learned it the hard way that objc selectors can not be async.

In simple words, this will crash at runtime:

```swift
let button = UIButton(frame: buttonFrame)
view.addSubview(button)
button.setTitle("This is a button", for: .normal)
button.addTarget(
  self,
  action: #selector(handleTap), 
  for: .touchUpInside
)
  
@objc func handleTap() async {
  Task {
    debugPrint("handle Tap")
  }
}
```

No warnings, no errors, no messages on console, nothing. Just crash!

As it turns out all objc selector marked as async will crash. For example:

 ```swift
 perform(#selector(handleTap))
 ``` 

So what is going on here. Let's first print the selector generated by the compiler for `async` methods.

```swift
debugPrint(NSStringFromSelector(#selector(handleTap)))
// "handleTapWithCompletionHandler:"
```

As per swift objc interops guidelines, Swift maps all objc methods with suffix `"WithCompletionHandler"` as `async. So these two are equivalent:

```objc
- (void)stopRecordingWithCompletionHandler: void(^ _Nullable)(RPPreviewViewController * _Nullable, NSError * _Nullable)handler;
```

```swift
func stopRecording() async throws -> RPPreviewViewController
```

So our `func handleTap()` becomes `- (void) handleTapWithCompletionHandler:(CompletionHandler completionHandler)` in objc. 

The next question naturally is what is the type of `CompletionHandler`?

I tried to search online if others have also faced this problem. And yes there are quite a number of discussions on this topic. 
[This one](https://github.com/swiftlang/swift/issues/60084#issuecomment-1192174616) even has nice a workaround to the problem:

```swift
let completion: @convention(block) () -> Void = {
  debugPrint("completion handler")
}

perform(#selector(handleTap), with: completion)

// prints:
// completion handler
// handle Tap
```

`@convention` is a attribute provided by swift for calling conventions in Swift. And `@convention(block)` is the attribute to represent Objective-C blocks as Swift closures.

Now, with all this acquired knowledge can we fix our crash? We still can't use the `#selector(handleTap)` for target action pattern because that pattern is very well documented to support only following method signatures:

```swift
func doSomething()
func doSomething(sender: Any)
func doSomething(sender: Any, forEvent event: UIEvent)
```

Here the `sender` and `event` is autofilled by objc runtime while invoking the provided `selector`. And as you can see there is no place to provide arbitrary arguments. But for this entire mechanism to work we want the `sender` to be of type `@convention(block) () -> Void`.

So we can create our own custom `UIControl` subclass and hijack the target-action mechanism to update the sender. 
In other words, we need to override the `sendAction()` method and then call the `performSelector`.

```swift
class MyButton: UIButton {
  override func sendAction(_ action: Selector, to target: Any?, for event: UIEvent?) {
    let sender: @convention(block) () -> Void = {
      debugPrint("completion handler")
    }
    (target as? NSObject)?.perform(action, with: sender)
  }
}

class ViewController: UIViewController {
  
  override func viewDidLoad() {
    super.viewDidLoad()
    let buttonFrame = CGRect(
      x: 20,
      y: (view.bounds.height * 0.5) - 30,
      width: view.bounds.width - 40,
      height: 60
    )
    let button = MyButton(frame: buttonFrame)
    view.addSubview(button)
    button.setTitleColor(.black, for: .normal)
    button.setTitle("This is a button", for: .normal)
    button.addTarget(
      self,
      action: #selector(handleTap),
      for: .touchUpInside
    )
  }
  
  @objc func handleTap() async {
    Task {
      debugPrint("handle Tap")
    }
  }
}
```

And that is how you can have a button action as `async`.

### References
- [https://developer.apple.com/documentation/uikit/responding-to-control-based-events-using-target-action](https://developer.apple.com/documentation/uikit/responding-to-control-based-events-using-target-action)
- [https://developer.apple.com/documentation/uikit/uicontrol/sendaction](https://developer.apple.com/documentation/uikit/uicontrol/sendaction(_:to:for:))
- [https://reintech.io/blog/using-swifts-convention](https://reintech.io/blog/using-swifts-convention)
- [https://github.com/swiftlang/swift/issues/60084](https://github.com/swiftlang/swift/issues/60084)
- [https://github.com/swiftlang/swift-evolution/blob/main/proposals/0297-concurrency-objc.md](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0297-concurrency-objc.md)
- [https://docs.swift.org/swift-book/documentation/the-swift-programming-language/attributes/#convention](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/attributes/#convention)