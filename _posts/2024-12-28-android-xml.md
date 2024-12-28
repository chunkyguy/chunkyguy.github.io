---
layout: post
title: Finally trying out the Android XML layout
date: 2024-12-28 10:55 +0100
categories: kotlin xml android
published: true
---

So year after year since the dawn of the Android SDK back in 2009 I've been making the yearly resolution to try it out. But for one reason or the other I've always been putting it off. But not this year. Before the sun goes down for the last time for the year of 2024 let's make our favorite photo app using the classic Android UI toolkit - The framework with no name - You know that thing where you write XML for layout. Let the fun begin!

### Set up

I'm going to use retrofit for networking with glide for loading images all connected together via kotlin coroutines.

The first step is to install all the dependencies. Which from what I understand is now a 2 step job. First you list all the dependencies in the `libs.versions.toml` file

```toml
[versions]
kotlin = "2.0.0"
coroutinesKtx = "2.4.0"
coroutines = "1.9.0"
gson = "2.11.0"
retrofit = "2.11.0"
okhttp = "4.12.0"
glide = "4.16.0"

[libraries]
retrofit = { module = "com.squareup.retrofit2:retrofit", version.ref = "retrofit" }
gson = { module = "com.google.code.gson:gson", version.ref = "gson" }
retrofit-gson-converter = { module = "com.squareup.retrofit2:converter-gson", version.ref = "retrofit" }
okhttp-logging-interceptor = { module = "com.squareup.okhttp3:logging-interceptor", version.ref = "okhttp" }
glide = { module = "com.github.bumptech.glide:glide", version.ref = "glide" }
glide-compiler = { module = "com.github.bumptech.glide:compiler", version.ref = "glide" }
kotlinx-coroutines-core = { module = "org.jetbrains.kotlinx:kotlinx-coroutines-core", version.ref = "coroutines" }
kotlinx-coroutines-android = { module = "org.jetbrains.kotlinx:kotlinx-coroutines-android", version.ref = "coroutines" }
androidx-viewmodel-ktx = { module = "androidx.lifecycle:lifecycle-viewmodel-ktx", version.ref = "coroutinesKtx" }
androidx-runtime-ktx = { module = "androidx.lifecycle:lifecycle-runtime-ktx", version.ref = "coroutinesKtx" }
```

And then you actually add them to the project by updating the `build.gradle.kts` file

```kotlin
dependencies {
    implementation(libs.retrofit)
    implementation(libs.retrofit.gson.converter)
    implementation(libs.gson)
    implementation(libs.okhttp.logging.interceptor)
    implementation(libs.glide)
    annotationProcessor(libs.glide.compiler)

    implementation(libs.androidx.viewmodel.ktx)
    implementation(libs.androidx.runtime.ktx)
    implementation(libs.kotlinx.coroutines.core)
    implementation(libs.kotlinx.coroutines.android)
}
```

And finally, the most important step that I almost always forget. To add the **INTERNET** permission in the `AndroidManifest.xml` file
```xml
<manifest>
    <uses-permission android:name="android.permission.INTERNET" />
    <uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" />
    <application>
        ...
    </application>
</manifest>
```

### Fetching data
Creating the network layer is pretty straightforward with Retrofit. You declare an interface of the API and let retrofit do the rest.

For our needs, the API consists of just a single `GET` call to fetch a `List` of `Photo`

```kotlin
data class Photo(
    val albumId: Int?,
    val id: Int?,
    val title: String?,
    val url: String?,
    val thumbnailUrl: String?
) : Serializable
```

```kotlin
interface API {
    @GET("photos")
    suspend fun photos(): Response<List<Photo>>
}
```

```kotlin
class NetworkService {
    companion object {
        fun api(): API {
            val loggingInterceptor = HttpLoggingInterceptor()
            loggingInterceptor.level = HttpLoggingInterceptor.Level.BODY

            val okHttpClient = OkHttpClient
                .Builder()
                .addInterceptor(loggingInterceptor)
                .build()

            val retrofit = Retrofit
                .Builder()
                .baseUrl("https://jsonplaceholder.typicode.com/")
                .addConverterFactory(GsonConverterFactory.create())
                .client(okHttpClient)
                .build()

            return retrofit.create(API::class.java)
        }
    }
}
```

Next piece of the puzzle is the `PhotoListViewModel` to map the network data into various UI states. Again pretty straightforward thanks to the beautiful magic of `ViewModelScope` and `StateFlow`

```kotlin
class PhotoListViewModel : ViewModel() {
    sealed class State {
        data object Loading : State()
        data class Error(val message: String) : State()
        data class Content(val photos: List<Photo>) : State()
    }

    private val _state = MutableStateFlow<State>(State.Loading)
    val state: StateFlow<State> = _state

    fun fetchPhotos() {
        viewModelScope.launch {
            _state.value = State.Loading
            try {
                val photos = NetworkService
                    .api()
                    .photos()
                    .body() ?: emptyList()
                _state.value = State.Content(photos)
            } catch (e: Exception) {
                val message = e.message ?: "Something went wrong"
                _state.value = State.Error(message)
            }
        }
    }
}
```

### UI
On the UI side we need 2 Activities, a `MainActivity` to render a grid of images and a `DetailActivity` to render the detailed image view.

The basic structure of the `MainActivity` is to simply observes viewmodel changes and update the UI.

```kotlin
class MainActivity : AppCompatActivity() {

    private val viewModel: PhotoListViewModel by viewModels()

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        enableEdgeToEdge()
        setContentView(R.layout.activity_main)
        ViewCompat.setOnApplyWindowInsetsListener(findViewById(R.id.main)) { v, insets ->
            val systemBars = insets.getInsets(WindowInsetsCompat.Type.systemBars())
            v.setPadding(systemBars.left, systemBars.top, systemBars.right, systemBars.bottom)
            insets
        }

        lifecycleScope.launch {
            repeatOnLifecycle(Lifecycle.State.STARTED) {
                viewModel.state.collect { state ->
                    when (state) {
                        is PhotoListViewModel.State.Loading -> showLoading()
                        is PhotoListViewModel.State.Error -> showError(state.message)
                        is PhotoListViewModel.State.Content -> showPhotoList(state.photos)
                    }
                }
            }
        }
        viewModel.fetchPhotos()
    }

    private fun showLoading() {
        Toast
            .makeText(this, "Loading ...", Toast.LENGTH_LONG)
            .show()
    }

    private fun showError(message: String) {
        Toast
            .makeText(this, message, Toast.LENGTH_LONG)
            .show()
    }

    private fun showPhotoList(photos: List<Photo>) {
        // TODO show grid view
    }
}
```

The interesting bit is with building a photo grid. This requires a `PhotoGridAdapter`

```kotlin
class PhotoGridAdapter(
    private val context: Context,
    private val photos: List<Photo>,
) :
    BaseAdapter() {

    override fun getCount(): Int {
        return photos.size
    }

    override fun getItem(position: Int): Any {
        return photos[position]
    }

    override fun getItemId(position: Int): Long {
        return position.toLong()
    }

    override fun getView(position: Int, convertView: View?, parent: ViewGroup?): View {
        val contentVw =
            convertView ?: LayoutInflater.from(context).inflate(R.layout.grid_item, parent, false)
        val imageVw: ImageView = contentVw.findViewById(R.id.imageVw)
        val textVw: TextView = contentVw.findViewById(R.id.textVw)

        val photo = photos[position]
        Glide
            .with(context)
            .load(photo.thumbnailUrl)
            .into(imageVw)
        textVw.text = photo.title ?: "-"

        return contentVw
    }
}
```

And then use this adapter to provide UI data for the `GridView`

```kotlin
class MainActivity : AppCompatActivity() {
    // ...

    private fun showPhotoList(photos: List<Photo>) {
        val gridVw = findViewById<GridView>(R.id.gridVw)
        val photoGridAdapter = PhotoGridAdapter(this, photos)
        gridVw.adapter = photoGridAdapter
    }
}
```

![Main Activity]({{site.url}}/assets/photoapp-android/mainactivity.png)

Next, when tapped on a photo tile we need to spawn `DetailActivity` and provide the selected `Photo` as argument.

```kotlin
class MainActivity : AppCompatActivity() {
    // ...

    private fun showPhotoList(photos: List<Photo>) {
        val gridVw = findViewById<GridView>(R.id.gridVw)

        gridVw.setOnItemClickListener { _, _, position, _ ->
            val photo = photos[position]
            startActivity(Intent(this, DetailsActivity::class.java).apply {
                putExtra("photo", photo)
            })
        }

        // ...
    }
}
```

And then finally the `DetailActivity`

```kotlin
class DetailsActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        enableEdgeToEdge()
        setContentView(R.layout.activity_details)
        ViewCompat.setOnApplyWindowInsetsListener(findViewById(R.id.main)) { v, insets ->
            val systemBars = insets.getInsets(WindowInsetsCompat.Type.systemBars())
            v.setPadding(systemBars.left, systemBars.top, systemBars.right, systemBars.bottom)
            insets
        }

        val photo = intent.getSerializableExtra("photo") as Photo

        val imageVw: ImageView = findViewById(R.id.imageVw)
        val textVw: TextView = findViewById(R.id.textVw)

        Glide
            .with(this)
            .load(photo.url)
            .thumbnail(
                Glide.with(this).load(photo.thumbnailUrl)
            )
            .into(imageVw)
        textVw.text = photo.title ?: "-"
    }
}
```

![Detail Activity]({{site.url}}/assets/photoapp-android/detailactivity.png) 

## Conclusion

The classic Android development feels very much like UIKit for iOS. Most the concepts are 1:1 mapped. Kotlin is always very `fun` to work with.
The one thing I find weird about Android is the lack of in-house libraries for such a  basic thing like Networking, which is the core building block for almost all apps these days. And then how various companies have rolled out their own solution for building the data-event pipeline. Like with `LiveData`, `StateFlow` and kotlin coroutines leaving the developers to figure out the glue to make it all work.

Before I discovered `Glide` I tried building my own `ImageRepository` to handle networking and data caching. And there too I felt the tools offered by Android were very primitive compared to say Apple's `URLSession`.

Also before falling back to retrofit, I was trying out ktor for networking. Personally I like ktor as it seems more flexible and more in line with other networking libraries I'm used to.

And finally the XML based layout feels very nice and robust. I like the code-design split view. 

The hardest part about Android development is dealing with the unknowns. Like I ran into a runtime crash when running the app on my device with is running on Android 11/SDK 30 while on the emulator I was using Android 13/SDK 34. But I guess that is just a day in life for a developer, you know the classic "Works on my machine" phenomenon.

The code from this experiment is sitting right next to all other experiments. [github.com/chunkyguy/PhotoApp/android](https://github.com/chunkyguy/PhotoApp/tree/master/android)

### References
- [https://developer.android.com/topic/libraries/architecture/coroutines](https://developer.android.com/topic/libraries/architecture/coroutines)
- [https://mubaraknative.medium.com/making-an-http-request-using-ktor-client-a87c2593ed25](https://mubaraknative.medium.com/making-an-http-request-using-ktor-client-a87c2593ed25)
- [https://developer.android.com/reference/android/widget/GridView](https://developer.android.com/reference/android/widget/GridView)
- [https://developer.android.com/courses/pathways/android-basics-compose-unit-5-pathway-1](https://developer.android.com/courses/pathways/android-basics-compose-unit-5-pathway-1)
- [https://ktor.io/docs/welcome.html](https://ktor.io/docs/welcome.html)
- [https://jsonplaceholder.typicode.com/photos](https://jsonplaceholder.typicode.com/photos)
- [https://www.youtube.com/watch?v=Yps0G5M4ilA](https://www.youtube.com/watch?v=Yps0G5M4ilA)
