---
layout: post
title: Dear expo, who are you?
date: 2024-07-23 22:58 +0200
categories: js reactnative
published: true
---

Let's try expo. The motivation behind this experiment is neither my eternal curiosity for exploring new technologies nor is this post sponsored by expo in any ways. Rather it is the fact that the nice folks at React Native have killed the beloved React Native CLI. You don't even see the instructions to create React Native apps using the CLI, as if it never existed! So, without losing another tear on the past let's take a sneak peek into the future.

### Set up

The official command on [React Native official docs](https://reactnative.dev/docs/environment-setup) to create a new project `npx create-expo-app@latest` generates a lot of boilerplate code. It's better to use the command the hand out at the official [expo docs](https://docs.expo.dev/tutorial/create-your-first-app/), `npx create-expo-app PhotoApp --template blank`. Which brings to first source of ambiguity - which docs are more recent?

But step 1 is probably all you need to get started. Running `npm run ios` generates a nice UI

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
├── components
├── index.js
├── ios
├── jest.config.js
├── metro.config.js
├── node_modules
├── package-lock.json
├── package.json
├── screens
├── tsconfig.json
└── vendor
```

### Expo router
Since I don't think a lot has changed in the UI and state management. I'm going to jump directly at the real deal - the expo router. In the real React Native developer spirit, just install these dependencies:
`npx expo install expo-router react-native-safe-area-context react-native-screens expo-linking expo-constants expo-status-bar`

Update the entry point
```diff
-  "main": "expo/AppEntry.js"
+  "main": "expo-router/entry"
```

Add `scheme` to `app.json`
```diff
+  "scheme": "your-app-scheme"
```

And again ignoring all the rest of the steps, since we don't care about the web. Voila!

![Hello expo-router]({{site.url}}/assets/hello-expo/hello_expo_router.png)

### Navigation
It seems the navigation is heavily inspired by next.js. Every screen is path based. `index.js` the `/` path, aka the `Home`. And for details screen we can use the parameterized path, like `/photos/[id]` that would then switch to the `Details` screen. And finally to describe the container we use the special component called `_layout.js`

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
      <Stack.Screen
        name="index"
        options={{
          title: "Home",
        }}
      />
      <Stack.Screen
        name="photos/[id]"
        options={{
          title: "Details",
        }}
      />
    </Stack>
  );
}
```

Then within the `/index.js` where we render a `FlatList` with each cell rendering a image and a title. And whenever a cell is tapped we invoke the router as

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
Expo looks neat. I like the path based navigation. Feels way better than other frameworks and the best part is that it just works.
I also like that they cleaned up the project a lot. 

But that said, I don't know if I'm going to miss having access to the dedicated `ios` and `android` folders. What if for some reason I have to add some data to `Info.plist` file or toggle some specific build setting. I hope there's a easy way to make that work.

For my future React Native project I'm definitely going to use expo!

The code from this experiment is with the rest of the project at [https://github.com/chunkyguy/PhotoApp](https://github.com/chunkyguy/PhotoApp/tree/master/react-native/PhotoApp_expo)