---
layout: post
title: Juice it up with UIKit Dynamics
date: 2026-03-02 11:51 +0100
categories: swift ios uikit dynamics
published: true
---

One of my favorite framework on iOS has to be UIKit Dynamics and not enough people talk about it. But this is going to change today. Let's build a color palette generator and have some fun with UIKit Dynamics along the way.

![Hero](/assets/2026-03-02-juice-it-up-with-uikit-dynamics/hero.jpg)
> Photo by <a href="https://unsplash.com/@bharath9110?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText">Bharath Kumar</a> on <a href="https://unsplash.com/photos/a-group-of-white-balls-floating-in-the-air-cA9HLrY2FC8?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText">Unsplash</a>
      

### Set up
To get started we need a simple `UICollectionView` that draws a 4 cells per screen with each cell colored as one of the color of the color palette.

```swift
class ViewController: UIViewController {

  private static let cellId = "cell-id"
  private static let maxPalettes = 24

  private let colors: [UIColor] = {
    // generate a tetradic color palette by generating a random hue value
    // and 4 corresponding colors from that base hue: [h, h+0.25, h+0.5, h+0.75]
    (0..<maxPalettes).flatMap { _ in
      let hueBase = CGFloat.random(in: 0...1)
      let saturation = CGFloat.random(in: 0.5...1)
      return (0..<4).map {
        UIColor(
          hue: hueBase + (CGFloat($0) * 0.25),
          saturation: saturation,
          brightness: CGFloat.random(in: 0.5...1),
          alpha: 1
        )
      }
    }
  }()

  override func viewDidLoad() {
    super.viewDidLoad()

    let layout = UICollectionViewFlowLayout()
    // each cell is 1/4th the height of screen
    layout.itemSize = CGSize(
      width: view.bounds.width,
      height: view.bounds.height * 0.25
    )
    layout.minimumInteritemSpacing = 0
    layout.minimumLineSpacing = 0

    let collectionView = UICollectionView(
      frame: view.bounds,
      collectionViewLayout: layout
    )

    view.addSubview(collectionView)

    collectionView.register(
      UICollectionViewCell.self,
      forCellWithReuseIdentifier: Self.cellId
    )
    collectionView.dataSource = self
    collectionView.contentInsetAdjustmentBehavior = .never
    collectionView.isPagingEnabled = true
  }
}

extension ViewController: UICollectionViewDataSource {
  func collectionView(
    _ collectionView: UICollectionView,
    numberOfItemsInSection section: Int
  ) -> Int {
    return colors.count
  }

  func collectionView(
    _ collectionView: UICollectionView,
    cellForItemAt indexPath: IndexPath
  ) -> UICollectionViewCell {
    let cell = collectionView.dequeueReusableCell(
      withReuseIdentifier: Self.cellId,
      for: indexPath
    )
    cell.backgroundColor = colors[indexPath.item]
    return cell
  }
}
```

<video controls width="300">
  <source src="/assets/2026-03-02-juice-it-up-with-uikit-dynamics/01-setup.mp4" type="video/mp4" />
</video>


### Add UIKit Dynamics

The nice and easy way of introducing UIKit Dynamics in a `UICollectionView` is by subclassing a `UICollectionViewFlowLayout` and use that subclass to provide the effective layout data to the `UICollectionView`

```swift
class BouncyListLayout: UICollectionViewFlowLayout { 
  // ...
}

class ViewController: UIViewController {

  override func viewDidLoad() {
    super.viewDidLoad()

    let layout = BouncyListLayout()
    // ...
    let collectionView = UICollectionView(
      frame: view.bounds,
      collectionViewLayout: layout
    )
    // ...
  }
  // ...
}
```

Within the `BouncyListLayout` we first need to initialize a `UIDynamicAnimator` instance. The idea is it have an `UIDynamicAnimator` and one `UIDynamicBehavior` per item. We let the base class `UICollectionViewFlowLayout` do the heavy lifting of creating the `UICollectionViewLayoutAttributes` we then simply use this `UICollectionViewLayoutAttributes` to create the `UIDynamicBehavior` and let `UIDynamicAnimator` run the physics simulation and return the data back to `UICollectionView`.

```swift
class BouncyListLayout: UICollectionViewFlowLayout {
  lazy var animator = UIDynamicAnimator(collectionViewLayout: self)

  override func prepare() {
    super.prepare()
    setUp()
  }

  private func setUp() {
    // ...
  }

  override func layoutAttributesForElements(in rect: CGRect) -> [UICollectionViewLayoutAttributes]? {
    return animator.items(in: rect).compactMap { $0 as? UICollectionViewLayoutAttributes }
  }

  override func layoutAttributesForItem(at indexPath: IndexPath) -> UICollectionViewLayoutAttributes? {
    return animator.layoutAttributesForCell(at: indexPath)
  }
}
```

This mapping between physics world and the UI world works thanks to the fact that `UICollectionViewLayoutAttributes` conforms to the protocol `UIDynamicItem` and `UIDynamicAnimator` providing convenience methods to work with `UICollectionView`

```swift
class UICollectionViewLayoutAttributes : NSObject, NSCopying, UIDynamicItem { 
  // ...
}

extension UIDynamicAnimator {

    convenience init(
      collectionViewLayout layout: UICollectionViewLayout
    )

    func items(in rect: CGRect) -> [any UIDynamicItem]

    func layoutAttributesForCell(
      at indexPath: IndexPath
    ) -> UICollectionViewLayoutAttributes?
}
```

In our `setUp` method the simplest and the least performant way is to let the `UICollectionViewFlowLayout` prepare all the items. We invoke `super.layoutAttributesForElements` for the entire content size. Then for each item we create a `UIAttachmentBehavior` that controls the springiness of each item. The spring is anchored to the center of the item.


```swift
class BouncyListLayout: UICollectionViewFlowLayout {
  // ...

  private func setUp() {
    guard animator.behaviors.isEmpty else {
      // animators already setup - skip
      return
    }
    let items = super.layoutAttributesForElements(
      in: CGRect(origin: .zero, size: collectionViewContentSize)
    ) ?? []

    for item in items {
      let spring = UIAttachmentBehavior(item: item, attachedToAnchor: item.center)
      spring.length = 0
      spring.frequency = 1.0
      spring.damping = 2.0
      spring.frictionTorque = 2.0
      animator.addBehavior(spring)
    }
  }
  // ...
}
```

The final piece of the magic comes from `shouldInvalidateLayout`, which is invoked whenever the content within the `UICollectionView` updates. Remember that the `UIScrollView` works by changing the `bounds` origin value. 

The plan here is to adjust the spring center by a certain offset based on how far it is from the touch point. So, items just under the finger have no spring effect while the items furthest have the most spring effect. And then finally call the `animator.updateItem(usingCurrentState: item)` to update the `animator` which would then in turn run the physics simulation for the `item`

```swift
class BouncyListLayout: UICollectionViewFlowLayout {
  // ...

  override func shouldInvalidateLayout(forBoundsChange newBounds: CGRect) -> Bool {
    guard let collectionView else { return false }

    let touchPt = collectionView.panGestureRecognizer.location(in: collectionView)
    let dy = newBounds.origin.y - collectionView.bounds.origin.y

    for behaviors in animator.behaviors {
      if let spring = behaviors as? UIAttachmentBehavior,
          let item = spring.items.first as? UICollectionViewLayoutAttributes {
        let offset = abs(touchPt.y - spring.anchorPoint.y)
        let yFactor = offset * 0.0001
        item.center = CGPoint(x: item.center.x, y: item.center.y + (dy * yFactor))
        animator.updateItem(usingCurrentState: item)
      }
    }

    return true
  }
  // ...
}
```

<video controls width="300">
  <source src="/assets/2026-03-02-juice-it-up-with-uikit-dynamics/02-spring-effect.mp4" type="video/mp4" />
</video>

### Performance improvements

Now for the last bit, if our `UICollectionView` has a lot of items, we possibly do not want to load all the springs during setup time but rather on the fly when needed. So we need to get rid of the `setUp` 

```diff
class BouncyListLayout: UICollectionViewFlowLayout {
-  override func prepare() {
-    super.prepare()
-    setUp()
-  }
-
-  private func setUp() {
-    guard animator.behaviors.isEmpty else {
-      // animators already setup - skip
-      return
-    }
-    let items = super.layoutAttributesForElements(
-      in: CGRect(origin: .zero, size: collectionViewContentSize)
-    ) ?? []
-
-    for item in items {
-      let spring = UIAttachmentBehavior(item: item, attachedToAnchor: item.center)
-      spring.length = 0
-      spring.frequency = 1.0
-      spring.damping = 2.0
-      spring.frictionTorque = 2.0
-      animator.addBehavior(spring)
-    }
-  }
}
```

And add spring when needed by keeping track of `IndexPath` for which the springs have already been added. Notice that we are still getting the `UICollectionViewLayoutAttributes` from `UICollectionViewFlowLayout` by invoking the `super` implementation but still returning the values from the `UIDynamicAnimator`


```swift
class BouncyListLayout: UICollectionViewFlowLayout {
  lazy var animator = UIDynamicAnimator(collectionViewLayout: self)
  private var indexPaths: Set<IndexPath> = []

  private func addSpringIfNeeded(toItem item: UICollectionViewLayoutAttributes) {
    if indexPaths.contains(item.indexPath) { return }

    let spring = UIAttachmentBehavior(item: item, attachedToAnchor: item.center)
    spring.length = 0
    spring.frequency = 1.0
    spring.damping = 2.0
    spring.frictionTorque = 2.0
    animator.addBehavior(spring)
    indexPaths.insert(item.indexPath)
  }

  override func layoutAttributesForElements(in rect: CGRect) -> [UICollectionViewLayoutAttributes]? {
    guard let items = super.layoutAttributesForElements(in: rect) else {
      return nil
    }
    items.forEach { addSpringIfNeeded(toItem: $0) }
    return animator.items(in: rect).compactMap { $0 as? UICollectionViewLayoutAttributes }
  }

  override func layoutAttributesForItem(at indexPath: IndexPath) -> UICollectionViewLayoutAttributes? {
    guard let item = super.layoutAttributesForItem(at: indexPath) else {
      return nil
    }
    addSpringIfNeeded(toItem: item)
    return animator.layoutAttributesForCell(at: indexPath)
  }

  // ...
}
```

### References

- [Juice it or lose it](https://www.youtube.com/watch?v=Fy0aCDmgnxg)
- [UICollectionView + UIKit Dynamics](https://www.objc.io/issues/5-ios7/collection-views-and-uidynamics/)
- [UIDynamicsAnimator](https://developer.apple.com/documentation/uikit/uidynamicanimator)
- [UIKit Dynamics Catalog](https://developer.apple.com/library/archive/samplecode/DynamicsCatalog/Introduction/Intro.html#//apple_ref/doc/uid/DTS40013414)
- [Color schemes](https://en.wikipedia.org/wiki/Color_wheel#Color_schemes)