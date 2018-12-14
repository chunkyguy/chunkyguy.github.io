---
layout: post
title:  "Hello Flutter"
date:   2018-12-14 22:49:54 +0200
categories: flutter app cross-platform
published: true
---

Another weekend another tool. The tool for today is [Flutter](https://flutter.io/).

# Why Flutter?

I had known about Flutter for sometime now, but with the recent news of [Flutter 1.0](https://developers.googleblog.com/2018/12/flutter-10-googles-portable-ui-toolkit.html) being out, I got curious about it one more time. Then, recently at work [someone](https://github.com/LcTwisk) ignited my real interest by informing me that Flutter does its own rendering from scratch. I had to try it out.

In the past, I used to wonder a lot that why is that when it comes to cross platform mobile developement, nobody has written a UIKit alternative from scratch or maybe on top of some existing widget library and make it work on all sort of mobile devices. I mean how hard could it be? Given that there are people writing game engines using OpenGL ES, Metal or Vulkan all the time, but why not a UI framework. And that is exactly what Flutter does.

For me it was a love at first sight.

I read a few articles on the official source to get a hang of it. To know more about the vision of the project. I knew this is a product from Google, so I wanted to be sure they're not baised towards Android. When they're investing in this tool, what are they getting out of it? Like my initial impression of Xamarin was that Microsoft wants developers to write cross platform apps with Xamarin, and also make a Windows mobile app as a side effect. 

But, so far with Flutter this does not seems to be the case, even though they demo a lot with Material designs which looks a lot like Android UI, but then Flutter comes with a rendering engine of its own, and one can render whatever they feel like. There is even a UIKit like widget library which they call as Cupertino. 

Then a few hours into making my first Flutter app it occured to me that maybe with Flutter the thing Google is selling is the Dart programming language. I think Google likes to see Dart as a lightweight programming language which is good for quickly iterating and building things as fast as possible. A good candidate for making web and mobile apps. And there is nothing wrong with that. From little experience I had with Dart, I found nothing unlikable about it. At least they did not pick Javascript.

I know most of Game Engines ship a core engine written with a high performance system language like C or C++, and a high level language for writing the actual game, like Lua. Which is also what Flutter does, the core engine is written in C++ and Dart is used to then write the app.

# Set up

Whatever they say on the official instructions works out of the box. Although there are quite of bunch of things that have to be installed to get the thing running, but that is expected when installing a tool for cross platform development. The instructions come in 2 flavors, Android Studio and Visual Studio Code (again reflecting the fact that Google does not wants to be biased towards Android). I followed the instructions for Visual Studio Code and it just works.

# First run

Okay, even before I get to the part of running the app. I would like acknowledge the fact that how awesome is Visual Studio Code. This is the first time I'm actually using it for anything and I really like it. 

Being an iOS developer, I want to give iOS a try first. I created a new project and selected iPhone Simulator (which it magically discovered) And viola!

![Hello iOS](https://i.imgur.com/YnloK02.png)

Next, I lets give Android a run. But can't find a way to run the app on Android emulator. The drop down shown only the iOS simulator. The [faq section](https://dartcode.org/docs/quickly-switching-between-flutter-devices/) says this about switching the target device, but they do nothing:

```
1. Clicking on the currently selected device in the status bar
2. Executing the Flutter: Change Device command
3. Pressing your custom key binding for the Flutter: Change Device command
```

Then I discovered, if Flutter can not find an Android emulator, try running it manually by launching the `Android Studio > Tool > AVD Manager > Run emulator` and then the Android emulator is visible in the status bar.

![Status bar](https://i.imgur.com/vMvGnYu.png)

And now I think, if the main target device is Android, then maybe it is wise to set up the IDE as Android Studio. Anyways, after that part is taken care of, the app runs perfectly on Android emulator as well!

![Hello Android](https://i.imgur.com/shXV1oG.png)

# Make the real app

Now, since we have tested the tool and know for sure that the simulators are working perfectly, it's time to dive a bit deeper and make a real app.

As always, I'll try to make the Photo app that I try to make every now and then, but probably not today. There seems to be a lot of things that I've to learn before I get to that stage. The first stage would be render a list of things and navigate to a details screen when a cell is tapped. But no real networking or concurrency for this stage. I'll be following the google codelabs [part 1](https://codelabs.developers.google.com/codelabs/first-flutter-app-pt1/#0) and [part 2](https://codelabs.developers.google.com/codelabs/first-flutter-app-pt2/#0)

So, everything in Flutter is a `Widget`, which is again very common pattern with Game Engines to have everything as a `Node` and each node implements a `update()` and `render()` functions which are then called by the engine whenever required. In case of Flutter, there is only one function that has to be implemented which is the `build()` function. 

Another concept at the core of Flutter is the react like philosopy, where data flows in one direction so every node or `Widget` is more or less a stateless entity. Think of it this way, in every UI systems like `UIKit` or every game engines, at the core there is this tree like data structure, some call it the [scene graph](https://en.wikipedia.org/wiki/Scene_graph) other call it the [view hierarchy](https://developer.apple.com/library/archive/documentation/WindowsViews/Conceptual/ViewPG_iPhoneOS/WindowsandViews/WindowsandViews.html). And then there is is the infinite loop that traverses the tree and updates each node of the tree.

```
void main()
{
  SceneGraph *rootNode = buildSceneGraph();
  Clock c;
  Event e;
  while(!e.isQuit()) 
  {
    e = nextEvent();
    rootNode->update(c.deltaTime());
    c.tick();
  }
}
```
So, traditionally each node in the tree had to maintain its own state, which has a tendency to getting messier over time. The [react philosophy](https://reactjs.org/docs/thinking-in-react.html) is more like to not let direct access to nodes, but instead only provide the factory methods which takes in some model data and returns a new node. The tree is the internally managed by the system, and whenever the data changes the factory method would be called to build the new node. 

I'm personally a big fan of this philosophy. With this approach, one could freely optimize the tree manipulation as much as they like and on the other hand one could focus only on providing the factory methods. The only part that gets a bit messier is the animation which by definition can not be stateless.

Coming back to Flutter, the `build()` function is like the factory method that uses some model data and makes a `Widget`.

Another nice thing about Flutter is that it comes its own package manager. You just have to define the dependencies in the `pubspec.yaml` and then simply run `flutter package get`. Nice!

With all this basic understanding of the architecture, writing the first table view is pretty dead simple. I guess this piece of code is self explainatory:

```
class ListState extends State<Item> {
  final _list = <Item>[];
  Widget _buildWidget() {
    return new ListView.builder(
      padding: const EdgeInsets.all(16.0),
      itemBuilder: (BuildContext _context, int i) {
        if (_needsMoreItems(i)) {
          _data.addAll(_getMoreItems());
        }
        return _buildRow(_data[i]);
      }
    );
  }
}
  ```

![Table View](https://i.imgur.com/Z0qk8gP.png)

Since everything is a `Widget`, it is a piece of cake to add decorations to a `Widget`

```
Icon
(
  isSaved ? Icons.favorite : Icons.favorite_border,
  color: isSaved ? Colors.red : null,
)
```

![Add hearts](https://i.imgur.com/ffno2SU.png)

Another interesting fact is that Flutter builds a Xcode project internally which one can simply open and run. And if one looks close enough, one can find that there is just a single `UIView` that renders everything.

![Xcode](https://i.imgur.com/SBsNuK1.png)

One question I still had was how does the one maintains a state if required. For example, if say we want to react on a touch interaction we need to use the `StatefulWidget` which then comes with a `createState()` function that can provide access to some internal data store where we can read and write data like so:
```
onTap: () {
  setState((){
    if (isSaved) {
      _saved.remove(word);
    } else {
      _saved.add(word);
    }
  });
}
```

If you think that is weird, see how the screen transitions are done:

```
void onDetails() {
  Navigator.of(context)
      .push(MaterialPageRoute<void>(builder: (BuildContext context) {
    final Iterable<ListTile> tiles = _saved.map((WordPair word) {
      return ListTile(title: Text(word.asPascalCase, style: _biggerFont));
    });

    final List<Widget> children =
        ListTile.divideTiles(context: context, tiles: tiles).toList();

    return Scaffold(
        appBar: AppBar(title: const Text('Favorites')),
        body: ListView(children: children));
  }));
}
```

### Conclusion

If you can move out of your comfortable `UIKit` zone, I think Flutter is a pretty good tool. The performance so far was really nice (at least on the simulators). I can not say much anymore, I would have to complete my photo app to give any good opinion, but I can already say that so far with all the things I've tried with cross platform development, Flutter was the best.

The code can be found along with my other prototypes at [https://github.com/chunkyguy/PhotoApp](https://github.com/chunkyguy/PhotoApp). Thansk for reading. See you again later!