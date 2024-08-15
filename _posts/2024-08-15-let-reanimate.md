---
layout: post
title: Let's reanimate
date: 2024-08-15 17:16 +0200
categories: js reactnative animation
published: true
---

Drawing text and images is one thing but the real test of a UI framework is in how good is it with animating content.

<a href="https://imgflip.com/i/9098i4"><img src="https://i.imgflip.com/9098i4.jpg" title="made at imgflip.com"/></a>

And my test for animation is the classic [MoveMe](https://developer.apple.com/library/archive/samplecode/MoveMe/Introduction/Intro.html#//apple_ref/doc/uid/DTS40007315) based on the Apple's sample code. 

The idea is to draw three boxes on the screen. When selected the box changes color and scales up and then can be moved around with the drag gesture and eventually restores back to original color and size when released.

Let's build that sample using React Native's Reanimated library.

### Setup
I'm following the official docs but not using their template. So I've a basic project created with the blank template and installed the dependencies

```
npx create-expo-app playground --template blank
npx expo install react-native-reanimated
npx expo install react-native-gesture-handler
```

Next, I added the plugin

```jsx
module.exports = function (api) {
  api.cache(true);
  return {
    presets: ["babel-preset-expo"],
    plugins: ["react-native-reanimated/plugin"],
  };
};
```

And then draw 3 squares on screen:

```jsx
import { StatusBar } from "expo-status-bar";
import { StyleSheet, View } from "react-native";
import Animated from "react-native-reanimated";

function Square() {
  return <Animated.View style={styles.square}></Animated.View>;
}

export default function App() {
  return (
    <View style={styles.container}>
      <StatusBar style="auto" />
      <Square />
      <Square />
      <Square />
    </View>
  );
}

const styles = StyleSheet.create({
  container: {
    flex: 1,
    backgroundColor: "#fff",
    alignItems: "center",
    justifyContent: "space-evenly",
  },
  square: {
    width: 100,
    height: 100,
    backgroundColor: "blue",
  },
});
```
![set up]({{site.url}}/assets/reanimate/setup.png)

### Add gesture handler
To add support for gesture handlers we first need to wrap the content within the `GestureHandlerRootView`

```jsx
<GestureHandlerRootView style={styles.container}>
  <Square />
  <Square />
  <Square />
</GestureHandlerRootView>
```

And then wrap each `Square` within `GestureDetector`

```jsx
function Square() {
  const gesture = Gesture.Pan();

  return (
    <GestureDetector gesture={gesture}>
      <Animated.View style={styles.square} />
    </GestureDetector>
  );
}
```

### Handle gesture events
To handle gesture we first need to create a `SharedValue` which is like `State` but for animation states. For example, to change the background color when selected we need to listen to `onBegin` and `onFinalize` events and update the style:

```jsx
function Square() {
  const isPressed = useSharedValue(false);
  const animStyle = useAnimatedStyle(() => {
    return {
      backgroundColor: isPressed.value ? "red" : "blue",
    };
  });

  const gesture = Gesture.Pan()
    .onBegin(() => {
      isPressed.value = true;
    })
    .onFinalize(() => {
      isPressed.value = false;
    });

  return (
    <GestureDetector gesture={gesture}>
      <Animated.View style={[styles.square, animStyle]} />
    </GestureDetector>
  );
}
```

Supporting drag is similar. We need to store start and current positions and then update the current position on `onChange` event. The `onChange` provides the delta change that we then need to add to the start position to calculate the final current position. And then, finally at the `onFinalize` event we can sync the start and current positions.

```jsx
function Square() {
  const isPressed = useSharedValue(false);
  const startPos = useSharedValue({ x: 0, y: 0 });
  const pos = useSharedValue({ x: 0, y: 0 });
  const animStyle = useAnimatedStyle(() => {
    return {
      backgroundColor: isPressed.value ? "red" : "blue",
      transform: [
        { translateX: pos.value.x },
        { translateY: pos.value.y },
        { scale: withSpring(isPressed.value ? 1.2 : 1) },
      ],
    };
  });

  const gesture = Gesture.Pan()
    .onBegin(() => {
      isPressed.value = true;
    })
    .onChange((e) => {
      pos.value = {
        x: startPos.value.x + e.translationX,
        y: startPos.value.y + e.translationY,
      };
    })
    .onFinalize(() => {
      isPressed.value = false;
      startPos.value = {
        x: pos.value.x,
        y: pos.value.y,
      };
    });

  return (
    <GestureDetector gesture={gesture}>
      <Animated.View style={[styles.square, animStyle]} />
    </GestureDetector>
  );
}
```

And there you have it

![final]({{site.url}}/assets/reanimate/final.gif)

### References
- [react-native-reanimated](https://docs.swmansion.com/react-native-reanimated)
- [react-native-gesture-handler](https://docs.swmansion.com/react-native-gesture-handler)
- [The basics of PanGestureHandler with React Native Reanimated 2](https://youtu.be/4HUreYYoE6U?si=rZ53Yvft9nbdlGAC)
- [Data-Driven UI with UIKit](https://whackylabs.com/layout/ui/swift/ios/2022/12/12/layout-driven-ui/)