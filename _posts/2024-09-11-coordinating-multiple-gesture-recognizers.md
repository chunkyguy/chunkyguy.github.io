---
layout: post
title: Coordinating multiple gesture recognizers
date: 2024-09-11 21:53 +0200
categories: swift uikit ios animation
published: true
---

So how does one actually work with multiple gesture recognizers on same view?

![UIGestureRecognizer is now my best friend](https://i.imgflip.com/936fxw.jpg)

Let's recreate the **MoveMe** sample with `UIGestureRecognizer`. The idea is to have both `UILongPressGestureRecognizer` and `UIPanGestureRecognizer` play nicely with each other. With `UILongPressGestureRecognizer` responsible for detecting selection and `UIPanGestureRecognizer` responsible for dragging the selected squares.

### Setup

We have a `SquareView` that can react to changes such as selection and change in position. The main updates happen in `layoutSubviews`. `setNeedsLayout` reschedules a update at next draw cycle and `layoutIfNeeded` requests an update immediately.

```swift
struct SquareViewProps {
  var color = UIColor.blue
  var scale: CGFloat = 1.0
  var position: CGPoint
}

class SquareView: UIView {
  
  private var props: SquareViewProps {
    didSet { setNeedsLayout() }
  }
  
  override init(frame: CGRect) {
    props = SquareViewProps(position: CGPoint(x: frame.midX, y: frame.midY))
    super.init(frame: frame)
  }
  
  override func layoutSubviews() {
    super.layoutSubviews()
    backgroundColor = props.color
    center = props.position
    transform = CGAffineTransform(scaleX: props.scale, y: props.scale)
  }
  
  private func resetProps(_ props: SquareViewProps) {
    self.props = props
    UIView.animate(withDuration: 0.3, delay: 0, options: [.beginFromCurrentState]) {
      self.layoutIfNeeded()
    }
  }
}
```

And then we can host the `SquareView` in some parent `UIView` or `UIViewController`

```swift
class ViewController: UIViewController {
  
  private var squareVws: [SquareView] = []
  
  override func viewDidLoad() {
    super.viewDidLoad()
    
    squareVws = (0..<3).map { idx in
      SquareView(
        frame: CGRect(
          x: view.bounds.midX - 50,
          y: CGFloat.lerp(
            start: (view.bounds.minY + 100),
            end: (view.bounds.maxY - 200),
            factor: CGFloat(idx) / 2
          ),
          width: 100, height: 100
        )
      )
    }
    
    squareVws.forEach { view.addSubview($0) }
  }  
}
```

![setup]({{site.url}}/assets/coordinating-gestures/01-setup.png)

### Problem

Next we can add a `UIPanGestureRecognizer` on the `ViewController` and forward the gesture events to selected views.

```swift
enum GestureEvent {
  case began
  case changed(CGPoint)
  case ended
}

class SquareView: UIView {
    func handleGestureEvent(_ event: GestureEvent) {
    switch event {
    case .began:
      resetProps(SquareViewProps(
        color: .red,
        scale: 1.2,
        position: props.position
      ))
      
    case .changed(let translation):
      props.position = CGPoint.add(translation, props.position)
      
    case .ended:
      resetProps(SquareViewProps(position: props.position))
    }
  }

  // ...
}

class ViewController: UIViewController {
  
  private var squareVws: [SquareView] = []
  private var selectedVws: [SquareView] = []
  
  override func viewDidLoad() {
    // ...
    
    let dragGesture = UIPanGestureRecognizer(
      target: self,
      action: #selector(handleDrag)
    )
    
    view.addGestureRecognizer(dragGesture)
  }
  
  @objc func handleDrag(_ sender: UIPanGestureRecognizer) {
    switch sender.state {
    case .began:
      let pt = sender.location(in: view)
      selectedVws = squareVws.filter { $0.frame.contains(pt) }
      handleGestureEvent(.began)
      
    case .changed:
      let translation = sender.translation(in: view)
      handleGestureEvent(.changed(translation))
      sender.setTranslation(.zero, in: view)

    case .ended:
      handleGestureEvent(.ended)
      selectedVws = []
      
    default:
      break
    }
  }
  
  func handleGestureEvent(_ event: GestureEvent) {
    selectedVws.forEach { $0.handleGestureEvent(event) }
  }
}
```

So far so good but with this implementation we receive gesture events after the drag has started but we want to receive the `.began` as soon as the user touches the square. We can use the `UITapGestureRecognizer` but it only activates at touch up and not at touch down. So either we need to rollout our own gesture or 'hack' `UILongPressGestureRecognizer`

```swift
class ViewController: UIViewController {
  
  private var squareVws: [SquareView] = []
  private var selectedVws: [SquareView] = []
  
  override func viewDidLoad() {
    super.viewDidLoad()

    // ...
    
    let tapGesture = UILongPressGestureRecognizer(
      target: self,
      action: #selector(handleTap)
    )
    tapGesture.minimumPressDuration = 0.1
    
    let dragGesture = UIPanGestureRecognizer(
      target: self,
      action: #selector(handleDrag)
    )
    
    view.addGestureRecognizer(tapGesture)
    view.addGestureRecognizer(dragGesture)
  }
  
  @objc func handleTap(_ sender: UILongPressGestureRecognizer) {
      switch sender.state {
      case .began:
        let pt = sender.location(in: view)
        selectedVws = squareVws.filter { $0.frame.contains(pt) }
        handleGestureEvent(.began)

      case .ended:
        handleGestureEvent(.ended)
        selectedVws = []
        
      default:
        break
      }
  }
  
  @objc func handleDrag(_ sender: UIPanGestureRecognizer) {
    switch sender.state {
    case .changed:
      let translation = sender.translation(in: view)
      handleGestureEvent(.changed(translation))
      sender.setTranslation(.zero, in: view)

    default:
      break
    }
  }
  
  func handleGestureEvent(_ event: GestureEvent) {
    selectedVws.forEach { $0.handleGestureEvent(event) }
  }
}
```

But this poses another problem. On a `UIView` by default only one gesture recognizer is active at a time. So once the `UILongPressGestureRecognizer` is activated `UIPanGestureRecognizer` is ignored.

### Solution
Gesture recognizers have a defined order of precedence. So if multiple gesture recognizers are attached to a view, the winner is decided by the default rules. But we can override the rules by implementing the `UIGestureRecognizerDelegate`. For our case since each gesture recognizer is listening to different states we can have both the gestures active at the same time

```swift
class ViewController: UIViewController {
  
  // ...    
  
  
  override func viewDidLoad() {
    super.viewDidLoad()

    let tapGesture = UILongPressGestureRecognizer()
    let dragGesture = UIPanGestureRecognizer()

    // ...    
    
    tapGesture.delegate = self
    dragGesture.delegate = self
  }
}

extension ViewController: UIGestureRecognizerDelegate {
  public func gestureRecognizer(
    _ gestureRecognizer: UIGestureRecognizer,
    shouldRecognizeSimultaneouslyWith otherGestureRecognizer: UIGestureRecognizer
  ) -> Bool {
    return gestureRecognizer is UILongPressGestureRecognizer
    && otherGestureRecognizer is UIPanGestureRecognizer
  }
}
```

![setup]({{site.url}}/assets/coordinating-gestures/02-solution.gif)

The solution is available on [https://github.com/chunkyguy/MoveMe](https://github.com/chunkyguy/MoveMe/tree/main/uikit)

### References
- [Coordinating multiple gesture recognizers](https://developer.apple.com/documentation/uikit/touches_presses_and_gestures/coordinating_multiple_gesture_recognizers)
- [Allowing the simultaneous recognition of multiple gestures](https://developer.apple.com/documentation/uikit/touches_presses_and_gestures/coordinating_multiple_gesture_recognizers/allowing_the_simultaneous_recognition_of_multiple_gestures)
