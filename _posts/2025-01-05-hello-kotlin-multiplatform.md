---
layout: post
title: Hello Kotlin Multiplatform
date: 2025-01-05 15:09 +0100
categories: swift kotlin kmm
published: true
---
What better way to start the new year than trying out the Kotlin Multiplatform. 

I wanted to try KMM for a very long time now. But first I wanted to get enough knowledge of Kotlin and the Android development environment. Which I think I've sort of good enough now. So let's take a quick look at KMM.

### Setup

To verify if everything is in order we can use the `kdoctor`. 

```bash
 kdoctor        
...
Conclusion:
  ✓ Your operation system is ready for Kotlin Multiplatform Mobile Development!
```

And then there is the amazing [KMM plugin for Android Studio](https://plugins.jetbrains.com/plugin/14936-kotlin-multiplatform) which provides a nice template for creating a new project from Android Studio.

Next update the dependencies by updating the `libs.versions.toml`. The idea here is to list all of the libraries regardless of the platform, we will get the chance later to add libraries to targets later on.

```toml
[versions]
ktor = "3.0.3"
kotlinxSerializationJson = "1.6.3"
coroutines = "1.9.0"
coil = "3.0.4"
androidx-navigation = "2.7.7"

[libraries]
ktor-client-core = { module = "io.ktor:ktor-client-core", version.ref = "ktor" }
ktor-client-okhttp = { module = "io.ktor:ktor-client-okhttp", version.ref = "ktor" }
ktor-client-darwin = { module = "io.ktor:ktor-client-darwin", version.ref = "ktor" }
ktor-serialization-kotlinx-json = { module = "io.ktor:ktor-serialization-kotlinx-json", version.ref = "ktor" }
ktor-client-content-negotiation = { module = "io.ktor:ktor-client-content-negotiation", version.ref = "ktor" }
kotlinx-coroutines-core = { module = "org.jetbrains.kotlinx:kotlinx-coroutines-core", version.ref = "coroutines" }
kotlinx-coroutines-android = { module = "org.jetbrains.kotlinx:kotlinx-coroutines-android", version.ref = "coroutines" }
kotlinx-serialization-json = { module = "org.jetbrains.kotlinx:kotlinx-serialization-json", version.ref = "kotlinxSerializationJson" }
coil-kt-compose = { module = "io.coil-kt:coil-compose", version.ref = "coil" }
coil-compose = { module = "io.coil-kt.coil3:coil-compose", version.ref = "coil" }
coil-network-okhttp = { module = "io.coil-kt.coil3:coil-network-okhttp", version.ref = "coil" }
androidx-navigation-compose = { module = "androidx.navigation:navigation-compose", version.ref = "androidx-navigation" }

[plugins]
kotlin-serialization = { id = "org.jetbrains.kotlin.plugin.serialization", version.ref = "kotlin" }
```

Next we first need to add the dependencies for the shared library in `shared/build.gradle.kts`.

```kotlin

plugins {
  // ...
  alias(libs.plugins.kotlin.serialization)
}

kotlin {
  // ...

  sourceSets {
    commonMain.dependencies {
      implementation(libs.ktor.client.core)
      implementation(libs.ktor.serialization.kotlinx.json)
      implementation(libs.ktor.client.content.negotiation)
      implementation(libs.kotlinx.coroutines.core)
      implementation(libs.kotlinx.serialization.json)
    }

    androidMain.dependencies {
      implementation(libs.ktor.client.okhttp)
    }

    iosMain.dependencies {
      implementation(libs.ktor.client.darwin)
    }
  }
}

// ...
```

Notice how we provide different ktor engines per platform. So `okhttp` for android and `darwin` for iOS.

Next, for the Android app we need to update the `PhotoAppAndroidApp/build.gradle.kts`

```kotlin
// ...

dependencies {
  implementation(projects.shared)
  implementation(libs.compose.ui)
  implementation(libs.compose.ui.tooling.preview)
  implementation(libs.compose.material3)
  implementation(libs.androidx.activity.compose)
  implementation(libs.kotlinx.coroutines.android)
  implementation(libs.coil.compose)
  implementation(libs.coil.network.okhttp)
  implementation(libs.androidx.navigation.compose)
}
```

Regarding iOS app, we can simply open the `PhotoAppApple/PhotoAppApple.xcodeproj` and update the settings if required. For example I was getting some Java errors due to Xcode unable to find `JAVA_HOME` so I had to manually update the *Build Phase > Run Script* to

```bash
export JAVA_HOME="/opt/homebrew/opt/openjdk"

cd "$SRCROOT/.."
./gradlew :shared:embedAndSignAppleFrameworkForXcode
```

### Networking

Since networking is part of the shared module, we can have the entire network layer as

```kotlin
package com.whackylabs.photoapp

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
package com.whackylabs.photoapp

class NetworkService {
  private val client = HttpClient() {
      install(ContentNegotiation) { json() }
  }

  suspend fun photos(): List<Photo> {
    return client
            .get("https://jsonplaceholder.typicode.com/photos")
            .body()
  }
}
```

### Android UI

The Android UI can be built with the regular Jetpack compose

```kotlin
package com.whackylabs.photoapp.android

class MainActivity : ComponentActivity() {
  override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    setContent {
      MyApplicationTheme {
        Surface(
          modifier = Modifier.fillMaxSize(),
          color = MaterialTheme.colorScheme.background
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
          NavHost(navController = navController, startDestination = "home") {
            composable("home") {
              PhotoGridView(
                onSelectPhoto = { navController.navigate("photos/${it.id}") },
                photos = photos
              )
            }
            composable("photos/{id}") {
              val photoId = it.arguments?.getString("id") ?: "0"
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
  }
}
```

```kotlin
package com.whackylabs.photoapp.android

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
      elevation = CardDefaults.cardElevation(defaultElevation = 8.dp)
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
package com.whackylabs.photoapp.android

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
    items(items = photos, key = { photo -> photo.id ?: 0 }) { photo: Photo ->
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

![Android Home](/assets/hello-kmm/android_01.png)
![Android Details](/assets/hello-kmm/android_02.png)

### iOS UI
Similarly the iOS counterpart can be built with SwiftUI

```swift
@main
struct iOSApp: App {
  var body: some Scene {
    WindowGroup {
      ContentView()
    }
  }
}
```

```swift
struct ContentView: View {
  
  @State var photos: [Photo] = []
  
  let columns = [
    GridItem(.flexible(), alignment: .trailing),
    GridItem(.flexible(), alignment: .leading),
  ]
  
  var body: some View {
    NavigationStack {
      PhotoGridView(photos)
        .navigationTitle("Photos")
        .navigationDestination(for: Photo.self) { photo in
          PhotoCardView(
            photoUrl: photo.url,
            photoTitle: photo.title
          )
        }
    }
    .task {
      do {
        self.photos = try await NetworkService().photos()
      } catch {
        self.photos = []
      }
    }
  }
}
```

```swift
struct PhotoCardView: View {
  var photoUrl: String?
  var photoTitle: String?
  
  var body: some View {
    VStack {
      AsyncImage(url: photoUrl.flatMap(URL.init)) { image in
        image
          .resizable()
          .scaledToFit()
      } placeholder: {
        ProgressView()
      }

      Text(photoTitle ?? "")
    }
  }
}
```

```swift
struct PhotoGridView: View {
  var photos: [Photo] = []
  
  init(_ photos: [Photo]) {
    self.photos = photos
  }

  let columns = [
    GridItem(.flexible(), alignment: .trailing),
    GridItem(.flexible(), alignment: .leading),
  ]

  var body: some View {
    ScrollView {
      LazyVGrid(columns: columns) {
        ForEach(photos, id: \.id) { photo in
          NavigationLink(value: photo) {
            PhotoCardView(photoUrl: photo.thumbnailUrl, photoTitle: nil)
              .frame(width: 150, height: 150)
          }
        }
      }
    }
  }
}
```

![iOS Home](/assets/hello-kmm/ios_01.png)
![iOS Details](/assets/hello-kmm/ios_02.png)

### Conclusion

I really love the vision of Kotlin Multiplatform. It helps with keeping all the business logic in one place while allowing the platform native UI. So you would love near to nothing. I also like it allows us app developers to use all the latest and greatest each platform has to offer. Like Jetpack Compose, Swift UI on the UI side of things while kotlin coroutines that get nicely mapped to Swift async-await API.

The only headache I had was with dealing with errors related to java vm. But I'm really looking forward to KMM becoming more mainstream.

The code from this experiment is available at [github.com/chunkyguy/PhotoApp/tree/master/kmm](https://github.com/chunkyguy/PhotoApp/tree/master/kmm)

### References
- [Kotlin Multiplatform](https://kotlinlang.org/docs/multiplatform.html)
- [Creating Your First Hello World KMM App (Kotlin Multiplatform Mobile) - KMM for Beginners](https://youtu.be/7Wl-G8aXxDA)