---
layout: post
title:  "Hello Boden the x-platform framework"
date:   2019-06-01 13:19:00 +0200
categories: x-platform cpp
published: true
---

Today I wanna try [boden](https://www.boden.io/) to make a cross platform app. I saw the framework few days back and to be honest did not think too much about it. One of the main reasons being I'm already too deep into [Flutter](https://flutter.dev), and although not a day goes by when I wish Flutter exposed some other language than Dart but still I really believe that Flutter has everything we need to develop a real world cross platform mobile app.

So why do I wanna try Boden? First, I like trying new things. Second, I love C++. I think C++ is the one lanugage that can be used in many ways and people should stop writing new languages because almost everything you need from a programming language probably already exists in C++.

# Set up

The set up is actually pretty straightforward, especially if you read the README. I skipped over the part where it expects you to have `cmake` installed on your machine and that alone ruined 20 minutes of my excitement.

After creating a new app with `./boden new -n DemoBoden`, a Xcode project is generated which starts with clean app

![HelloBoden.png]({{ site.url }}/assets/hello_boden/1.png)

# Getting started

The first challenge I faced was that the window for some reasons was of size 320 x 480. By adding breakpoint at the point where the `UIWindow` was being created I saw that the `UIKit` is responding incorrect screen size:

```
(lldb) po [[UIScreen mainScreen] bounds]
(origin = (x = 0, y = 0), size = (width = 320, height = 480))
```

The fix is to use the Launch Screen storyboard, which for some reasons is not generated with the Xcode project.

![Launch.png]({{ site.url }}/assets/hello_boden/launch.png)


And add the entry in the `Info.plist`, otherwise the Launch storyboard will be simply ignored.

![Infoplist.png]({{ site.url }}/assets/hello_boden/infoplist.png)


If all goes well, you should see a nice full screen layout.

![Fullscreen.png]({{ site.url }}/assets/hello_boden/full_screen.png)

As confirmed by the `lldb`

```
(lldb) po [[UIScreen mainScreen] bounds]
(origin = (x = 0, y = 0), size = (width = 414, height = 896))
```

# Layout

Boden uses the [yoga](https://yogalayout.com) layout engine internally which is wrapped by other platform targeted libraries like YogaKit for Swift. It is really hard to find a documentation that actually uses yoga in the raw from, I mean in C++. But conceptually it is based on the CSS flexbox model, and it feels a bit familiar even though you might struggle with the exact syntax for a while.

For Boden, in most of examples I saw the typical usage was to pass the layout information as json string. If that is the case the probably the best approach would be to use the [yoga playground](https://yogalayout.com/playground) and use the generated code for ReactNative.

This is a sample code for a full screen `UINavigationController`

```cpp
    auto navigationView = std::make_shared<ui::NavigationView>();
    navigationView->stylesheet = FlexJsonStringify({
        "direction" : "Column",
        "flexGrow" : 1.0,
        "flexShrink" : 1.0,
        "alignItems" : "Stretch"
    });
```

# Table view

The concepts of Boden framework seems very similar to the `UIKit`, which is a good thing for me. So, for example, implementing a `UITableView` like layout comes with similar constructs. 

**Step 1:** Add a `ListView`

Implement a view controller that returns a `ListView` as a `std::shared_ptr<bdn::ui::View> ListViewController::view()`
```cpp
    std::shared_ptr<bdn::ui::ListView> _listView;
```

Which then is added a child to the root view, `NavigationView` in our case

```cpp
    navigationView->pushView(_listViewController->view(), "Photo List");
```

![List.png]({{ site.url }}/assets/hello_boden/list.png)


**Step 2:** Add a `Data Source`

To feed data to the `ListView` we need a data source.

```cpp
class PhotoListViewDataSource : public bdn::ui::ListViewDataSource {
};
```

And just like `UITableViewCell` we need to provide values for cells to be rendered.

```cpp
size_t PhotoListViewDataSource::numberOfRows()
{
    return 10;
}

std::shared_ptr<ui::View> PhotoListViewDataSource::viewForRowIndex(size_t rowIndex, std::shared_ptr<ui::View> reusableView)
{
    auto cell = std::make_shared<PhotoCell>();
    cell->build();
    return cell;
}

float PhotoListViewDataSource::heightForRowIndex(size_t rowIndex)
{
    return 100.0;
}
```

This is how a `PhotoCell` looks like:

```cpp
class PhotoCell : public bdn::ui::CoreLess<bdn::ui::ContainerView> {
    
public:
    void build();
};
```

```cpp
void PhotoCell::build()
{
    stylesheet = FlexJsonStringify({
        "direction" : "Row",
        "flexGrow" : 1.0,
        "justifyContent" : "SpaceBetween",
        "alignItems" : "FlexStart",
        "padding" : {"all" : 2.5}
    });
    
    auto label = std::make_shared<ui::Label>();
    label->stylesheet = FlexJsonStringify({
        "alignSelf": "Center",
        "flexGrow" : 1.0
    });
    label->text = "Ho ho";
    addChildView(label);
}
```

**Step 3:** Reload

Even for reloading the data, the API is a thin wrapper of the `UITableView.reloadData()`

```cpp
void ListViewController::reloadData()
{
    _listView->reloadData();
    _listView->refreshDone();
}
```

The cell reuse is pretty familiar

```
std::shared_ptr<ui::View> PhotoListViewDataSource::viewForRowIndex(size_t rowIndex, std::shared_ptr<ui::View> reusableView)
{
    std::shared_ptr<PhotoCell> cell;
    if (reusableView) {
        cell = std::dynamic_pointer_cast<PhotoCell>(reusableView);
    } else {
        cell = std::make_shared<PhotoCell>();
        cell->build();
    }
    
    cell->text = "Cell" + std::to_string(rowIndex);
    
    return cell;
}
```

![Cells.png]({{ site.url }}/assets/hello_boden/cells.png)

# Conclusion

So that is probably enough poking around Boden for today. I think I like the idea of Boden. It seems usable for an iOS developer who like C++ and get an Android app for free. 

The thing I really enjoyed about Boden compared to Flutter was that, with Boden I could simply step into the C++ framework and see how things are tied together. I suspect if that would still be the case with Android. But for iOS it was amazing, especially when dealing with crazy runtime issues. Also, being C++ you get a much strict compile time safety as with Swift but better.

I would to play more with networking, unit testing and probably try to run on Android before deciding if I would ship any app with this. But so far, I'm impressed.

As usual, all the code from today is in the PhotoApp project [github.com/chunkyguy/PhotoApp](https://github.com/chunkyguy/PhotoApp)