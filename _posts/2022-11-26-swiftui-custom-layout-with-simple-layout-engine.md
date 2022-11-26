---
layout: post
title:  "SwiftUI custom layout with Simple Layout Engine"
date:   2022-11-26 14:39:00 +0200
categories: swiftui ios
published: true
---

The maths required for SwiftUI custom layout reminds me of the days before AutoLayout and the constraints based system. The good thing is that [Simple Layout Engine](https://whackylabs.com/objc/ui/2020/09/15/simple-manual-layout/) already provides a nice system to handle all the maths involved. To demonstrate I would use build the subset of the demo app from the WWDC session on this topic: [Compose custom layouts with SwiftUI](https://developer.apple.com/videos/play/wwdc2022/10056/).

# Problem

The idea is to have a container view similar to `HStack` where every child has the same width but with the exception that the width should be that of the maximum a child has. This is how `HStack` places the children by default

```swift
HStack {
    WLText("hi")
    WLText("!")
    WLText("beautiful")
    WLText("world")
}
```

![hstack]({{ site.url }}/assets/swiftui-custom-layout-with-simple-layout-engine/01.png)

Notice, all the children are have equal width but that isn't the width of the max child but rather *width = total width / number of children*

What we actually want is to have something that looks like *width = max(children.width)*, which would have all children having the width equal to whatever is the width of the *beautiful* text

```swift
BalancedHStack {
    WLText("hi")
    WLText("!")
    WLText("beautiful")
    WLText("world")
}
```

![hstack]({{ site.url }}/assets/swiftui-custom-layout-with-simple-layout-engine/02.png)

# SwiftUI Layout System

`SwiftUI` provides a way to plug into the layout system to provide all the custom maths. For our case we can create a `BalancedHStack` that conforms to `Layout` protocol. The `Layout` protocol requires two methods:

1. `sizeThatFits`: To provide the total `CGSize` of the container to the system
2. `placeSubviews`: To update the positions of the children within the provided bounds

```swift
struct BalancedHStack: Layout {
  func sizeThatFits(proposal: ProposedViewSize,
                    subviews: Subviews,
                    cache: inout ()) -> CGSize {
    fatalError()
    // TODO
  }

  func placeSubviews(in bounds: CGRect,
                     proposal: ProposedViewSize,
                     subviews: Subviews,
                     cache: inout ()) {
    // TODO
  }
}
```

Other than that we also get an optional helper `makeCache` method to perform and store any layout math in a custom `struct`. We can use that to find the child with largest size and also store the distances between adjacent children (the gray spaces in the image above)

```swift
struct BalancedHStack: Layout {

  struct CacheData {
    let childSize: CGSize
    let distances: [CGFloat]
  }

  func makeCache(subviews: Subviews) -> CacheData {
    let subviewSizes = subviews.map { $0.sizeThatFits(.unspecified) }
    let width = subviewSizes.map { $0.width }.max() ?? 0
    let height = subviewSizes.map { $0.height }.max() ?? 0
    let distances: [CGFloat] = (0..<subviews.count).map { idx in
      guard idx < subviews.count - 1 else { return 0 }
      return subviews[idx].spacing.distance(to: subviews[idx + 1].spacing, along: .horizontal)
    }
    return CacheData(
      childSize: CGSize(width: width, height: height),
      distances: distances
    )
  }

  // ...
}
```


Calculating the total size of the container is easy. Every child is going to get the same width plus the total distance

```swift
  func sizeThatFits(proposal: ProposedViewSize,
                    subviews: Subviews,
                    cache: inout CacheData) -> CGSize {
    let totalDistance = cache.distances.reduce(0, +)
    return CGSize(
      width: cache.childSize.width * CGFloat(subviews.count) + totalDistance,
      height: cache.childSize.height
    )
  }
```

Next for layout, we can use the *Simple Layout Engine* to calculate the frames

```swift
  func placeSubviews(in bounds: CGRect,
                     proposal: ProposedViewSize,
                     subviews: Subviews,
                     cache: inout CacheData) {
    let layout = SLELayout(parentFrame: bounds, direction: .row, alignment: .center)
    do {
      var items: [SLEItem] = []
      for idx in 0..<subviews.count {
        items.append(try layout.add(item: .size(cache.childSize)))
        try layout.add(item: .width(cache.distances[idx]))
      }

      for (idx, subview) in subviews.enumerated() {
        subview.place(
          at: try items[idx].frame().origin,
          proposal: ProposedViewSize(cache.childSize)
        )
      }
    }
    catch { print("Unable to layout \(error)") }
  }
```

![custom-hstack]({{ site.url }}/assets/swiftui-custom-layout-with-simple-layout-engine/02.png)

And we can even make it work along both axis by providing the `direction` with init

```swift
extension SLEDirection {
  var axis: Axis {
    switch self {
    case .row: return .horizontal
    case .column: return .vertical
    }
  }
}

struct BalancedStack: Layout {

  let direction: SLEDirection

  init(_ direction: SLEDirection) {
    self.direction = direction
  }

  struct CacheData {
    let childSize: CGSize
    let distances: [CGFloat]
  }

  func makeCache(subviews: Subviews) -> CacheData {
    let subviewSizes = subviews.map { $0.sizeThatFits(.unspecified) }
    let width = subviewSizes.map { $0.width }.max() ?? 0
    let height = subviewSizes.map { $0.height }.max() ?? 0
    let distances: [CGFloat] = (0..<subviews.count).map { idx in
      guard idx < subviews.count - 1 else { return 0 }
      return subviews[idx].spacing.distance(to: subviews[idx + 1].spacing, along: direction.axis)
    }
    return CacheData(
      childSize: CGSize(width: width, height: height),
      distances: distances
    )
  }

  func sizeThatFits(proposal: ProposedViewSize,
                    subviews: Subviews,
                    cache: inout CacheData) -> CGSize {
    let totalDistance = cache.distances.reduce(0, +)
    let containerWidth: CGFloat
    let containerHeight: CGFloat
    switch direction {
    case .row:
      containerWidth = cache.childSize.width * CGFloat(subviews.count) + totalDistance
      containerHeight = cache.childSize.height
    case .column:
      containerWidth = cache.childSize.width
      containerHeight = cache.childSize.height * CGFloat(subviews.count) + totalDistance
    }

    return CGSize(width: containerWidth, height: containerHeight)
  }

  func placeSubviews(in bounds: CGRect,
                     proposal: ProposedViewSize,
                     subviews: Subviews,
                     cache: inout CacheData) {
    let layout = SLELayout(parentFrame: bounds, direction: direction, alignment: .center)
    do {
      var items: [SLEItem] = []
      for idx in 0..<subviews.count {
        items.append(try layout.add(item: .size(cache.childSize)))
        try layout.add(item: .width(cache.distances[idx]))
      }

      for (idx, subview) in subviews.enumerated() {
        subview.place(
          at: try items[idx].frame().origin,
          proposal: ProposedViewSize(cache.childSize)
        )
      }
    }
    catch { print("Unable to layout \(error)") }
  }
}
```

![custom-stack]({{ site.url }}/assets/swiftui-custom-layout-with-simple-layout-engine/03.png)

This can then also be used do automatically adapt the layout based on the available size with `ViewThatFits`

```swift
struct TextList: View {
  var body: some View {
    WLText("hi")
    WLText("!")
    WLText("beautiful")
    WLText("world")
  }
}
```

```swift
ViewThatFits {
    BalancedStack(.row) {
        TextList()
    }

    BalancedStack(.column) {
        TextList()
    }
}
```

# References

1. [Simple Layout Engine](https://github.com/chunkyguy/SimpleLayoutEngine)
1. [Layout](https://developer.apple.com/documentation/swiftui/layout)
2. [ViewThatFits](https://developer.apple.com/documentation/swiftui/viewthatfits/)