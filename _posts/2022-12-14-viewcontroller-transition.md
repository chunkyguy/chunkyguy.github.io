---
layout: post
title:  "Simple UIViewController Transitions"
date:   2022-12-14 18:00:00 +0200
categories: uikit swift ios animation
published: true
---
When it comes to `UIViewController` transitions there are many articles and videos out there that talk more in depth about the sophisticated solutions provided by UIKit and that is probably for the good reason that those UIKit APIs are very complicated to understand. But UIKit also provides a simpler API for simpler needs. And I think nobody talks enough about it either because it is so simple that everyone assumes that everyone knows how it works or that it "just" works somehow that nobody cares about the details of how it's intended to be used.

What API am I talking about? I'm talking about [this one liner](https://developer.apple.com/documentation/uikit/uiviewcontroller/1621363-transition):

```swift
func transition(
    from fromViewController: UIViewController,
    to toViewController: UIViewController,
    duration: TimeInterval,
    options: UIView.AnimationOptions = [],
    animations: (() -> Void)?,
    completion: ((Bool) -> Void)? = nil
)
```

If used incorrectly it might either just emit a warning on the console:
```
Unbalanced calls to begin/end appearance transitions for <MyViewController>
```

Or throw an exception at runtime and crash your app:
```
Terminating app due to uncaught exception 'NSInvalidArgumentException', 
reason: 'Children view controllers <ViewController> and <ViewController>
must have a common parent view controller when calling 
-[UIViewController transitionFromViewController:toViewController:duration:options:animations:completion:]'
```

## Let's build a demo!
What we want to make is a simple container view controller that has a single child view controller that can be replaced with some basic transition. Let's begin.

First let's see how the `ContainerViewController` should be used. So if we have a so called `ContainerViewController` then we should be able to `init` it and assign the content to it. 

For example, we could use a `Timer` to periodically cycle through a bunch of children.

```swift
let vwCtrls = (0..<5).map { SomeViewController(desc: "Child \($0)") }
let containerVwCtrl = ContainerViewController(contentViewController: vwCtrls[selectedIndex])
let timer = Timer.scheduledTimer(
    timeInterval: 3,
    target: self, selector: #selector(displayNext),
    userInfo: nil, repeats: true
)

@objc func displayNext() {
    selectedIndex = (selectedIndex + 1) % vwCtrls.count
    containerVwCtrl.contentViewController = vwCtrls[selectedIndex]
}
```

Or call the animated variant, one we are more interested in
```swift
@objc func displayNext() {
    selectedIndex = (selectedIndex + 1) % vwCtrls.count
    containerVwCtrl.set(contentViewController: vwCtrls[selectedIndex], animationDuration: 1.0)
}
```

The [docs](https://developer.apple.com/documentation/uikit/view_controllers/creating_a_custom_container_view_controller) on creating our `ContainerViewController` are pretty clear. Comes even with a snippet, we can simply copy-paste them:

```swift
class ContainerViewController: UIViewController {
      private func add(viewController: UIViewController) {
        addChild(viewController)
        view.addSubview(viewController.view)
        view.addConstraints(to: viewController.view, insets: contentEdgeInsets)
        viewController.didMove(toParent: self)
    }

    private func remove(viewController: UIViewController) {
        viewController.willMove(toParent: nil)
        viewController.view.removeFromSuperview()
        viewController.removeFromParent()
    }
}
```

Now since we want our container to always start with a child what better place than to ask than in the `init`

```swift
class ContainerViewController: UIViewController {
    private var childVwCtrl: UIViewController

    init(contentViewController: UIViewController) {
        childVwCtrl = contentViewController
        super.init(nibName: nil, bundle: nil)
        add(viewController: contentViewController)
    }
  // ...
}
```

![Some text]({{ site.url }}/assets/viewcontroller-transition/01.png)


So far so good. Next we want to expose methods to be able to reset the child

```swift
class ContainerViewController: UIViewController {
      var contentViewController: UIViewController {
        get { childVwCtrl }
        set { set(contentViewController: newValue, animationDuration: nil) }
    }

    func set(contentViewController toVwCtrl: UIViewController, animationDuration: Double?) {
      // TODO: stuff goes here ...
      childVwCtrl = toVwCtrl
    }
}
```

Since we want to be able to change the child with or without animation. If the `animationDuration` is provided we assume the change is animated otherwise not. Filling the non-animated part is easy, we just remove the last child and add the new child

```swift
class ContainerViewController: UIViewController {
    func set(contentViewController toVwCtrl: UIViewController, animationDuration: Double?) {
        let fromVwCtrl = childVwCtrl

        guard let duration = animationDuration else {
            remove(viewController: fromVwCtrl)
            add(viewController: toVwCtrl)
            childVwCtrl = toVwCtrl
            return
        }
        // ...
    }
}
```

![Some text]({{ site.url }}/assets/viewcontroller-transition/02.gif)

Coming to animated transition, we could use the API simply as:

```swift
class ContainerViewController: UIViewController {
    func set(contentViewController toVwCtrl: UIViewController, animationDuration: Double?) {
        let fromVwCtrl = childVwCtrl

        guard let duration = animationDuration else {
            remove(viewController: fromVwCtrl)
            add(viewController: toVwCtrl)
            childVwCtrl = toVwCtrl
            return
        }

        add(viewController: toVwCtrl)
        fromVwCtrl.view.alpha = 1.0
        toVwCtrl.view.alpha = 0.0
        transition(
            from: fromVwCtrl,
            to: toVwCtrl,
            duration: duration,
            options: [.curveEaseOut],
            animations: {
                fromVwCtrl.view.alpha = 0.0
                toVwCtrl.view.alpha = 1.0
            },
            completion: { _ in
                self.remove(viewController: fromVwCtrl)
                self.childVwCtrl = toVwCtrl
            }
        )
    }        
}
```

But this doesn't work. When run it emits a warning to the console.

```
Unbalanced calls to begin/end appearance transitions for Child 1.
```

 Let's see what the [docs](https://developer.apple.com/documentation/uikit/uiviewcontroller/1621363-transition) have to say about it.

> This method adds the second view controller's view to the view hierarchy and then performs the animations defined in your animations block. After the animation completes, it removes the first view controller's view from the view hierarchy.

Hmm, doesn't seems to add much. Let's also read the [docs](https://developer.apple.com/documentation/appkit/nsviewcontroller/1434407-transition) for `NSViewController` because why not

> This method adds the view in the toViewController view controller to the superview of the view in the fromViewController view controller. Likewise, this method removes the fromViewController view from the parent view controller’s view hierarchy at the appropriate time. It is important to allow this method to add and remove these views.

 Okay turns out this doc gives a bit more implementation detail. One interesting extra fact we learn here is that the view add and remove operations are called internally *at the appropriate time* (wonder when that might be). But now at least we know that we just need to add and remove the view controllers and let the API handle the add-remove of the views.

```swift
addChild(toVwCtrl)
toVwCtrl.didMove(toParent: self)

fromVwCtrl.view.alpha = 1.0
toVwCtrl.view.alpha = 0.0
transition(
    from: fromVwCtrl,
    to: toVwCtrl,
    duration: duration,
    options: [.curveEaseOut],
    animations: {
        fromVwCtrl.view.alpha = 0.0
        toVwCtrl.view.alpha = 1.0
    },
    completion: { _ in
        fromVwCtrl.willMove(toParent: nil)
        fromVwCtrl.removeFromParent()
        self.childVwCtrl = toVwCtrl
    }
)
```

So our warning is gone, but now the constraints are gone too! So the next child becomes full screen, or more specifically the size of the `ContainerViewController`

![Some text]({{ site.url }}/assets/viewcontroller-transition/03.gif)

We can try adding the constraints back in the animation block.

```swift
addChild(toVwCtrl)
toVwCtrl.didMove(toParent: self)

fromVwCtrl.view.alpha = 1.0
toVwCtrl.view.alpha = 0.0
transition(
    from: fromVwCtrl,
    to: toVwCtrl,
    duration: duration,
    options: [.curveEaseOut],
    animations: {
        self.view.addConstraints( to: toVwCtrl.view, insets: self.contentEdgeInsets)
        fromVwCtrl.view.alpha = 0.0
        toVwCtrl.view.alpha = 1.0
    },
    completion: { _ in
        fromVwCtrl.willMove(toParent: nil)
        fromVwCtrl.removeFromParent()
        self.childVwCtrl = toVwCtrl
    }
)
```
This seems to work as expected. 

![Some text]({{ site.url }}/assets/viewcontroller-transition/04.gif)

But since we don't have control over adding the view, we can't be sure if this would work all the time. Let's read up the more docs to see if we can find any more information.

### [**addChild(_ childController: UIViewController)**](https://developer.apple.com/documentation/uikit/uiviewcontroller/1621394-addchild)
> This method creates a parent-child relationship between the current view controller and the object in the `childController` parameter. This relationship is necessary when embedding the child view controller’s view into the current view controller’s content. If the new child view controller is already the child of a container view controller, it is removed from that container before being added.

### [**removeFromParent()**](https://developer.apple.com/documentation/uikit/uiviewcontroller/1621425-removefromparent)
> This method is only intended to be called by an implementation of a custom container view controller. If you override this method, you must call super in your implementation.

### [**willMove(toParent parent: UIViewController?)**](https://developer.apple.com/documentation/uikit/uiviewcontroller/1621381-willmove)
> If you are implementing your own container view controller, it must call the `willMove(toParent:)` method of the child view controller before calling the `removeFromParent()` method, passing in a parent value of nil.

### [**didMove(toParent parent: UIViewController?)**](https://developer.apple.com/documentation/uikit/uiviewcontroller/1621405-didmove)
> If you are implementing your own container view controller, it must call the `didMove(toParent:)` method of the child view controller after the transition to the new controller is complete or, if there is no transition, immediately after calling the `addChild(_:)` method.

It seems the only new thing here is that `didMove(toParent:)` has to be called after the transition. So the correct implementation should be following, although I don't find any visual difference

```swift
addChild(toVwCtrl)
fromVwCtrl.willMove(toParent: nil)
fromVwCtrl.view.alpha = 1.0
toVwCtrl.view.alpha = 0.0
transition(
    from: fromVwCtrl,
    to: toVwCtrl,
    duration: duration,
    options: [.curveEaseOut],
    animations: {
        self.view.addConstraints( to: toVwCtrl.view, insets: self.contentEdgeInsets)
        fromVwCtrl.view.alpha = 0.0
        toVwCtrl.view.alpha = 1.0
    },
    completion: { _ in
        toVwCtrl.didMove(toParent: self)
        fromVwCtrl.removeFromParent()
        self.childVwCtrl = toVwCtrl
    }
)
```

## Let's get rid of AutoLayout
This API is pretty nice for simpler transitions and maybe excellent if you don't have to bother with auto layout. Personally I find animating constraints always not so much fun.

For example making a slide in/out transition is pretty easy to build using just `frame`

```swift
func set(contentViewController toVwCtrl: UIViewController, animationDuration: Double?) {
    let fromVwCtrl = childVwCtrl

    guard let duration = animationDuration else {
        remove(viewController: fromVwCtrl)
        add(viewController: toVwCtrl)
        childVwCtrl = toVwCtrl
        return
    }

    addChild(toVwCtrl)
    toVwCtrl.view.frame = contentFrame
    fromVwCtrl.willMove(toParent: nil)
    beginAnimation(fromView: fromVwCtrl.view, toView: toVwCtrl.view)
    transition(
        from: fromVwCtrl,
        to: toVwCtrl,
        duration: duration,
        options: [.curveEaseOut],
        animations: {
            self.endAnimation(
                fromView: fromVwCtrl.view,
                toView: toVwCtrl.view
            )
        },
        completion: { _ in
            toVwCtrl.didMove(toParent: self)
            fromVwCtrl.removeFromParent()
            self.childVwCtrl = toVwCtrl
        }
    )
}

private func beginAnimation(fromView: UIView, toView: UIView) {
    fromView.transform = CGAffineTransform.identity
    toView.transform = CGAffineTransform(translationX: 500, y: 0)
}

private func endAnimation(fromView: UIView, toView: UIView) {
    fromView.transform = CGAffineTransform(translationX: -500, y: 0)
    toView.transform = CGAffineTransform.identity
}
```
![Some text]({{ site.url }}/assets/viewcontroller-transition/05.gif)

Or even go crazy with some mind blowing 3D transformations

```swift
    private func transform(angle: Double) -> CATransform3D {
        let angleRads = angle * .pi / 180.0
        var transform = CATransform3DIdentity
        transform.m34 = 1.0 / -100
        transform = CATransform3DRotate(transform, angleRads, 1, 0, 0)
        transform = CATransform3DTranslate(transform, 0, 0, -10)
        return transform
    }

    private func beginAnimation(fromView: UIView, toView: UIView) {
        fromView.layer.transform = CATransform3DIdentity
        toView.layer.transform = transform(angle: -90)
    }

    private func endAnimation(fromView: UIView, toView: UIView) {
        fromView.layer.transform = transform(angle: 90)
        toView.layer.transform = CATransform3DIdentity
    }
```

![Some text]({{ site.url }}/assets/viewcontroller-transition/06.gif)

If this seems interesting to you [I wrote about 3D transformation]((https://whackylabs.com/uikit/ios/2014/10/29/add-some-perspective-to-your-uiviews/)) in more details some time back, go check it out.

## Source

The code for this experiment is available at: [https://gist.github.com/chunkyguy/39a8e0c2151e0b13955141544627ec46](https://gist.github.com/chunkyguy/39a8e0c2151e0b13955141544627ec46)