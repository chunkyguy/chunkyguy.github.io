---
layout: post
title:  "iOS: Moving in and out of NSLogs"
date:   2011-01-20 04:15:00 +0530
categories: logging ios
---

iOS: Moving in and out of NSLogs
----------------------------------------

So, if you are a iPhone developer like me you must be using a lot of
`NSLog` in your code, and at times you must have really wished for a way
to turn them all `ON` and `OFF` from single place.\
Here is a easy way I do it.\
<span id="more-134"></span>\
Step 1: Create a `SJLogger.h` and `SJLogger.m`. In the `SJLogger.h` add
this macro:

``` {.brush: .objc; .title: .; .notranslate title=""}
/*
 1: ON
 0: OFF
 */
#define LOG 1
```

Step 2: Put this code in `SJLogger.m` file:

``` {.brush: .objc; .title: .; .notranslate title=""}
void SJLog(NSString *format,...)
{
    if(LOG)
    {
        va_list args;
        va_start(args,format);
        NSLogv(format, args);
        va_end(args);
    }
}
```

Step 3: Now replace all the `NSLogs` with `SJLog`, like:

``` {.brush: .objc; .title: .; .notranslate title=""}
SJLog(@"Simple log");
NSArray *arr = [NSArray arrayWithObjects:@"This",@"is",@"a",@"array",@"log",nil];
SJLog(@"%@",arr);
NSString *str = @"This is a string log";
SJLog(str);
SJLog(@"This is formatted %@",@"log");
```

Quick Tip: Use **Cmd + Shift + F** and replace all occurrence of `NSLog`
with `SJLog`

Step 4: Now, anytime you wanna turn logs `ON/OFF`, just set the
`DEBUG MACRO` in `SJLogger.h` accordingly.

**Get the [SJLogger.tar](http://code.127.0.0.1/sjlogger.tar) from
here.**\
Happy Logging
![:)](http://127.0.0.1/rants/wp-includes/images/smilies/icon_smile.gif){.wp-smiley}

> FEW MONTHS LATER…

The original code was great, but as I’d already mentioned that I’m going
to upgrade this code to be more meaningful(read comments). One big
problem with our code is it doesn’t prints **filename** or **line
number**, so it is hard at time if you just want to comment the log at
one place and not all logs entirely.\
Then, today I ran into [this awesome article by
AgentM](http://borkware.com/rants/agentm/mlog/), and it made me update
my code as:

``` {.brush: .objc; .title: .; .notranslate title=""}
//SJLogger.h
#define SJLog(s,...) [SJLogger logFile:__FILE__ lineNumber:__LINE__ format:(s),##__VA_ARGS__]

/*
 1: ON
 0: OFF
 */
#define LOG 1

@interface SJLogger : NSObject {

}

+(void)logFile:(char*)sourceFile lineNumber:(int)lineNumber format:(NSString*)format, ...;
@end
```

``` {.brush: .objc; .title: .; .notranslate title=""}
@implementation SJLogger
+(void)logFile:(char*)sourceFile lineNumber:(int)lineNumber format:(NSString*)format, ...
{
    va_list ap;
    NSString *print,*file;
    if(!LOG)
        return;
    va_start(ap,format);
    file=[[NSString alloc] initWithBytes:sourceFile length:strlen(sourceFile) encoding:NSUTF8StringEncoding];
    print=[[NSString alloc] initWithFormat:format arguments:ap];
    va_end(ap);
    //NSLog handles synchronization issues
    NSLog(@"%s:%d %@",[[file lastPathComponent] UTF8String],lineNumber,print);
    [print release];
    [file release];
}
@end
```

What I like in this approach is the fact that it carefully uses
**NSLog** at the core, so we should get all the current and future
benefits of **NSLog** (though I highly doubt the future part).

I haven’t used this in my production code, but I’m planning to update
the files, fingers crossed
![:)](http://127.0.0.1/rants/wp-includes/images/smilies/icon_smile.gif){.wp-smiley}

</div>

<div class="entry-utility">

This entry was posted in
[\_dev](http://127.0.0.1/rants/?cat=3 "View all posts in _dev"),
[\_iOS](http://127.0.0.1/rants/?cat=5 "View all posts in _iOS") and
tagged [iOS](http://127.0.0.1/rants/?tag=ios),
[mac](http://127.0.0.1/rants/?tag=mac),
[NSLog](http://127.0.0.1/rants/?tag=nslog). Bookmark the
[permalink](http://127.0.0.1/rants/?p=134 "Permalink to iOS: Moving in and out of NSLogs.").

</div>

</div>

<div id="nav-below" class="navigation">

<div class="nav-previous">

[<span class="meta-nav">←</span> My entry for coke mobile app
challenge](http://127.0.0.1/rants/?p=123)

</div>

<div class="nav-next">

[Dynamic Programming: Take I <span
class="meta-nav">→</span>](http://127.0.0.1/rants/?p=149)

</div>

</div>

<div id="disqus_thread">

<div id="dsq-content">

<div id="dsq-comment-header-17" class="dsq-comment-header">

<span id="dsq-author-user-17">Laurent Tarral</span>

</div>

<div id="dsq-comment-body-17" class="dsq-comment-body">

<div id="dsq-comment-message-17" class="dsq-comment-message">

Thanks! And do you \#import the SJLogger.h (I get some warnings but it
works) or the SJLogger.m (I get an error) in the classes which use
SJLog?

</div>

</div>

-   <div id="dsq-comment-18">

    </div>

    <div id="dsq-comment-header-18" class="dsq-comment-header">

    http://127.0.0.1 <span id="dsq-author-user-18">sid</span>

    </div>

    <div id="dsq-comment-body-18" class="dsq-comment-body">

    <div id="dsq-comment-message-18" class="dsq-comment-message">

    I import SJLogger.h, yes there’s a warning and I’m planning on to
    upgrade the code to add conditional level flags
    like:DEADLY, WARNING, INFO,.. something like the Java’s Logger
    class. \
    But for now its serving the purpose
    ![:)](http://127.0.0.1/rants/wp-includes/images/smilies/icon_smile.gif){.wp-smiley}

    </div>

    </div>

<div id="dsq-comment-header-60" class="dsq-comment-header">

<span id="dsq-author-user-60">Laurent Tarral</span>

</div>

<div id="dsq-comment-body-60" class="dsq-comment-body">

<div id="dsq-comment-message-60" class="dsq-comment-message">

Thanks! And do you \#import the SJLogger.h (I get some warnings but it
works) or the SJLogger.m (I get an error) in the classes which use
SJLog?

</div>

</div>

-   <div id="dsq-comment-61">

    </div>

    <div id="dsq-comment-header-61" class="dsq-comment-header">

    http://127.0.0.1 <span id="dsq-author-user-61">chunkyguy</span>

    </div>

    <div id="dsq-comment-body-61" class="dsq-comment-body">

    <div id="dsq-comment-message-61" class="dsq-comment-message">

    I import SJLogger.h, yes there’s a warning and I’m planning on to
    upgrade the code to add conditional level flags
    like:DEADLY, WARNING, INFO,.. something like the Java’s Logger
    class. \
    But for now its serving the purpose
    ![:)](http://127.0.0.1/rants/wp-includes/images/smilies/icon_smile.gif){.wp-smiley}
