---
layout: post
title:  "Introducing VFL"
date:   2023-07-01 18:30:00 +0200
categories: swift uikit layout
published: true
---

There are a plethora of wrappers around the AutoLayout engine. They all seem to provide convenience towards making the layout code look cleaner. In all fairness this is due to the fact that the first version of `NSLayoutContraint` API was very verbose. But I remember when Apple announced the constraint based layout they were equally excited about providing a what they referred to as *'ascii based constraints'*. The idea is to create constraints between siblings and parent UI elements in a string based format. They even provide a guide to "Visual Format Language" aka **VFL**.

The grammar of the syntax is very simple:

| Symbol | Meaning |
|---|---|
| V | Vertical axis |
| H | Horizontal axis |
| - | System spacing |
| -16- | Constant value of 16 |
| &#124; | Parent view |
| [view] | Child view |
| [view(==300)] | Exact size of 300 |
| [view(>=300)] | Greater than or equal to size of 300 |
| [view(<=300)] | Less than or equal to size of 300 |
| [view(>=40,<=80)] | Size between 40 and 80 |
| [label(textField)] | Set size of label equal to that of textField |

As you can realize with this syntax you can build up pretty much a lot of the UI in very little code. To help with reducing the boilerplate code I wrote a simple library called [**VFL**](https://github.com/chunkyguy/VFL). I call it a library even though it's just a single file called `VFL.swift` with a bunch of methods.

The minimal usage looks like:

```swift
let view = UIView(frame: .zero)
VFL(self)
    .add(subview: view, name: "view")
    .appendConstraints(formats: [
        "V:|[view]|", "H:|[view]|"
    ])
```

So if we have a `VFLExampleView` defined as:

```swift
class VFLExampleView: UIView {
  init() {
    super.init(frame: .zero)
    setUp()
  }
  
  required init?(coder: NSCoder) {
    fatalError("init(coder:) has not been implemented")
  }
  
  func setUp() {
    // override
  }
}
```

And a `VFLColorView` that generates a random background color

```swift
extension UIColor {
  static var random: UIColor {
    return UIColor(
      hue: CGFloat.random(in: 0...1),
      saturation: 0.8, brightness: 0.8, alpha: 1
    )
  }
}

class VFLColorView: UIView {
  init(color: UIColor = .random) {
    super.init(frame: .zero)
    backgroundColor = color
  }
  
  required init?(coder: NSCoder) {
    fatalError("init(coder:) has not been implemented")
  }
}
```

We can use **VFL** to layout some common scenario as:

```swift
// Pin subview to all edges
class VFLFullView: VFLExampleView {
  override func setUp() {
    super.setUp()
    VFL(self)
      .add(subview: VFLColorView(), name: "view")
      .appendConstraints(formats: [
        "V:|[view]|", "H:|[view]|"
      ])
  }
}
```
![VFLFullView]({{ site.url }}/assets/vfl/VFLFullView.png)

```swift
// Pin subview of size 100x100 to top-left corner with default spacing
class VFLFixView: VFLExampleView {
  override func setUp() {
    super.setUp()
    VFL(self)
      .add(subview: VFLColorView(), name: "view")
      .appendConstraints(formats: [
        "V:|-[view(100)]",
        "H:|-[view(100)]"
      ])
  }
}
```
![VFLFixView]({{ site.url }}/assets/vfl/VFLFixView.png)


```swift
// Add two subviews of equal width and same height as parent
class VFLHSplitView: VFLExampleView {
  override func setUp() {
    super.setUp()
    VFL(self)
      .add(subview: VFLColorView(), name: "leftVw")
      .add(subview: VFLColorView(), name: "rightVw")
      .appendConstraints(formats: [
        "V:|[leftVw]|", "V:|[rightVw]|",
        "H:|[leftVw(==rightVw)][rightVw]|"
      ])
  }
}
```
![VFLHSplitView]({{ site.url }}/assets/vfl/VFLHSplitView.png)

```swift
// Add two subviews of equal height and same width as parent
class VFLVSplitView: VFLExampleView {
  override func setUp() {
    super.setUp()
    VFL(self)
      .add(subview: VFLColorView(), name: "topVw")
      .add(subview: VFLColorView(), name: "bottomVw")
      .appendConstraints(formats: [
        "H:|[topVw]|", "H:|[bottomVw]|",
        "V:|[topVw(==bottomVw)][bottomVw]|"
      ])
  }
}
```
![VFLVSplitView]({{ site.url }}/assets/vfl/VFLVSplitView.png)

But **VFL** is not a replacement for `NSLayoutContraint`, rather an supplement. So we should be able to add constraints created via the system API. Here's an example:

```swift
// Add a subview of size 320x480 and pinned to the center of parent
class VFLCenterView: VFLExampleView {
  override func setUp() {
    super.setUp()
    
    let view = VFLColorView()
    VFL(self)
      .add(subview: view, name: "view")
      .appendConstraints(formats: [
        "H:[view(320)]",
        "V:[view(480)]",
      ])
      .appendConstraints([
        NSLayoutConstraint(
          item: view, attribute: .centerX,
          relatedBy: .equal,
          toItem: self, attribute: .centerX,
          multiplier: 1, constant: 0
        ),
        NSLayoutConstraint(
          item: view, attribute: .centerY,
          relatedBy: .equal,
          toItem: self, attribute: .centerY,
          multiplier: 1, constant: 0
        )
      ])
  }
}
```
![VFLCenterView]({{ site.url }}/assets/vfl/VFLCenterView.png)

For more advanced cases, like supporting different layouts for landscape and portrait orientations we can simply add the subviews once and then define two set of constraints. Since these constraints need to replace the older constraints rather than append, we need to call `replaceConstraints()` instead of `appendConstraints()`.

Here's an example:

```swift
// add subviews
let topVw = UIImageView(image: UIImage(named: "square"))
let leftVw = UIImageView(image: UIImage(named: "square"))
let rightVw = UIImageView(image: UIImage(named: "square"))
let borderVw = VFLColorView(color: .green)
let vfl = VFL()

vfl
    .setParent(self)
    .add(subview: borderVw, name: "borderVw")
    .add(subview: topVw, name: "topVw")
    .add(subview: leftVw, name: "leftVw")
    .add(subview: rightVw, name: "rightVw")
```

```swift
// detect the orientation and call appropriate method
override func layoutSubviews() {
    super.layoutSubviews()
    if bounds.width < bounds.height {
        layoutSubviewsPortrait()
    } else {
        layoutSubviewsLandscape()
    }
}
```

```swift
// portait layout
private func layoutSubviewsPortrait() {
    vfl.replaceConstraints(
        metrics: [
            "w": bounds.width,
            "hw": bounds.width / 2
          ],
        formats: [
            "V:|[topVw(w)][borderVw(40)]",
            "H:|[topVw]|",
            "H:|[borderVw]|",
            "V:[leftVw(hw)]|",
            "V:[rightVw(hw)]|",
            "H:|-[leftVw(==rightVw)]-[rightVw]-|"
          ]
        )
}
```
![VFLComplexView portrait]({{ site.url }}/assets/vfl/VFLComplexView_p.png)

```swift
// landscape layout
private func layoutSubviewsLandscape() {
    vfl.replaceConstraints(
        metrics: [
          "w": bounds.height,
          "hw": bounds.height / 2.0,
        ],
        formats: [
          "V:|[topVw]|",
          "H:|[topVw(w)][borderVw(40)]",
          "V:|[borderVw]|",
          "V:|-[leftVw(==rightVw)]-[rightVw]-|",
          "H:[leftVw(hw)]|",
          "H:[rightVw(hw)]|",
        ]
      )
}
```
![VFLComplexView landscape]({{ site.url }}/assets/vfl/VFLComplexView_l.png)

I think of **VFL** as a good candidate for simple fixed sort of layouts and use the regular `NSLayoutContraint` for more complicated usage, like for scenarios where constraints needs to be animated.

Checkout the source code: [github.com/chunkyguy/VFL](https://github.com/chunkyguy/VFL)

## References
- [Visual Format Language](https://developer.apple.com/library/archive/documentation/UserExperience/Conceptual/AutolayoutPG/VisualFormatLanguage.html)
- [Auto Layout Visual Format Language Tutorial](https://www.kodeco.com/277-auto-layout-visual-format-language-tutorial)
