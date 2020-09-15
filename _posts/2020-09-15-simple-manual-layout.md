---
layout: post
title:  "Building UI without AutoLayout"
date:   2020-09-15 08:00:00 +0200
categories: objc ui
published: true
---

I love [Auto Layout](https://developer.apple.com/library/archive/documentation/UserExperience/Conceptual/AutolayoutPG/index.html). It helps a lot when designing complex UI. But there are times when the UI is very simple and Auto Layout might feel a bit overkill, while other times the UI might be a bit too complex and Auto Layout actually starts affecting the app performance. Before auto layout there was another technique to creating UI, it's called *Springs and Struts* (also known as *Manual Layout* to be in contrast with Auto Layout). I like Manual Layout a lot as well for its simplicity. Like with every other tool, there are trade-offs when selecting the best tool for the job, and it also applies when selecting Auto Layout vs Manual Layout.

The good thing is that, the Auto Layout has not been designed as an alternative to Manual Layout, rather more like an complement. Where instead of us having to calculate the `frame`, we start with a `CGRectZero` and let the Auto Layout fill in the `frame` value later. Most of the time it's wonderful and doesn't impact our flow. Other times we might have to wait for the layout pass run to read back the calculated `frame` values

```c
  // let Auto layout calculate the frame values
  dispatch_async(dispatch_get_main_queue(), ^{
    // start using the frame values for something else.
  });
```

I often wish if the Auto Layout were not that tightly coupled with `UIKit`. In a sense, if I could just run Auto Layout without a layout pass. This inspired me to take another take of building UIs without Auto Layout with something I'd like to call as *Simple Manual Layout*.

## Inspiration

The inspiration is from how `UIBarButtonItem` works with `UIToolbar` or `UINavigationBar`. If we wanted to build a UI like

![img]({{ site.url }}/assets/simple-manual-layout/toolbar.png)

We would create a `UIToolBar` and add a bunch of `UIBarButtonItem`

```objc
UIToolbar *toolbar = [[UIToolbar alloc] initWithFrame:toolbarFrame];
UIBarButtonItem *playButton = [[UIBarButtonItem alloc]
                               initWithBarButtonSystemItem:UIBarButtonSystemItemPlay
                               target:self
                               action:@selector(playVideo)];
UIBarButtonItem *pauseButton = [[UIBarButtonItem alloc]
                                initWithBarButtonSystemItem:UIBarButtonSystemItemPause
                                target:self
                                action:@selector(pauseVideo)];
UIBarButtonItem *rewindButton = [[UIBarButtonItem alloc]
                                 initWithBarButtonSystemItem:UIBarButtonSystemItemRewind
                                 target:self
                                 action:@selector(rewindVideo)];
UIBarButtonItem *forwardButton = [[UIBarButtonItem alloc]
                                  initWithBarButtonSystemItem:UIBarButtonSystemItemFastForward
                                  target:self
                                  action:@selector(forwardVideo)];
UIBarButtonItem *spaceButton = [[UIBarButtonItem alloc]
                                initWithBarButtonSystemItem:UIBarButtonSystemItemFlexibleSpace
                                target:nil
                                action:nil];
[toolbar setItems:[NSArray arrayWithObjects:
                   spaceButton,
                   rewindButton,
                   spaceButton,
                   playButton,
                   spaceButton,
                   pauseButton,
                   spaceButton,
                   forwardButton,
                   spaceButton,
                   nil]];
```

The interesting element here is `UIBarButtonSystemItemFlexibleSpace`. Which is [documented as](https://developer.apple.com/documentation/uikit/uibarbuttonsystemitem/uibarbuttonsystemitemflexiblespace) _"Blank space to add between other items. The space is distributed equally between the other items."_. Similarly, there's another one called `UIBarButtonSystemItemFixedSpace` which is [documented as](https://developer.apple.com/documentation/uikit/uibarbuttonsystemitem/uibarbuttonsystemitemfixedspace) _"Blank space to add between other items. Only the width property is used when this value is set."_.

I think this approach could be used to build a layout engine which is very simple in terms of mental model but can be used to build as sophisticated layouts as we'd want.

## Simple Manual Layout

With that design in mind we can build out layout engine. If there is a class `SLELayoutItem` which is a placeholder for a `UIView` and another class `SLELayout` that takes in one or more of these `SLELayoutItem` and immediately calculates the `frame` of the every `SLELayoutItem`. Then we can use the calculated `frame` value when constructing our `UIView` objects.

So to create a full screen subview we should be able to create as:

![img]({{ site.url }}/assets/simple-manual-layout/testFullScreenLayout.png)

```objc
SLELayout *layout = [SLELayout layoutWithParentBounds:_rootFrame
                                            direction:SLELayoutDirectionColumn];
SLELayoutItem *mainItem = [SLELayoutItem flexItem];
[layout addItem:mainItem];

UIView *redView = SLECreateView(mainItem.frame, [UIColor redColor]);
[_rootView addSubview:redView];
```

```objc
UIView *SLECreateView(CGRect frame, UIColor *color)
{
  UIView *view = [[UIView alloc] initWithFrame:frame];
  view.backgroundColor = color;
  return [view autorelease];
}
```

And a 2 subview layout, where the top is flexible and bottom is fixed

![img]({{ site.url }}/assets/simple-manual-layout/testBottomFixLayout.png)

```objc
  SLELayout *layout = [SLELayout layoutWithParentBounds:_rootFrame
                                              direction:SLELayoutDirectionColumn];
  [layout addItem:[SLELayoutItem flexItem]];
  [layout addItem:[SLELayoutItem itemWithHeight:200]];

  CGRect topFrame = [layout frameAtIndex:0];
  CGRect bottomFrame = [layout frameAtIndex:1];

  [_rootView addSubview:SLECreateView(topFrame, [UIColor redColor])];
  [_rootView addSubview:SLECreateView(bottomFrame, [UIColor blueColor])];
```

A more interesting layout would be where we have a column that contains a row.

![img]({{ site.url }}/assets/simple-manual-layout/testInnerViewLayout.png)

```objc
  CGFloat contentHeight = 200.f;
  SLELayout *mainLayout = [SLELayout layoutWithParentBounds:_rootFrame
                                                  direction:SLELayoutDirectionColumn];
  [mainLayout addItem:[SLELayoutItem flexItem]];
  [mainLayout addItem:[SLELayoutItem itemWithHeight:44]];
  [mainLayout addItem:[SLELayoutItem itemWithHeight:contentHeight]];

  CGRect headerFrame = [mainLayout frameAtIndex:0];
  CGRect toolbarFrame = [mainLayout frameAtIndex:1];
  CGRect contentFrame = [mainLayout frameAtIndex:2];

  SLELayout *contentLayout = [SLELayout layoutWithParentBounds:contentFrame
                                                   direction:SLELayoutDirectionRow];
  [contentLayout addItem:[SLELayoutItem flexItem]];
  [contentLayout addItem:[SLELayoutItem itemWithWidth:contentHeight]];
  [contentLayout addItem:[SLELayoutItem flexItem]];
  [contentLayout addItem:[SLELayoutItem itemWithWidth:contentHeight]];
  [contentLayout addItem:[SLELayoutItem flexItem]];

  CGRect content1Frame = [contentLayout frameAtIndex:1];
  CGRect content2Frame = [contentLayout frameAtIndex:3];

  [_rootView addSubview:SLECreateView(headerFrame, [UIColor redColor])];
  [_rootView addSubview:SLECreateView(toolbarFrame, [UIColor blueColor])];
  UIView *contentView = SLECreateView(contentFrame, [UIColor yellowColor]);
  [_rootView addSubview:contentView];

  [contentView addSubview:SLECreateView(content1Frame, [UIColor cyanColor])];
  [contentView addSubview:SLECreateView(content2Frame, [UIColor magentaColor])];
```

# Implementation details

The implementation of this layout engine turns out to be not as sophisticated. If we provide a `SLELayoutItem` which can have some properties fixed and others flexible. 

```objc
SLELayoutItem.h
@interface SLELayoutItem : NSObject

// no values fixed
+ (instancetype)flexItem;

// partially fixed
+ (instancetype)itemWithWidth:(CGFloat)width;
+ (instancetype)itemWithHeight:(CGFloat)height;

// total fixed
+ (instancetype)itemWithSize:(CGSize)size;

// would be filled by the layout engine
@property (nonatomic, readonly) CGRect frame;

@end
```

So we can mark any flexible value as `kSLELayoutValueUndefined` which is `-1` in our case.

```objc
// SLELayoutItem.m
@implementation SLELayoutItem
- (instancetype)initWithSize:(CGSize)size
{
  self = [super init];
  if (self) {
    // requested frame
    _originalFrame = (CGRect) {
      .origin = { .x = kSLELayoutValueUndefined, .y = kSLELayoutValueUndefined },
      .size = { .width = size.width, size.height }
    };
    // will be updated later
    _finalFrame = CGRectZero;
  }
  return self;
}

// called by layout engine
- (void)setOrigin:(CGPoint)origin
{
  _finalFrame.origin = origin;
}

- (void)setSize:(CGSize)size
{
  _finalFrame.size = size;
}

@end
```

And we can have an internal _setter_ interface only visible to `SLELayout`

```objc
// SLELayoutItem+Internal.h
@interface SLELayoutItem ()

- (void)setOrigin:(CGPoint)origin;
- (void)setSize:(CGSize)size;

@property (nonatomic, readonly) CGRect originalFrame;

@end
```

Next, within `SLELayout` we have an mutable array that contains `SLELayoutItem`. And whenever a new item is added we recalculate the frames per item.

```objc
// SLELayout.m
@implementation SLELayout

// ... 

- (void)addItem:(SLELayoutItem *)item
{
  [_items addObject:item];
  [self updateFrames];
}
@end
```

If we calculate only for one direction, say vertical. The `updateFrames` might look something like:

```objc
// SLELayout.m
@implementation SLELayout

// ... 

- (void)updateFrames
{
  // calculate total fixed height
  CGFloat fixHeight = 0;
  NSInteger flexItems = 0;
  for (SLELayoutItem *item in _items) {
    CGFloat itemHeight = item.originalFrame.size.height;
    if (itemHeight == kSLELayoutValueUndefined) {
      flexItems += 1;
    } else {
      fixHeight += itemHeight;
    }
  }

  // calculate height per flex item
  CGFloat flexHeight = _parentSize.height - fixHeight;
  CGFloat flexItemHeight = flexHeight / (CGFloat)flexItems;

  // update final frames per item
  CGFloat offsetY = 0.f;
  for (SLELayoutItem *item in _items) {
    CGSize itemSize = item.originalFrame.size;
    CGFloat itemHeight = (itemSize.height == kSLELayoutValueUndefined)
      ? flexItemHeight
      : itemSize.height;
    itemSize = (CGSize) { .width = _parentSize.width, .height = itemHeight };

    [item setOrigin:CGPointMake(0.f, offsetY)];
    [item setSize:itemSize];

    offsetY += itemSize.height;
  }
}
@end
```

And similar calculations for width.

And now it doesn't seem hard to imagine to support alignment for sub views (currently they are all set `0.0f` or all aligned to start) with something like:

```objc
typedef NS_ENUM(NSUInteger, SLELayoutAlignment) {
  SLELayoutAlignmentStart,
  SLELayoutAlignmentCenter,
  SLELayoutAlignmentEnd
};
```
 
If I actually start using this code in real life, I might start supporting it. The code for the `Simple Layout Engine` is available at [github.com/chunkyguy/SimpleLayoutEngine](https://github.com/chunkyguy/SimpleLayoutEngine)