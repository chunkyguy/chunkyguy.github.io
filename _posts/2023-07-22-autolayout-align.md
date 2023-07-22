---
layout: post
title:  "Autolayout and aligning subviews"
date:   2023-07-22 15:55:00 +0200
categories: swift uikit layout vfl
published: true
---

Quick question: How would you align all subviews horizontally while keeping their centers aligned using autolayout? 

With visual formatting language that is quite easy. The [method](https://developer.apple.com/documentation/uikit/nslayoutconstraint/1526944-constraints) we are interested in looks like:

```swift
class func constraints(
    withVisualFormat format: String,
    options opts: NSLayoutConstraint.FormatOptions = [],
    metrics: [String : Any]?,
    views: [String : Any]
) -> [NSLayoutConstraint]
```

And we are interested in `NSLayoutConstraint.FormatOptions`:

```swift
// Align all specified interface elements using NSLayoutConstraint.Attribute.left on each.
static var alignAllLeft: NSLayoutConstraint.FormatOptions

// Align all specified interface elements using NSLayoutConstraint.Attribute.right on each.
static var alignAllRight: NSLayoutConstraint.FormatOptions

// Align all specified interface elements using NSLayoutConstraint.Attribute.top on each.
static var alignAllTop: NSLayoutConstraint.FormatOptions

// Align all specified interface elements using NSLayoutConstraint.Attribute.bottom on each.
static var alignAllBottom: NSLayoutConstraint.FormatOptions

// Align all specified interface elements using NSLayoutConstraint.Attribute.leading on each.
static var alignAllLeading: NSLayoutConstraint.FormatOptions

// Align all specified interface elements using NSLayoutConstraint.Attribute.trailing on each.
static var alignAllTrailing: NSLayoutConstraint.FormatOptions

// Align all specified interface elements using NSLayoutConstraint.Attribute.centerX on each.
static var alignAllCenterX: NSLayoutConstraint.FormatOptions

// Align all specified interface elements using NSLayoutConstraint.Attribute.centerY on each.
static var alignAllCenterY: NSLayoutConstraint.FormatOptions

// Align all specified interface elements using the last baseline of each one.
static var alignAllLastBaseline: NSLayoutConstraint.FormatOptions
```

Using the simple [VFL](https://whackylabs.com/swift/uikit/layout/2023/07/01/introducing-vfl/) wrapper, we can play around with these values. So, if we have three subviews of different sizes added to parent view:

```swift
let vfl = VFL()
vfl
    .setParent(self)
    .add(subview: VFLColorView(color: .random), name: "redVw")
    .add(subview: VFLColorView(color: .random), name: "blueVw")
    .add(subview: VFLColorView(color: .random), name: "greenVw")
```

We can create 2 set of constraints, one for horizontal layout and another for vertical layout
```swift
vfl
    .storeConstraints(
        formats: [
            "H:|[redVw(30)][blueVw(60)][greenVw(90)]",
            "V:|-(>=100)-[redVw(30)]",
            "V:|-(>=100)-[blueVw(60)]",
            "V:|-(>=100)-[greenVw(90)]"
        ],
        name: "h_layout"
    )
    .storeConstraints(
        formats: [
            "H:|-(>=100)-[redVw(30)]",
            "H:|-(>=100)-[blueVw(60)]",
            "H:|-(>=100)-[greenVw(90)]",
            "V:|[redVw(30)][blueVw(60)][greenVw(90)]"
        ],
        name: "v_layout"
    )
```

And then we can apply the constraints based on the alignment type:
```swift
let alignment: NSLayoutConstraint.FormatOptions
let vLayouts: NSLayoutConstraint.FormatOptions = [
    .alignAllLeft, .alignAllCenterX, .alignAllRight
]
let constraintsName = vLayouts.contains(alignment) ? "v_layout" : "h_layout"
      
vfl
    .removeAllConstraints()
    .removeAllOptions()
    .addOptions(alignment)
    .applyConstraints(name: constraintsName)
```

Here are the outputs for all various alignment types:

![Left]({{ site.url }}/assets/vfl-align/01.png)
![CenterX]({{ site.url }}/assets/vfl-align/02.png)
![Right]({{ site.url }}/assets/vfl-align/03.png)
![Top]({{ site.url }}/assets/vfl-align/04.png)
![CenterY]({{ site.url }}/assets/vfl-align/05.png)
![Bottom]({{ site.url }}/assets/vfl-align/06.png)

As an extra bonus, if we with to have the changes always animated all we need to add is:

```swift
vfl
    .removeAllConstraints()
    .removeAllOptions()
    .addOptions(alignment)
    .applyConstraints(name: constraintsName)
      
UIView.animate(withDuration: 0.2) {
    self.vfl.parentView?.layoutIfNeeded()
}
```
![Bottom]({{ site.url }}/assets/vfl-align/animation.gif)

The full source code is available at: [https://github.com/chunkyguy/VFL](https://github.com/chunkyguy/VFL)