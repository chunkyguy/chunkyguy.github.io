---
layout: post
title: Compose multiplatform is real!
date: 2025-01-12 09:56 +0100
categories: swift kotlin compose
published: true
---

So yes after getting my hands dirty with Kotlin multiplatform the obvious next step would be to try Compose multiplatform. Which I did. And yes it's a game changer.

### Set up
The setup for compose multiplatfrom as of today is a bit weird. First you need to the same set up as for Kotlin multiplatform, which makes sense. And then, you need to visit the [https://kmp.jetbrains.com/](https://kmp.jetbrains.com/) to create the boilerplate project. This web app then gives you a zip file that you download and open with Android Studio. But for me it all went pretty straightforward. No hiccups.

With compse multiplatform you don't have to think in terms for different platforms anymore. Which is great. The folder structure looks like:

```
.
├── README.md
├── build
│   ├── ios
│   ├── kotlin
│   └── tmp
├── build.gradle.kts
├── composeApp
│   ├── build
│   ├── build.gradle.kts
│   └── src
├── gradle
│   ├── libs.versions.toml
│   └── wrapper
├── gradle.properties
├── gradlew
├── gradlew.bat
├── iosApp
│   ├── Configuration
│   ├── iosApp
│   └── iosApp.xcodeproj
├── local.properties
└── settings.gradle.kts
```

So most of the app's source code lives within the *composeApp/src* directory. 

Another nice thing about compose multiplatform is that you can build and run the app on iOS simulator directly from Android Studio.

![Set up](/assets/compose-multiplatform/01_setup.png)

Next installing dependencies is a bit challenging. Since you need to figure out if the dependency is okay for compose multiplatform. Otherwise figure out the alternatives and update the dependencies. 

Fortunately for our *PhotoApp* though, all the libraries we wanted to use are already available for compose multiplatform.

Here's the `libs.versions.toml` and the `composeApp/build.gradle.kts` files we need.

```toml
[versions]
# ...
ktor = "3.0.3"
kotlinxSerializationJson = "1.6.3"
coroutines = "1.9.0"
coil = "3.0.4"
androidx-navigation = "2.8.0-alpha10"

[libraries]
# ...
ktor-client-core = { module = "io.ktor:ktor-client-core", version.ref = "ktor" }
ktor-client-okhttp = { module = "io.ktor:ktor-client-okhttp", version.ref = "ktor" }
ktor-client-darwin = { module = "io.ktor:ktor-client-darwin", version.ref = "ktor" }
ktor-serialization-kotlinx-json = { module = "io.ktor:ktor-serialization-kotlinx-json", version.ref = "ktor" }
ktor-client-content-negotiation = { module = "io.ktor:ktor-client-content-negotiation", version.ref = "ktor" }
kotlinx-coroutines-core = { module = "org.jetbrains.kotlinx:kotlinx-coroutines-core", version.ref = "coroutines" }
kotlinx-coroutines-android = { module = "org.jetbrains.kotlinx:kotlinx-coroutines-android", version.ref = "coroutines" }
kotlinx-serialization-json = { module = "org.jetbrains.kotlinx:kotlinx-serialization-json", version.ref = "kotlinxSerializationJson" }
coil-compose = { module = "io.coil-kt.coil3:coil-compose", version.ref = "coil" }
coil-network-ktor = { module = "io.coil-kt.coil3:coil-network-ktor3", version.ref = "coil" }
androidx-navigation-compose = { module = "org.jetbrains.androidx.navigation:navigation-compose", version.ref = "androidx-navigation" }

[plugins]
# ...
kotlinMultiplatform = { id = "org.jetbrains.kotlin.multiplatform", version.ref = "kotlin" }
kotlin-serialization = { id = "org.jetbrains.kotlin.plugin.serialization", version.ref = "kotlin" }
```

```kotlin
plugins {
  // ...
  alias(libs.plugins.composeMultiplatform)
  alias(libs.plugins.kotlin.serialization)
}

kotlin {

  sourceSets {
    androidMain.dependencies {
      implementation(compose.preview)
      implementation(libs.androidx.activity.compose)
      implementation(libs.ktor.client.okhttp)
    }

    iosMain.dependencies {
      implementation(libs.ktor.client.darwin)
    }

    commonMain.dependencies {
      implementation(compose.runtime)
      implementation(compose.foundation)
      implementation(compose.material)
      implementation(compose.ui)
      implementation(compose.components.resources)
      implementation(compose.components.uiToolingPreview)
      implementation(libs.androidx.lifecycle.viewmodel)
      implementation(libs.androidx.lifecycle.runtime.compose)
      implementation(libs.ktor.client.core)
      implementation(libs.ktor.serialization.kotlinx.json)
      implementation(libs.ktor.client.content.negotiation)
      implementation(libs.kotlinx.coroutines.core)
      implementation(libs.kotlinx.serialization.json)
      implementation(libs.androidx.navigation.compose)
      implementation(libs.coil.compose)
      implementation(libs.coil.network.ktor)
    }
  }

  // ...
}

// ...
```

### Writing code
For the *PhotoApp* I did not had to touch the `composeApp/androidMain` or the `composeApp/iosMain`. But if there's any platform specific code required, this is where it has to live. For our app we only have to touch the `composeApp/commonMain`.

Starting with the network layer, the `NetworkService` looks very much like as with the regualar ktor based Android app.

```kotlin
@Serializable
data class Photo(
  val albumId: Int?,
  val id: Int?,
  val title: String?,
  val url: String?,
  val thumbnailUrl: String?
)
```

```kotlin
class NetworkService {
  private val client = HttpClient() {
    install(ContentNegotiation) { json()}
  }

  suspend fun photos(): List<Photo> {
    return client
      .get("https://jsonplaceholder.typicode.com/photos")
      .body()
  }
}
```

Next up, the components are also very regular Compose based components

```kotlin
@Composable
fun PhotoCardView(
  photoUrl: String?,
  photoTitle: String?,
  modifier: Modifier = Modifier
) {
  Column {
    Card(
      modifier = modifier,
      shape = MaterialTheme.shapes.medium,
      elevation = 8.dp,
    ) {
      AsyncImage(
        model = photoUrl,
        contentDescription = photoTitle,
        contentScale = ContentScale.Crop,
        modifier = Modifier.fillMaxWidth(),
      )
    }
    Text(photoTitle ?: "")
  }
}
```

```kotlin
@Composable
fun PhotoGridView(
  onSelectPhoto: (Photo) -> Unit,
  photos: List<Photo>,
  modifier: Modifier = Modifier,
  contentPadding: PaddingValues = PaddingValues(0.dp),
) {
  LazyVerticalGrid(
    columns = GridCells.Adaptive(150.dp),
    modifier = modifier.padding(horizontal = 4.dp),
    contentPadding = contentPadding,
  ) {
    items(
      items = photos,
      key = { photo -> photo.id ?: 0 }
    ) { photo ->
      Surface(onClick = { onSelectPhoto(photo) }) {
        PhotoCardView(
          photoUrl = photo.thumbnailUrl,
          photoTitle = null,
          modifier = Modifier
            .padding(4.dp)
            .fillMaxWidth()
            .aspectRatio(1f)
        )
      }
    }
  }
}
```

And finally the main `App` component, which is the starting point for the app also looks like a regular compose UI based component

```kotlin
@Composable
fun App() {
  MaterialTheme {
    Surface(
      modifier = Modifier.fillMaxSize(),
      color = MaterialTheme.colors.background,
    ) {
      val scope = rememberCoroutineScope()
      val navController = rememberNavController()
      var photos: List<Photo> by remember { mutableStateOf(emptyList()) }
      LaunchedEffect(true) {
        scope.launch {
          photos = try {
            NetworkService().photos()
          } catch (e: Exception) {
            emptyList()
          }
        }
      }
      NavHost(
        navController = navController,
        startDestination = "home"
      ) {
        composable("home") {
          PhotoGridView(
            onSelectPhoto = { navController.navigate("photos/${it.id}") },
            photos = photos
          )
        }
        composable("photos/{id}") { entry ->
          val photoId = entry.arguments?.getString("id") ?: "0"
          val photo = photos.first { it.id.toString() == photoId }
          PhotoCardView(
            photoUrl = photo.url,
            photoTitle = photo.title,
            modifier = Modifier
              .padding(4.dp)
              .fillMaxWidth()
              .aspectRatio(1f)
          )
        }
      }
    }
  }
}
```

And with this I was able to build and run for both iOS and Android.

![Android home](/assets/compose-multiplatform/02_android_home.png)
![Android details](/assets/compose-multiplatform/03_android_details.png)
![iOS home](/assets/compose-multiplatform/04_ios_home.png)
![iOS details](/assets/compose-multiplatform/05_ios_details.png)

### Conclusion
The first impression of compose multiplatform is really good. To be honest, I was expecting a lot more hiccups. But everything just went fine out of the box. Very impressive! I really loved the ability to run iOS app directly from Android Studio. I'm really looking forward to playing around more with this framework.

The code from this experiment is available at [github.com/chunkyguy/PhotoApp](https://github.com/chunkyguy/PhotoApp/tree/master/compose-multiplatform)

### References
- [https://www.jetbrains.com/compose-multiplatform/](https://www.jetbrains.com/compose-multiplatform/)