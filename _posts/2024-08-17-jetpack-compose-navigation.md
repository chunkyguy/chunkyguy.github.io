---
layout: post
title: Jetpack compose navigation
date: 2024-08-17 01:20 +0200
categories: kotlin compose navigation
published: true
---

It's Friday night! And you know what that means right? It's time to dig deeper into Jetpack Compose. And today I would like to try out the navigation between screens with Jetpack Compose.

<a href="https://imgflip.com/i/90jgo1"><img src="https://i.imgflip.com/90jgo1.jpg" title="made at imgflip.com"/></a>

How hard can it?

### Setup
Installing dependencies is never fun in an Android project.
First you need to find the `libs.version.toml` file and add these lines:

```toml
[version]
androidx-navigation = "2.7.7"
kotlinxSerializationJson = "1.6.3"

[libraries]
androidx-navigation-compose = { module = "androidx.navigation:navigation-compose", version.ref = "androidx-navigation" }
kotlinx-serialization-json = { module = "org.jetbrains.kotlinx:kotlinx-serialization-json", version.ref = "kotlinxSerializationJson" }

[plugins]
kotlin-serialization = { id = "org.jetbrains.kotlin.plugin.serialization", version.ref = "kotlin" }
```

Run `gradle sync`. Then open the `build.gradle.kts` within the **app/** and apply the plugin and add the dependency

```
plugins {
  // ...
  alias(libs.plugins.kotlin.serialization)
}

dependencies {
  // ...
  implementation(libs.androidx.navigation.compose)
  implementation(libs.kotlinx.serialization.json)
}
```
And then do another round of `gradle sync`. 

If all went fine so far, then you're all set for writing the code. If not, delete your project and start from scratch.

### Implementation

Navigation in Android works with the help of these 2 components:
- `NavController`: Top level object to navigate between screens
- `NavHost`: Container within which all routing happens.

So to make this work, we first need to inject the `NavController` at the root level of the app. Like say the `MainActivity`

```kotlin
fun PhotoApp(
  navController: NavHostController = rememberNavController()
) { .. }
```

And then simply setup all the available routes

```kotlin
NavHost(
  navController = navController,
  startDestination = "home"
) {
  composable("home") {
    HomeScreen(
      onSelectPhoto = { navController.navigate("photos/${it.id}") },
      modifier = Modifier.fillMaxSize()
    )
  }

  composable("photos/{id}") {
    DetailScreen(
      id = it.arguments?.getString("id") ?: "0",
      modifier = Modifier.fillMaxSize()
    )
  }
}
```

And that is all there to it.

### Thoughts
Now of course, one can go crazy with typed routing with sealed classes and what not. But this is the gist of it.

Just like React Native, the path arguments are by default string types and if there's more data to be passed between the screens we need to come up with our strategy. 

The good thing is the the back button comes for free with Android OS. But for custom back button we again need to devise our own tooling.

So there is how you navigate between screens with Android. The code for this experiment is up there sitting in the cloud nicely with the rest of the PhotoApp. 

https://github.com/chunkyguy/PhotoApp/tree/master/kotlin

### References
- [Navigation with Compose](https://developer.android.com/develop/ui/compose/navigation)
- [Navigate between screens with Compose](https://developer.android.com/codelabs/basic-android-kotlin-compose-navigation)
- [Type-Safe Navigation with the OFFICIAL Compose Navigation Library](https://www.youtube.com/watch?v=AIC_OFQ1r3k&t=141s)
- [Navigation](https://developer.android.com/guide/navigation)
- [Navigation Basics in Jetpack Compose](https://www.youtube.com/watch?v=glyqjzkc4fk)
- [Jetpack Compose Navigation for Beginners - Android Studio Tutorial](https://www.youtube.com/watch?v=4gUeyNkGE3g)