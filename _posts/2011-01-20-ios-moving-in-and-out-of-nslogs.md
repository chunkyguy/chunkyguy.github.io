---
layout: post
title:  "iOS: Moving in and out of NSLogs"
date:   2011-01-20 04:15:00 +0530
categories: logging ios
---

So, if you are a iPhone developer like me you must be using a lot of
`NSLog` in your code, and at times you must have really wished for a way
to turn them all `ON` and `OFF` from single place.\
Here is a easy way I do it.\
<span id="more-134"></span>\
Step 1: Create a `SJLogger.h` and `SJLogger.m`. In the `SJLogger.h` add
this macro:

```
/*
 1: ON
 0: OFF
 */
#define LOG 1
```

Step 2: Put this code in `SJLogger.m` file:

``` objc
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

``` objc
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

Happy Logging! :)

> FEW MONTHS LATER…

The original code was great, but as I’d already mentioned that I’m going
to upgrade this code to be more meaningful(read comments). One big
problem with our code is it doesn’t prints **filename** or **line
number**, so it is hard at time if you just want to comment the log at
one place and not all logs entirely.\
Then, today I ran into [this awesome article by
AgentM](http://borkware.com/rants/agentm/mlog/), and it made me update
my code as:

``` objc
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

``` objc
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
