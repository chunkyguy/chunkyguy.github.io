---
layout: post
title: Dear expo, who are you?
date: 2024-07-23 22:58 +0200
categories: js reactnative
published: true
---

<a href="https://imgflip.com/i/8ymeiz"><img src="https://i.imgflip.com/8ymeiz.jpg" title="made at imgflip.com"/></a>

Let's try expo. The motivation behind this experiment is neither my eternal curiosity for exploring new technologies nor is this post sponsored by expo in any way. Rather it is the fact that the nice folks at React Native have killed our beloved React Native CLI. You don't even see the instructions to create React Native apps using the CLI anymore, as if it never existed! So, without losing another tear on the past let's take a sneak peek into the future.

### Set up

The official command on [React Native official docs](https://reactnative.dev/docs/environment-setup) to create a new project `npx create-expo-app@latest` generates a ton of boilerplate. It's better to use the command they hand out at the official [expo docs](https://docs.expo.dev/tutorial/create-your-first-app/). 

`npx create-expo-app PhotoApp --template blank`. 

Which brings to first source of ambiguity - which docs are more recent? But continuing with our experiment step 1 is probably all you need to get started. Next, running `npm run ios` generates a nice UI

![Hello expo]({{site.url}}/assets/hello-expo/hello_expo.png)

Looking at the project directory in details shows that there are way fewer files and folders:

```
.
├── App.js
├── app.json
├── assets
├── babel.config.js
├── node_modules
├── package-lock.json
└── package.json
```

Now compare this with the older files and folders:

```
.
├── App.jsx
├── Gemfile
├── Gemfile.lock
├── README.md
├── __tests__
├── android
├── app.json
├── babel.config.js
├── index.js
├── ios
├── jest.config.js
├── metro.config.js
├── node_modules
├── package-lock.json
├── package.json
├── tsconfig.json
└── vendor
```

### Expo router
Since I don't think a lot has changed in the UI and state management. I'm going to jump directly at the real deal - the expo router. And with the real React Native developer spirit, just copy this one liner and install these bunch of dependencies and see the time fly

`npx expo install expo-router react-native-safe-area-context react-native-screens expo-linking expo-constants expo-status-bar`

Next, update the entry point to `expo-router`
```diff
-  "main": "expo/AppEntry.js"
+  "main": "expo-router/entry"
```

And add `scheme` to `app.json`
```diff
+  "scheme": "your-app-scheme"
```

And then again ignoring all the rest of the steps, since we don't care about the web. Voila we have switched to expo-router!

![Hello expo-router]({{site.url}}/assets/hello-expo/hello_expo_router.png)

### Navigation
At first glance it seems the navigation is heavily inspired by next.js. Every screen is path based auto linked. So `index.js` links to the `/` path, aka the `Home`. And for details screen we can use the parameterized path, like `/photos/[id]` that would then magically navigate to the right `Details` screen. And finally to describe the container we use the special component called `_layout.js`

For our app then the folder structure looks like:

```
./app
├── _layout.js
├── index.js
└── photos
    └── [id].js
./components
├── photo_list.js
└── photo_tile.js
```

Since we want the navigation to be simple stack based, our `_layout.js` looks like:

```jsx
export default function RootLayout() {
  return (
    <Stack>
      <Stack.Screen name="index" />
      <Stack.Screen name="photos/[id]" />
    </Stack>
  );
}
```

Then within the `/index.js` file is where we render a `Home` screen that renders a `FlatList` with each cell rendering a cell with an image and a title. And whenever the cell is tapped we invoke the router to take us to the `photo/[id]` path

```jsx
<PhotoList
  photoList={photoList}
  onSelectPhoto={({ title, url }) => {
    router.push({
      pathname: "/photos/[id]",
      params: { title, url },
    });
  }}
/>
```

And later on in the `/photos/[id].js` we can read the passed params and draw the UI

```jsx
export default function DetailsScreen() {
  const { title, url } = useLocalSearchParams();
  return (
    <SafeAreaView style={styles.container}>
      <Image style={styles.image} source={{ uri: url }} />
      <Text style={styles.title}>{title}</Text>
    </SafeAreaView>
  );
}
```

### Conclusion
And that's all there is the learn. Expo looks neat. I like the path based navigation. Feels way better than other frameworks. And the best part is, like Steve Jobs would say, that it just works.

I also like that they cleaned up the project a lot. But that said, I don't know if I'm going to miss having access to the dedicated `ios` and `android` folders. What if for some reason I have to add some data to `Info.plist` file or toggle some specific build setting. I hope there's a easy way to make that work.

**TLDR; I like expo and for my future React Native project I'm definitely going to use expo!**

The code from this experiment is with the rest of the project at [https://github.com/chunkyguy/PhotoApp](https://github.com/chunkyguy/PhotoApp/tree/master/react-native/PhotoApp_expo)