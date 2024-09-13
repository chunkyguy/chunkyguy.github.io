---
layout: post
title: MoveMe - SwiftUI Edition
date: 2024-09-12 20:38 +0200
categories: swift swiftui ios animation
image: /assets/moveme-swiftui/meme.jpg
published: true
---

Taking about gestures and animation with SwiftUI is actually not as intuitive as it sounds. But how hard could it be?

![Best way to build an app is with Swift and SwiftUI](/assets/moveme-swiftui/meme.jpg)

### Setup

So like always we need 3 squares nicely lined up in the center of the screen. I'm going to use `ZStack` because I want the squares to layout independently of each other. The only thing the parent container view needs to provide is the initial position.

```swift
struct SquareView: View {
  var position: CGPoint
  
  var body: some View {
    RoundedRectangle(cornerRadius: 25.0, style: .continuous)
      .frame(width: 100, height: 100)
      .position(position)
      .foregroundStyle(.blue)
  }
}

struct ContentView: View {
  var body: some View {
    ZStack {
      ForEach(0..<3) { idx in
        GeometryReader { geometry in
          SquareView(position: CGPoint(
            x: geometry.size.width * 0.5,
            y: geometry.size.height * (CGFloat(idx +  1) / 4.0)
          ))
        }
      }
    }
  }
}
```

![setup]({{site.url}}/assets/moveme-swiftui/01-setup.png)

### Gestures
Next we need a tap gesture and when detected the selected square should become red. The obvious solution would be to use a `TapGesture` but the `TapGesture` only activates at touch end and what we need is a way to detect touch began. The actual solution I found is to use the `DragGesture` with `minimumDistance` set to `0`. 

```swift
struct SquareView: View {
  var position: CGPoint
  @State var isSelected = false
  
  var body: some View {
    RoundedRectangle(cornerRadius: 25.0, style: .continuous)
      .frame(width: 100, height: 100)
      .position(position)
      .foregroundStyle(isSelected ? .red : .blue)
      .gesture(
        DragGesture(minimumDistance: 0)
          .onChanged({ _ in isSelected = true })
          .onEnded({ _ in isSelected = false })
      )
  }
}
```

![touch]({{site.url}}/assets/moveme-swiftui/02-touch.gif)

Next we need to square to scale up when selected. Again the obvious solution is to use the `scaleEffect`.

```swift
struct SquareView: View {
  var position: CGPoint
  @State var isSelected = false
  
  var body: some View {
    RoundedRectangle(cornerRadius: 25.0, style: .continuous)
      .frame(width: 100, height: 100)
      .position(position)
      .foregroundStyle(isSelected ? .red : .blue)
      .scaleEffect(isSelected ? 1.2 : 1)
      .gesture(
        DragGesture(minimumDistance: 0)
          .onChanged({ _ in isSelected = true })
          .onEnded({ _ in isSelected = false })
      )
  }
}
```

But this brings another problem. Notice how the squares are not centered when scaling up.

![scale-bug]({{site.url}}/assets/moveme-swiftui/03-scale-bug.gif)

This is because the view tree in SwiftUI is inverted because each modifier creates a new `View` and wraps the invoking object as its child. So these two are equivalent:

```swift
Box()
 .firstModifier()
 .secondModifier()
```

```swift
SecondModifierView(
  FirstModifierView(
    Box()
  )
)
```

So back in our solution above the `scaleEffect` is applied first and then the `position`. This make the center to become off centered. 
The solution is to apply the `position` modifier after the `scaleEffect`.

```swift
struct SquareView: View {
  var position: CGPoint
  @State var isSelected = false
  
  var body: some View {
    RoundedRectangle(cornerRadius: 25.0, style: .continuous)
      .frame(width: 100, height: 100)
      .scaleEffect(isSelected ? 1.2 : 1)
      .position(position)
      .foregroundStyle(isSelected ? .red : .blue)
      .gesture(
        DragGesture(minimumDistance: 0)
          .onChanged({ _ in isSelected = true })
          .onEnded({ _ in isSelected = false })
      )
  }
}
```

![scale]({{site.url}}/assets/moveme-swiftui/04-scale.gif)

And finally we want the square to move around with the finger. This part is simple since we already have drag gesture, we just need to update the position of the square.

```swift
struct SquareView: View {
  @State var position: CGPoint
  @State var isSelected = false
  
  var body: some View {
    RoundedRectangle(cornerRadius: 25.0, style: .continuous)
      .frame(width: 100, height: 100)
      .scaleEffect(isSelected ? 1.2 : 1)
      .position(position)
      .foregroundStyle(isSelected ? .red : .blue)
      .gesture(
        DragGesture(minimumDistance: 0)
          .onChanged({ value in
            isSelected = true
            position = value.location
          })
          .onEnded({ _ in isSelected = false })
      )
  }
}
```

![drag]({{site.url}}/assets/moveme-swiftui/05-drag.gif)

### Animations
For the next part we would like the transitions to animate. For this we can either use the `withAnimation` block

```swift
struct SquareView: View {
  @State var position: CGPoint
  @State var isSelected = false
  
  var body: some View {
    RoundedRectangle(cornerRadius: 25.0, style: .continuous)
      .frame(width: 100, height: 100)
      .scaleEffect(isSelected ? 1.2 : 1)
      .position(position)
      .foregroundStyle(isSelected ? .red : .blue)
      .gesture(
        DragGesture(minimumDistance: 0)
          .onChanged({ value in
            withAnimation {
              isSelected = true
            }
            position = value.location
          })
          .onEnded({ _ in
            withAnimation {
              isSelected = false
            }
          })
      )
  }
}
```

But look our favorite bug is back again. Notice how the squares are off centered when selected.

![animation-bug]({{site.url}}/assets/moveme-swiftui/06-animation-bug.gif)

But now we know why this bug exists. We need to guarantee that the position is applied after the scaling. But since we are using the `withAnimation` block it sets some flag for the next draw pass to be done with animation. And that means all the changes are animated. But what we want is to have only have the scaling change as animated and everything else non animated.

To achieve this we can use the  `animation` modifier. It does the same thing as `withAnimation` block but we can control where in the `View` hierarchy we want the animation to happen.

```swift
struct SquareView: View {
  @State var position: CGPoint
  @State var isSelected = false
  
  var body: some View {
    RoundedRectangle(cornerRadius: 25.0, style: .continuous)
      .scaleEffect(isSelected ? 1.2 : 1, anchor: .center)
      .animation(.easeOut, value: isSelected)
      .frame(width: 100, height: 100)
      .position(position)
      .foregroundStyle(isSelected ? .red : .blue)
      .gesture(
        DragGesture(minimumDistance: 0)
          .onChanged({ value in
            isSelected = true
            position = value.location
          })
          .onEnded({ _ in
            isSelected = false
          })
      )
  }
}
```

![animation]({{site.url}}/assets/moveme-swiftui/07-animation.gif)

The solution is available on [https://github.com/chunkyguy/MoveMe](https://github.com/chunkyguy/MoveMe/tree/main/swiftui)

### References
- [Explore SwiftUI animation](https://developer.apple.com/videos/play/wwdc2023/10156)
- [Wind your way through advanced animations in SwiftUI](https://developer.apple.com/videos/play/wwdc2023/10157/)