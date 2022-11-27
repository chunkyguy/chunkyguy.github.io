---
layout: post
title:  "Introducing Simple Layout Engine"
date:   2022-11-27 18:00:00 +0200
categories: swift layout
published: true
---

*Note: This is a Swift rewrite of the [original article](https://whackylabs.com/objc/ui/2020/09/15/simple-manual-layout/) published in 2020*

I love [Auto Layout](https://developer.apple.com/library/archive/documentation/UserExperience/Conceptual/AutolayoutPG/index.html). It helps a lot when designing complex UI. But there are times when the UI is very simple and Auto Layout might feel a bit overkill, while other times the UI might be a bit too complex and Auto Layout actually starts affecting the app performance. Before auto layout there was another technique to creating UI, it's called *Springs and Struts* (also known as *Manual Layout* to be in contrast with Auto Layout). I like Manual Layout a lot as well for its simplicity. Like with every other tool, there are trade-offs when selecting the best tool for the job, and it also applies when selecting Auto Layout vs Manual Layout.

The good thing is that, the Auto Layout has not been designed as an alternative to Manual Layout, rather more like an supplement. Where instead of us having to calculate the `frame`, we start with a `CGRect.zero` and let the Auto Layout fill in the `frame` value later. Most of the time it's wonderful and doesn't impact our flow. Other times we might have to wait for the layout pass run to read back the calculated `frame` values

```swift
// let Auto layout calculate the frame values
DispatchQueue.main.async {
  // start using the frame values for something else.
}
```

I often wish if the Auto Layout were not that tightly coupled with `UIKit`. In a sense I wish I could just run Auto Layout without a UIKit draw layout pass. This inspired me to take another take of building UIs without Auto Layout with something I’d like to call as *Simple Manual Layout*.

## Inspiration

The inspiration is from how `UIBarButtonItem` works with `UIToolbar` or `UINavigationBar`. If we wanted to build a UI like

![img]({{ site.url }}/assets/simple-manual-layout/toolbar.png)

We would create a `UIToolBar` and add a bunch of `UIBarButtonItem`
```swift
  let toolbar = UIToolbar(frame: toolbarFrame)
  let playButton = UIBarButtonItem(systemItem: .play)
  let pauseButton = UIBarButtonItem(systemItem: .pause)
  let rewindButton = UIBarButtonItem(systemItem: .rewind)
  let forwardButton = UIBarButtonItem(systemItem: .fastForward)
  let spaceButton = UIBarButtonItem(systemItem: .flexibleSpace)
  toolbar.items = [
    spaceButton,
    rewindButton, spaceButton,
    playButton, spaceButton,
    pauseButton, spaceButton,
    forwardButton, spaceButton,
  ]
```

The interesting element here is `.flexibleSpace`. Which is [documented as](https://developer.apple.com/documentation/uikit/uibarbuttonsystemitem/uibarbuttonsystemitemflexiblespace) _"Blank space to add between other items. The space is distributed equally between the other items."_. Similarly, there's another one called `.fixedSpace` which is [documented as](https://developer.apple.com/documentation/uikit/uibarbuttonsystemitem/uibarbuttonsystemitemfixedspace) _"Blank space to add between other items. Only the width property is used when this value is set."_.

I think this approach could be used to build a layout engine which is very simple in terms of mental model but can be used to build as sophisticated layouts as we'd want.

### Simple Layout Engine

With that design in mind we can build out layout engine. If there is a class `Item` which is a placeholder for a `UIView` and another class `Layout` that takes in one or more of these `Item` and immediately calculates the `frame` of the every `Item`. Then we can use the calculated `frame` value when constructing our `UIView` objects.

So to create a full screen subview we should be able to create as:

![img]({{ site.url }}/assets/simple-manual-layout/testFullScreenLayout.png)

```swift
let layout = Layout(parentFrame: frame, direction: .column)
let mainItem = try layout.add(item: .flexible)
let redView = SLECreateView(try mainItem.frame(), .red)
addSubview(redView)
```

```swift
private func SLECreateView(_ frame: CGRect, _ color: UIColor) -> UIView {
  let view = UIView(frame: frame)
  view.backgroundColor = color
  return view
}
```

And a 2 subview layout, where the top is flexible and bottom is fixed

![img]({{ site.url }}/assets/simple-manual-layout/testBottomFixLayout.png)

```swift
let layout = Layout(parentFrame: frame, direction: .column)
try layout.add(item: .flexible)
try layout.add(item: .height(200))

let topFrame = try layout.frame(at: 0)
let bottomFrame = try layout.frame(at: 1)

addSubview(SLECreateView(topFrame, .red))
addSubview(SLECreateView(bottomFrame, .blue))
```

A more interesting layout would be where we have a column that contains a row.

![img]({{ site.url }}/assets/simple-manual-layout/testInnerViewLayout.png)

```swift
let mainLayout = Layout(parentFrame: frame, direction: .column)
try mainLayout.add(items: [.flexible, .height(44), .height(200)])

let headerFrame = try mainLayout.frame(at: 0)
let toolbarFrame = try mainLayout.frame(at: 1)
let footerFrame = try mainLayout.frame(at: 2)

addSubview(SLECreateView(headerFrame, .red))
addSubview(SLECreateView(toolbarFrame, .blue))
addSubview(SLECreateView(footerFrame, .yellow))

let contentLayout = Layout(parentFrame: footerFrame, direction: .row)
try contentLayout.add(items: [.flexible, .flexible])
let content1Frame = try contentLayout.frame(at: 0)
let content2Frame = try contentLayout.frame(at: 1)

addSubview(SLECreateView(content1Frame, .cyan))
addSubview(SLECreateView(content2Frame, .magenta))
```

## Implementation details

The implementation of this layout engine turns out to be not as sophisticated. If we provide a `Item` which can have some properties fixed and others flexible. 

```swift
public class Item {
    // no values fixed
    public static var flexible: Item { get }
    // partially fixed
    public static func width(_ value: CGFloat) -> Item
    public static func height(_ value: CGFloat) -> Item
    // all fixed
    public static func size(_ value: CGSize) -> Item
    // ...
}
```

So we can start with storing values as `optional` with flexible data as `nil`

```swift
public class Item {
  // ...

  public func frame() throws -> CGRect {
    return try rect.frame()
  }

  internal let originalWidth: CGFloat?
  internal let originalHeight: CGFloat?
  private let rect = Rect()

  private init(width: CGFloat?, height: CGFloat?) {
    originalWidth = width
    originalHeight = height
  }

  // called by layout engine
  func updateSize(value: CGFloat, 
                  in direction: Direction, 
                  parentSize: CGSize) { /* update rect */ }

  func updateOrigin(itemOrigin: CGPoint,
                    in direction: Direction, 
                    alignment: Alignment, 
                    parentFrame: CGRect) -> CGPoint { /* update rect */ }
}
```

And we can have the internal `Rect` as a bridge object which is read from `Item` and written by the `Layout`

```swift
private class Rect {
  internal private(set) var width: CGFloat?
  internal private(set) var height: CGFloat?
  private var x: CGFloat?
  private var y: CGFloat?

  // read back by Item
  func frame() throws -> CGRect {
    guard let originX = x, let originY = y, let width = width, let height = height else {
      throw LayoutError.itemIncomplete
    }
    return CGRect(x: originX, y: originY, width: width, height: height)
  }

  // set by layout engine
  func set(origin: CGPoint) {
    x = origin.x
    y = origin.y
  }

  // set by layout engine
  func set(size: CGSize) {
    width = size.width
    height = size.height
 }
}
```

Next, within `Layout` we have a mutable array that contains `Item`. And whenever a new item is added we recalculate the frames per item.

```swift
extension Layout {
  public func add(item: Item) throws {
    items.append(item)
    try updateFrames()
  }
}
```

If we calculate only for one direction, say vertical. The `updateFrames` might look something like:

```swift
private extension Layout {
  func updateFrames() throws {

    // calculate total flex height
    var totalFlexSpace = parentFrame.height
    var flexItems = 0
    for item in items {
      if let space = item.originalHeight {
        totalFlexSpace -= space
      } else {
        flexItems += 1
      }
    }

    // calculate height per flex item
    let itemSpace = totalFlexSpace / CGFloat(max(flexItems, 1))
    guard itemSpace >= 0 else {
      throw LayoutError.outOfSpace
    }

    // update final frames per item
    var itemOrigin = parentFrame.origin
    for item in items {
      item.updateSize(value: itemSpace,
                      in: .column,
                      parentSize: parentFrame.size)
      itemOrigin = item.updateOrigin(itemOrigin: itemOrigin,
                                     in: .column,
                                     alignment: alignment,
                                     parentFrame: parentFrame)
    }
  }
}
```

And similar calculations for width.

And now it doesn’t seem hard to imagine to support alignment for sub views (currently they are all set `0.0` or all aligned to `.leading`) with something like:

```swift
public enum Alignment {
  case leading
  case center
  case trailing
}
```

The `Alignment` value provides an offset while calculating the origin

```swift
private extension Alignment {
  func align(parent: CGFloat, item: CGFloat) -> CGFloat {
    switch self {
    case .leading: return 0
    case .trailing: return (parent - item)
    case .center: return (parent - item) / 2.0
    }
  }
}
```

So for example, vertically this would be

```swift
let offset = alignment.align(parent: parentFrame.height, item: rect.height)
y = parentFrame.origin.y + offset
```

## References

1. [Simple Layout Engine](https://github.com/chunkyguy/SimpleLayoutEngine)
2. There’s also an [Objective-C implementation](https://github.com/chunkyguy/SimpleLayoutEngine-objc) which I think has a much simpler implementation
3. And finally the [original article](https://whackylabs.com/objc/ui/2020/09/15/simple-manual-layout/)