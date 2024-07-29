---
layout: post
title:  "UIScrollView and Autolayout"
date: 2024-04-28 12:24 +0200
categories: ios uikit layout
published: true
---

This a quick reference on how to make `UIScrollView` work with autolayout

<a href="https://imgflip.com/gif/8ymfqc"><img src="https://i-download.imgflip.com/8ymfqc.gif" title="made at imgflip.com"/></a>

### Solution 1: Where the content has a fixed size

```swift
class ContentView: UIView {}

class ViewController: UIViewController {
  // ...
  override func viewDidLoad() {
    super.viewDidLoad()
    let contentVw = ContentView(frame: CGRect(
      x: 0, y: 0,
      width: 1024, 
      height: 1024
    ))
    let scrollVw = UIScrollView()
    
    VFL(view)
      .add(subview: scrollVw, name: "scrollVw")
      .applyConstraints(formats: [
        "H:|[scrollVw]|", "V:|[scrollVw]|"
      ])
    
    VFL(scrollVw)
      .addSubview(contentVw)
    
    scrollVw.backgroundColor = .red
    scrollVw.contentSize = contentVw.frame.size
  }
}
```

### Solution 2: Where the content has an intrinsicContentSize

```swift
class ContentView: UIView {
  override var intrinsicContentSize: CGSize {
    CGSize(width: 1024, height: 1024)
  }
}

class ViewController: UIViewController {
  // ...
  override func viewDidLoad() {
    super.viewDidLoad()
    let contentVw = ContentView()
    let scrollVw = UIScrollView()

    VFL(view)
      .add(subview: scrollVw, name: "scrollVw")
      .applyConstraints(formats: [
        "H:|[scrollVw]|", "V:|[scrollVw]|"
      ])
    
    VFL(scrollVw)
      .add(subview: contentVw, name: "contentVw")
      .applyConstraints(formats: [
        "H:|[contentVw]|", "V:|[contentVw]|"
      ])
  }
}
```

### Solution 3: Where the content has a flexible size

```swift
class ContentView: UIView {  
  weak var scrollVw: UIScrollView?
  
  override func layoutSubviews() {
    super.layoutSubviews()    
    scrollVw?.contentSize = bounds.size
  }
}

class ViewController: UIViewController {
  // ...
  override func viewDidLoad() {
    super.viewDidLoad()
    let contentVw = ContentView()
    let scrollVw = UIScrollView()

    VFL(view)
      .add(subview: scrollVw, name: "scrollVw")
      .applyConstraints(formats: [
        "H:|[scrollVw]|", "V:|[scrollVw]|"
      ])
    
    VFL(scrollVw)
      .add(subview: contentVw, name: "contentVw")
      .applyConstraints(formats: [
        "H:[contentVw]", "V:[contentVw]"
      ])

    // layout based on grandparent view    
    VFL(contentVw)
      .appendConstraints([
        contentVw.leadingAnchor.constraint(equalTo: view.leadingAnchor),
        contentVw.trailingAnchor.constraint(
            equalTo: view.trailingAnchor,
            constant: 1024
        ),
        contentVw.topAnchor.constraint(equalTo: view.topAnchor),
        contentVw.bottomAnchor.constraint(
            equalTo: view.bottomAnchor,
            constant: 1024
        ),
      ])
    
    contentVw.scrollVw = scrollVw
  }
}
```

## References:

1. [Introducing VFL](https://whackylabs.com/swift/uikit/layout/2023/07/01/introducing-vfl/)
1. [UIScrollView and Autolayout](https://developer.apple.com/library/archive/technotes/tn2154/_index.html)