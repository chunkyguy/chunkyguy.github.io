---
layout: post
title:  "Experimenting with ComponentKit"
date:   2018-11-09 20:18:00 +0200
categories: ios
published: true
---

# Experimenting with ComponentKit

It's a nice Friday afternoon and I just realized that I've never used ComponentKit.

One of the reasons that got me really interested in the project is that, I've been looking around alternatives to UIKit. Not that I've anything against UIKit, but I seen how things out of hands with UIKit pretty soon. And ComponentKit was on my radar for quite some time.

Also, I'm a huge fan of C++, but so far the only ObjectiveC++ I've written was just to glue ObjectiveC with C++, like I would start with an `AppEngine.cpp` and forward all the call from the `AppDelegate` to the `AppEngine`. ComponentKit on the other hand, seems to be making full use of C++ and ObjectiveC in the same space. Which sounds weird enough for me to at least try this thing out.

## Building an App with ComponentKit

I will be building the PhotoApp that I try to build every now and then but this time with ComponentKit. What the PhotoApp does is that it starts with a loading screen while we fetch a list of photo and when we are done we render them on screen. Next, whenever a user taps on the cell, we display the details screen with the photo in more details and some text.

This is pretty simple app that touches many parts of most of the apps out there. Okay, let's go!

### Set up

The first step is to checkout the [repository from github](https://github.com/facebook/componentkit)
The recommended way is to use Carthage. It's been a long time since I've used Carthage, personally I like to manage all my dependencies manually, which is know is not a popular opinion these days. So, let's give Carthage a shot. 

Following the instructions from [https://github.com/Carthage/Carthage#quick-start](https://github.com/Carthage/Carthage#quick-start), this is now my `Cartfie` looks like:

```
github "facebook/ComponentKit" ~> 0.24
```

Now, I can run ` carthage update`... and it does not work

```
*** Fetching ComponentKit
*** Checking out ComponentKit at "0.24"
*** xcodebuild output can be found in /var/folders/wv/x1yzd_351bbbwgyps66xgytw0000gn/T/carthage-xcodebuild.NGYTMn.log
*** Building scheme "ComponentKit" in ComponentKit.xcodeproj
Build Failed
	Task failed with exit code 65:
	/usr/bin/xcrun xcodebuild -project /Users/sid/Work/Personal/PhotoApp/component-kit/Carthage/Checkouts/ComponentKit/ComponentKit.xcodeproj -scheme ComponentKit -configuration Release -derivedDataPath /Users/sid/Library/Caches/org.carthage.CarthageKit/DerivedData/9.2_9C40b/ComponentKit/0.24 -sdk iphoneos ONLY_ACTIVE_ARCH=NO CODE_SIGNING_REQUIRED=NO CODE_SIGN_IDENTITY= CARTHAGE=YES archive -archivePath /var/folders/wv/x1yzd_351bbbwgyps66xgytw0000gn/T/ComponentKit SKIP_INSTALL=YES GCC_INSTRUMENT_PROGRAM_FLOW_ARCS=NO CLANG_ENABLE_CODE_COVERAGE=NO STRIP_INSTALLED_PRODUCT=NO (launched in /Users/sid/Work/Personal/PhotoApp/component-kit/Carthage/Checkouts/ComponentKit)

This usually indicates that project itself failed to compile. Please check the xcodebuild log for more details: /var/folders/wv/x1yzd_351bbbwgyps66xgytw0000gn/T/carthage-xcodebuild.NGYTMn.log
```

Life is too short to read a log message. Time for plan B; Manually building the workspace.

The very first thing one notices after the setup is that how much time it takes for the ComponentKit to compile. I've been using Swift mostly at work for past 4 years now, and there compile time is unacceptable, and that is why whenever I'm working on something on a Friday evening, I pick something blazing fast, which ObjectiveC is, but ObjectiveC++ is not.

Whenever I'm working on an ObjC project, the very first thing I do is to toggle the `TREAT_WARNINGS_AS_ERRORS` flag to `YES`. And I wish every other project does the same, specially a project as widely used as ComponentKit.

### Getting started

Okay all code set up and ready to go, the first challenge is to figure out where to start. To be honest, I've no clue, so I'll looking at the sample WildeGuess project. Like, `WildeGuessCollectionViewController` I'm also planning to start out with a `UICollectionView`, so that is a good thing.

The most important piece of the system when working with a `UICollectionView` based layout is `CKCollectionViewDataSource`. This object is who you talk to whenever the data updates. 
Probably the first piece of information we want to inform the data source is the number of section we intend to display. Unlike traditional `UICollectionViewDataSource`, we do not provide a integer value, rather, we provide the change set.

For our case, we already know at the load time that there is going to be one section, which contains the loading view. We need to provide that information as a change set, as the data source starts with zero sections.

```
    CKDataSourceChangeset *initialData = [[[CKDataSourceChangesetBuilder transactionalComponentDataSourceChangeset]
                                           withInsertedSections:[NSIndexSet indexSetWithIndex:0]]build];
    [_dataSource applyChangeset:initialData mode:CKUpdateModeAsynchronous userInfo:nil];
```

Before I get my hands dirty with building components, I would also like to plug in the other thing data source might be interested in. And that is a event callback for whenever a cell starts or ends displaying is invoked we want to update our data source with this information. In our case this is the `willDisplayCell` and `didEndDisplayingCell` of `UICollectionViewDelegate`

```
- (void)collectionView:(UICollectionView *)collectionView
       willDisplayCell:(UICollectionViewCell *)cell
    forItemAtIndexPath:(NSIndexPath *)indexPath
{
    [_dataSource announceWillDisplayCell:cell];
}

- (void)collectionView:(UICollectionView *)collectionView
  didEndDisplayingCell:(UICollectionViewCell *)cell
    forItemAtIndexPath:(NSIndexPath *)indexPath
{
    [_dataSource announceDidEndDisplayingCell:cell];
}
```

With our basic data source set up, we can now focus on providing the first view. Usually when we start our layout with a `UICollectionView` the first thing that comes to mind it to have a bunch of `UICollectionViewCell` and register them and later whenever UIKit needs to render them on screen, we paint them. With ComponentKit things are different, we probably only need to create a `UICollectionView` and leave the rest to ComponentKit. The starting point of the entire component system seems to be this `CKComponentProvider` which is a protocol with a class method, and the context is provided as an argument. Neat.

```
+ (CKComponent *)componentForModel:(id<NSObject>)model context:(id<NSObject>)context;
```

So, my first attempt would be insert a `NSNull` and render an `PHLoadingComponent` for that model

```
    NSDictionary<NSIndexPath *, NSObject *> *loadingItem = @{[NSIndexPath indexPathForRow:0 inSection:0]: [NSNull null]};
    CKDataSourceChangeset *loadingData = [[[CKDataSourceChangesetBuilder transactionalComponentDataSourceChangeset] withInsertedItems:loadingItem] build];
    [_dataSource applyChangeset:loadingData mode:CKUpdateModeAsynchronous userInfo:nil];
```

### Creating first component

Creating a component does not seems as trivial if you have been writing apps for some time. I know I'm not creating a view from scratch because that is the job of ComponentKit, I probably have to only provide the data for the view. But what data? Let's start with a prebaked component `CKLabelComponent`

```
@interface PHLoadingComponent : CKCompositeComponent
+ (instancetype)newWithTintColor:(UIColor *)color;
@end

@implementation PHLoadingComponent

+ (instancetype)newWithTintColor:(UIColor *)color
{
    return [super newWithComponent:[CKLabelComponent
                                    newWithLabelAttributes:{
                                        .string = @"Loading"
                                    }
                                    viewAttributes:{
                                        {@selector(setBackgroundColor:), color},
                                        {@selector(setUserInteractionEnabled:), @NO}
                                    }
                                    size:{}]];
}

@end
```

![First component](https://i.imgur.com/IojqLjG.png)

Not bad! So, by simply providing the text data I can get a view. How about making it full screen and replacing the text with a `UIActivityIndicatorView`.

To be honest the [official documentation](https://componentkit.org/appledoc/html/index.html) is outdated as I can not find anything about everything, for example `CKFlexboxComponent` is not there. So the only way out is to actually check the source code, which is probably even better than looking up the docs.

From what I've figured out, the way one creates components for views is by providing the `Class` type and a bunch of selectors to be invoked. Imagine it this way, if you were to create a custom view with a `UIActivityIndicatorView`, you might do it like:

```
-(UIActivityIndicatorView *) makeSpinnerWithColor: (UIColor *)color
{
    UIActivityIndicatorView *spinner = [[UIActivityIndicatorView alloc] initWithFrame:[UIScreen mainScreen].bounds];
    [spinner setColor:color];
    [spinner setActivityIndicatorViewStyle:UIActivityIndicatorViewStyleWhiteLarge];
    return spinner
}
```

With ComponentKit, you might do it like:

```
-(CKComponent *) makeSpinnerWithColor: (UIColor *)color
{
    return [CKComponent
            newWithView:{
                [UIActivityIndicatorView class],
                {
                    {@selector(setColor:), color},
                    {@selector(setActivityIndicatorViewStyle:), UIActivityIndicatorViewStyleWhiteLarge}
                }
            }
            size: CKComponentSize::fromCGSize([UIScreen mainScreen].bounds.size)];
}

```

With that in place, we have a loading screen.

![Loading screen](https://i.imgur.com/5BY8CJf.png)

Next step is to get the activity indicator start and stop animating. Which is pretty simple

```
{@selector(startAnimating), nil}
```

### Rendering list of images

Next step, rendering a bunch of images. The first part is easy, making a network request and get a JSON payload containing a bunch of image URLs.

```
- (void)updateImages
{
    NSArray *images = _viewModel.list;
    NSUInteger count = images.count;
    NSMutableDictionary<NSIndexPath *, PAListItemViewModel *> *photoList = [NSMutableDictionary dictionaryWithCapacity:count];
    for (NSInteger idx = 0; idx < count; ++idx) {
        [photoList
         setObject:[_viewModel itemAtIndex:idx]
         forKey:[NSIndexPath indexPathForRow:idx inSection:0]];
    }

    CKDataSourceChangeset *loadingData = [[[CKDataSourceChangesetBuilder transactionalComponentDataSourceChangeset] withInsertedItems:photoList] build];
    [_dataSource applyChangeset:loadingData mode:CKUpdateModeAsynchronous userInfo:nil];
}
```

![Placeholder images](https://i.imgur.com/dJLybpG.png)

The tricky part is fetching the images. Normally we have to consider the fact that the user might be scrolling and the cell that started the network request could be out of view already, and we don't want to render wrong images on wrong cells. So, there has to be a owner of the network callback, from what I've read so far that job is usually for the ComponentController, but then I also something called `CKNetworkImageComponent`.

To make that happen, first step is provide a concrete implementation of `CKNetworkImageDownloading` which is pretty close to what `PANetworkService` is doing already. Here is a naive implementation

```
- (id)downloadImageWithURL:(NSURL *)URL
                    caller:(id)caller
             callbackQueue:(dispatch_queue_t)callbackQueue
     downloadProgressBlock:(void (^)(CGFloat progress))downloadProgressBlock
                completion:(void (^)(CGImageRef image, NSError *error))completion;
{
    [PANetworkService getPhotoWithURL:URL completion:^(UIImage * _Nullable image) {
        dispatch_async(callbackQueue, ^{
            if (image == nil) {
                completion(nil, [NSError
                                 errorWithDomain:@"com.wl.error.image"
                                 code:0
                                 userInfo:nil]);
            } else {
                completion(image.CGImage, nil);
            }
        });
    }];
    return [URL absoluteString];
}
```

And the second step is then to create the component

```
@implementation PHPhotoComponent
+ (instancetype)newWithPhoto:(PAPhoto *)photo
             imageDownloader:(PAImageDownloader *)imageDownloader;
{
    UIImage *placeholderImage = [PAImageLoader placeholderImage];
    CKComponent *imageComponent = [CKNetworkImageComponent
                                   newWithURL:[photo imageURLWithResolution:ImageResolutionThumbnail]
                                   imageDownloader:imageDownloader
                                   size:CKComponentSize::fromCGSize(placeholderImage.size)
                                   options:{ .defaultImage = placeholderImage }
                                   attributes:{}];
    return [super newWithComponent:imageComponent];
}
@end
```

![List of images](https://i.imgur.com/z9C42dB.png)

### User interactions

Next we want the react on whenever a user taps on the cell, and here is where `CKComponentTapGestureAttribute` comes into the picture. After playing around for a while, this is what seems to be working

```
@implementation PHPhotoComponent
+ (instancetype)newWithPhoto:(PAPhoto *)photo
             imageDownloader:(PAImageDownloader *)imageDownloader;
{
    UIImage *placeholderImage = [PAImageLoader placeholderImage];
    CKComponentSize size = CKComponentSize::fromCGSize(placeholderImage.size);
    CKComponent *imageComponent = [CKNetworkImageComponent
                                   newWithURL:[photo imageURLWithResolution:ImageResolutionThumbnail]
                                   imageDownloader:imageDownloader
                                   size:size
                                   options:{ .defaultImage = placeholderImage }
                                   attributes:{}];

    CKComponent *centerYComponent = [CKCenterLayoutComponent
                                     newWithCenteringOptions:CKCenterLayoutComponentCenteringY
                                     sizingOptions:CKCenterLayoutComponentSizingOptionMinimumY
                                     child:imageComponent
                                     size:{}];

    CKComponent *containerComponent = [CKFlexboxComponent
                                       newWithView:{
                                           [UIView class],
                                           {CKComponentTapGestureAttribute(@selector(onTap))}
                                       }
                                       size:{}
                                       style:{}
                                       children:{
                                           {centerYComponent}
                                       }];

    return [super newWithComponent:containerComponent];
}

- (void)onTap
{
    NSLog(@"show details");
}
@end
```
Whats left is hooking a listener to the event. From what I've read about the components is that they should not be set as delegate, as they are by design short lived immutables. So I'm imagining them as being disposed off after a draw cycle. So the question is how do we handle the touch event? To understand let's see what is `CKComponentTapGestureAttribute`?

```
CKComponentViewAttributeValue CKComponentTapGestureAttribute(CKAction<UIGestureRecognizer *> action);
```
So it a function that takes a `CKAction` and returns a `CKComponentViewAttributeValue`. The `CKAction` on the other hand is a simple wrapper around the target selector pair, maybe we can use that. Or even better if we could construct the `CKAction` and inject it from outside.

```
        CKAction<UIGestureRecognizer *>action = CKAction<UIGestureRecognizer *>::actionFromBlock(^(CKComponent *, UIGestureRecognizer *__strong) {
            NSLog(@"did tap");
        });
        
        return [PHPhotoComponent
                newWithPhoto:(PAPhoto *)model
                imageDownloader:(PAImageDownloader *)context
                action: action];
```

The tricky part is sending the invocation back to the view controller. This whole component creation lives in a class method so has no reference to the view controller. One way to fix this is by passing the view controller as context.

```
@interface PHRootComponentProvider : NSObject
@end

@implementation PHRootComponentProvider

+ (CKComponent *)componentForModel:(id<NSObject>)model context:(id<NSObject>)context;
{
    if ([model isKindOfClass:[NSNull class]]) {
        return [PHRootComponentProvider loadingComponent];
    } else if ([model isKindOfClass:[PAPhoto class]]) {
        return [PHRootComponentProvider
                componentForPhoto:(PAPhoto *)model
                sender:(PHViewController *)context];
    } else {
        NSAssert(NO, @"Should not happen");
        return nil;
    }
}

+ (CKComponent *)loadingComponent
{
    return [PHLoadingComponent newWithTintColor:[UIColor redColor]];
}

+ (CKComponent *)componentForPhoto:(PAPhoto *)photo sender:(PHViewController *)sender;
{
    CKAction<UIGestureRecognizer *>action = CKAction<UIGestureRecognizer *>::actionFromBlock(^(CKComponent *, UIGestureRecognizer *__strong) {
        [sender onTap:photo];
    });

    return [PHPhotoComponent
            newWithPhoto:photo
            imageDownloader:sender.imageDownloader
            action: action];

}

@end

```

And then in the view controller

```
    _dataSource = [[CKCollectionViewDataSource alloc]
                   initWithCollectionView:_collectionView
                   supplementaryViewDataSource:nil
                   configuration:[[CKDataSourceConfiguration alloc]
                                  initWithComponentProvider:[PHRootComponentProvider class]
                                  context:self
                                  sizeRange:[[CKComponentFlexibleSizeRangeProvider
                                              providerWithFlexibility:CKComponentSizeRangeFlexibleHeight]
                                             sizeRangeForBoundingSize:self.view.bounds.size]]];
```

I'm not sure if this is how the design is intended to be.

### Filling up the rest

The rest should not be hard to do. We need another view that shows the details of the photo. We can either build it using the ComponentKit, or could simply have static UIViewController. I like the fact that if we build the detail UI with ComponentKit we would not have to worry about making the UI scrollable in future if it turns out that the UI does not fit one of those small screens. Also, the fact that we can reuse the components.

Although I'm still not sure whether the better design is to have only a single `UIViewController` and only update the data source or have multiple view controllers. If I were to make a bet, I think `UIViewController` would be a better choice, as then the view caching could take some benefits or maybe not. Another question is, even if I were to pull the entire thing off with a single view controller, would it make sense to twist my arm around to get the default navigation push/pop animation by writing a fake animator class that reloads the `UICollectionView` as if it were a push/pop animation.

### Conclusion

So what do I think of ComponentKit. I think its a great proof of concept that one could easily write the one directional UI, but I would not force to write the entire app with ComponentKit, but maybe only the critical screens. 

I would definetely like to poke around to see how easy it is to build static screens with ComponentKit, like the ones where I know I don't need a `UICollectionView` and maybe also do some animation.

The entire code is available at [github.com/chunkyguy/PhotoApp](https://github.com/chunkyguy/PhotoApp) sitting along the pure swift and pure Objc implementations.