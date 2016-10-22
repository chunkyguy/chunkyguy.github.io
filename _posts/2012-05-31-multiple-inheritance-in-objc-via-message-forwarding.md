---
layout: post
title:  "Multiple inheritance in ObjC via Message Forwarding"
date:   2012-05-31 23:28:54 +0530
categories: design-patterns
---

Multiple inheritance in ObjC via Message Forwarding
=====================================================

I don’t know how many people know that a sort of very intelligent Multiple inheritance is available with Objective-C.

What is multiple inheritance?

Multiple inheritance is a controversial concept where a class can derive from one or more classes. It is widely appreciated by lazy programmers who don’t like to code same thing twice, don’t even care to copy paste as they reason that it would increase the code maintenance overhead. Also, they add that while replicating some real world scenario into code, multiple inheritance is very helpful.

Case Study

Since the discussion is about ObjC, let’s consider a scenario. We’re working on a universal project, (targeting both iPhone and iPad with single code)

For iPad we have a class called iPadStylingTableViewController, this inherits from UITableViewController, and every controller on iPad inherits from iPadStylingTableViewController. Let’s say iPadStylingTableViewController adds a cool looking background image.

1
2
3
4
//iPadStylingTableViewController.h
@interface iPadStylingTableViewController : UITableViewController
-(void)awesomeStylingMethod;
@end
Now, let’s suppose we’ve a settings screen called SettingsTableViewController, this is how it looks:

1
2
3
4
//SettingsTableViewController.h
@interface SettingsTableViewController : iPadStylingTableViewController
-(void)showSettings;
@end
No problem, right?

But, then for iPhone in the settings screen you don’t need all the effects coming from iPadStylingTableViewController, you just want to keep it the default way, so this is how your code should look like

1
2
3
4
//SettingsTableViewController.h
@interface SettingsTableViewController : UITableViewController
-(void)showSettings;
@end
You must have noticed we’ve two different interface declarations for SettingsTableViewController class.



One ugly solution I’ve seen is that you create two classes SettingsTableViewController_iPad and SettingsTableViewController_iPhone, and it somehow works, but you need to maintain the same code two places.



If Java has been in your head for too long, you might do something creating an @protocol like:

1
2
3
4
5
6
7
8
9
10
11
12
13
//SettingsCommonInterface.h
@protocol SettingsCommonInterface
@required
-(void)showSettings;
@end
 
//SettingsTableViewController_iPad.h
@interface SettingsTableViewController_iPad : iPadStylingTableViewController <SettingsCommonInterface>
@end
 
//SettingsTableViewController_iPhone.h
@interface SettingsTableViewController_iPhone : UITableViewController <SettingsCommonInterface>
@end


But, still is doesn’t hides the fact that you still need to maintain two copies of  method showSettings.

But, here’s how you can cope with code duplication in Objective C using message forwarding.

You can delegate all the load from SettingsTableViewController_iPad to SettingsTableViewController_iPhone by creating anSettingsTableViewController_iPhone object

1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
//SettingsTableViewController_iPad.h
@interface SettingsTableViewController_iPad : iPadStylingTableViewController <SettingsCommonInterface>{
    SettingsTableViewController_iPhone *delegateController_;
}
@end
 
//SettingsTableViewController_iPad.m
@implementation
-(void)showSettings{
    [delegateController_ showSettings];
}
@end
 
//SettingsTableViewController_iPhone.h
@interface SettingsTableViewController_iPhone : UITableViewController <SettingsCommonInterface>
@end
 
//SettingsTableViewController_iPhone.m
@implementation
-(void)showSettings{
    //implementation here...
}
@end


This will solve the code duplicacy issue, as now we just have the real implementation of showSettings method in single class.
But, what if some other day you want to extend the functionality for the settings screen, let's say you want to add a new -(void)toggleAds method.
You would add the implementation to SettingsTableViewController_iPhone:

1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
//SettingsCommonInterface.h
@protocol SettingsCommonInterface
@required
-(void)showSettings;
-(void)toggleAds;
@end
 
//SettingsTableViewController_iPad.m
@implementation
-(void)showSettings{
    [delegateController_ showSettings];
}
 
-(void)toggleAds{
    [delegateController_ showSettings];
}
@end
 
//SettingsTableViewController_iPhone.m
@implementation
-(void)showSettings{
    //implementation here...
}
 
-(void)toggleAds{
    //implementation here...
}
@end
Message Forwarding in action

So, after a while you will realize that still upgrading the code is a real pain in the ass, as for each time you add some new functionality you still have to upgrade at more than one place. Now here is where we bring the awesome Message Forwarding into play.
Objective C has one awesome NSInvocation class that handles message forwarding very smartly.
All you need is to do is to implement these three methods, and it automatically redirects the messages to the classes that are capable to handle it, without you writing any extra lines :)

1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
//SettingsTableViewController_iPad.m
-(void)forwardInvocation:(NSInvocation *)anInvocation{
    if([delegateController_ respondsToSelector:[anInvocation selector]]){
        [anInvocation invokeWithTarget:delegateController_];
    }else{
        [super forwardInvocation:anInvocation];
    }
}
 
- (NSMethodSignature*)methodSignatureForSelector:(SEL)selector
{
    NSMethodSignature* signature = [super methodSignatureForSelector:selector];
    if (!signature) {
        signature = [delegateController_ methodSignatureForSelector:selector];
    }
    return signature;
}
 
-(BOOL)conformsToProtocol:(Protocol *)aProtocol{
    if([super conformsToProtocol:aProtocol] || [delegateController_ conformsToProtocol:aProtocol]){
        return YES;
    }
    return NO;
}
-(void)forwardInvocation:(NSInvocation *)anInvocation method is called everytime a message to passed to an object, we simply extended it’s implementation by checking that if our delegate controller, that is in this case the SettingsTableViewController_iPhone, implements the method being called in the message recieved. If yes, let it handle the message, otherwise do what it was originally meant to do (most probably crash the app).
The - (NSMethodSignature*)methodSignatureForSelector:(SEL)selector method matches the signature of the selector is also internally called each time to resolve the selector.

For further info check out the Apple’s documentation, it’s a great read.