---
layout: post
title:  "Singleton vs Class Methods: ObjC"
date:   2012-02-11 23:28:54 +0530
categories: design-patterns
---

Every sometime you might get into a situation with your design pattern, that you might have this inevitable desire to use a Singleton class, or maybe you are coming from C++ background and are much used to it.

But, you shouldn’t forget that Objective-C is a different language, that here the classes are first class objects, so today I’m going to show a way to avoid using the singleton pattern and fill with the class methods instead.

The motivation is this: Every time you use the Singleton pattern the code looks a little uglier. For example, if your code has a Librarian class that is singleton, and at some point you’re using it like this:

``` objc
[[Librarian sharedInstance] openLibrary]
```

Looks good right, but wouldn’t it be nice if you could do something like:

``` objc
[Librarian openLibrary]
```

And not only will this make the code more sensible, it will solve one more purpose, hiding the allocation part of the Librarian. As with the former code you need to add some extra code to check that no one allocates a new instance of the Librarian, because that’s the whole purpose of making it Singleton, right?

OK, lets consider a classroom scenario, where there are a bunch of students and a single teacher. All the students want to communicate to the single teacher, so we might consider it as a perfect place to use the singleton pattern.

For comparison purpose, lets consider there are two such classrooms in entirely different worlds. The fist one uses the same old Singleton pattern, where we have this teacher names MrHyde, as you might have guessed the better world has a similar teacher names DrJekyll.

And, implementation wise both the teachers should be equally capable of doing things, the same way, our only objective is to change the way we refer them, or pass messages in Objective-C way.

So, in each classroom students are just saying hello to the teacher, and the teacher is simply counting the number of hello he received, maybe just for roll call.

Here’s MrHyde’s classroom scenario:

``` objc
//MrHyde.h
@interface MrHyde : NSObject{
    int count_;
}
+(MrHyde *)shared;
-(void)hello;
 
@end
 
//MrHyde.m
#import "MrHyde.h"
 
static MrHyde *shared_ = nil;
 
@implementation MrHyde
 
+(MrHyde *)shared{
    if(!shared_){
        shared_ = [[self alloc] init];
    }
    return shared_;
}
 
-(id)init{
    self = [super init];
    if(self){
        count_ = 0;
    }
   return self;
}
 
-(void)hello{
    count_++;
    NSLog(@"Hello %d times today",count_);
}
 
@end
```

And here is how students are interacting:
``` objc
for(int student = 0; student < 10; student++){
    [[MrHyde shared] hello];
}

```

Looks familiar right, now take a look at Dr Jekyll’s classroom:
``` objc
//DrJekyll.h
@interface DrJekyll : NSObject{
}
 
+(void)hello;
 
@end
 
//DrJekyll.m
static MrHyde *shared_ = nil;
 
@implementation DrJekyll
 
+(void)initialize{
    NSLog(@"creating Hidden Mr Hyde");
    shared_ = [[MrHyde alloc] init];
}
 
+(void)hello{
    [shared_ hello];
}
```

In case you are wondering what’s that +(void)initialize doing there? The Documentation says:

The runtime sends initialize to each class in a program exactly one time just before the class, or any class that inherits from it, is sent its first message from within the program.


That you can confirm by looking for log creating Hidden Mr Hyde getting printed only once!

And, this is how students interact with Dr Jekyll:
``` objc
for(int student = 0; student < 10; student++){
    [DrJekyll hello];
}
```

Looks much better than Mr Hyde’s scenario right?

One more thing to notice here is you are not passing any messages to the instance of DrJekyll, so creating any instance of DrJekyll is waste of code, that solves our problem of Singleton class duplicate instance creation.

But, you can say that internally it’s just using MrHyde for all the deeds and what if someone accidentally creates a new MrHyde and starts interacting with it?

Yes, frankly that’s a scary situation, so we can improve our code by creating a private MrHyde, that is hidden inside DrJekyll, and is invisible to entire world.

Take a look at this new improved DrJekyll implementation:
``` objc
//DrJekyll.m
@interface HiddenMrHyde : NSObject {
    int count_;
}
 
-(void)hello;
 
@end
 
@implementation HiddenMrHyde
 
-(id)init{
    self = [super init];
    if(self){
        count_ = 0;
    }
    return self;
}
 
-(void)hello{
    count_++;
    NSLog(@"Hello %d times today",count_);
}
 
@end
 
static HiddenMrHyde *mrHyde_ = nil;
 
@implementation DrJekyll
 
+(void)initialize{
    NSLog(@"creating Hidden Mr Hyde");
    mrHyde_ = [[HiddenMrHyde alloc] init];
}
 
+(void)hello{
    [mrHyde_ hello];
}
 
@end
```

All of the code remains the same!

Hope this encourages you to write Singletons in a way that makes more sense, Objective-C way :)