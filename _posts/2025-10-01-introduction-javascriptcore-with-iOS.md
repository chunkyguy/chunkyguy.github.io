---
layout: post
title: Introduction to JavascriptCore with iOS
date: 2025-10-01 22:08 +0200
categories: js objc ios
published: true
---

iOS for some reason provides an excellent integration with javascript via JavaScriptCore. Let's have some fun with it.

### Set up

So if we have a function defined in js

```js
// main.js
function greeting() {
  return "Hello from JS!"
}
```

First we need to set up the js context:

```objc
JSVirtualMachine *_vm;
JSContext *_context;

_vm = [[JSVirtualMachine alloc] init];
_context = [[JSContext alloc] initWithVirtualMachine:_vm];
```

Then we need to load our javascript

```objc
- (void)evaluate:(NSString *)filename {
  NSURL *sourceURL = [[[NSBundle mainBundle] bundleURL] URLByAppendingPathComponent:filename];
  NSError *error = nil;
  NSString *script = [NSString stringWithContentsOfURL:sourceURL
                                              encoding:NSUTF8StringEncoding
                                                 error:&error];
  if (error != nil) NSLog(@"Error: %@", [error description]);
  [_context evaluateScript:script withSourceURL:sourceURL];
}

[self evaluate:@"main.js"];
```

### Calling javascript from native

Calling js from native is requires calling `evaluateScript` which returns a value of type `JSValue`. 
`JSValue` provides several convenience utilities to map the data back to native types.

```objc
JSValue *result = [_context evaluateScript:@"greeting()"];
NSLog(@"%@", [result toString]); // Hello from JS!
```

Another way to perform the same operation is by first getting the reference to the `greeting` and then invoking it

```objc
JSValue *greetingFn = _context[@"greeting"];
JSValue *result = [greetingFn callWithArguments:[NSArray array]];
NSLog(@"%@", [result toString]); // Hello from JS!
```

### Keeping reference to JSValue

Since `JSValue` has a strong reference to its `JSContext`. So it is not safe to keep a strong reference to `JSValue` outside of its immediate scope. If we have a need to keep the reference arround, then the solution is to use `JSManagedValue`

```objc
JSManagedValue *jsv;

jsv = [JSManagedValue managedValueWithValue:_context[@"greeting"]];
[_vm addManagedReference:jsv withOwner:self];

JSValue *greetingFn = [jsv value];
[greetingFn callWithArguments:[NSArray array]];

[_vm removeManagedReference:jsv withOwner:self];
```

### Calling native from javascript

So, if wish to have a `print()` that is actually implemented on the native side using with `NSLog`

```js
function greeting() {
  print("Hello from JS!")
}
```

The simplest way is to use a block. The runtime does all the heavy lifting of mapping data from native to js. 

```objc
_context[@"print"] = ^(NSString *message) {
  NSLog(@"JS:Log: %@", message);
};

JSValue *greetingFn = _context[@"greeting"];
[greetingFn callWithArguments:[NSArray array]];
```

Then to add something like `console.log` we need to first create a `console` object and have a `log` function as a property on it.

```js
function greeting() {
  console.log("Hello from JS!")
}
```

```objc
JSValue *console = [JSValue valueWithNewObjectInContext:_context];
_context[@"console"] = console;
console[@"log"] = ^(NSString *message) {
  NSLog(@"JS:Log: %@", message);
};
```

In case we wisth to inject some native data type into js we need to use the `JSExport`. The idea is to expose a type interface as `protocol JSExport` and then provide the implementation is a `NSObject` subclass.

```objc
@protocol JSConsole <JSExport>
- (NSString *)version;
- (void)log:(NSString *)message;
@end

@interface Console : NSObject <JSConsole>
@end

@implementation Console
+ (NSInteger)version {
  return @"1.0";
}
- (void)log:(NSString *)message {
  NSLog(@"JS:Log: %@", message);
}
@end
```

### Chess game

Enough theory, lets now make a chess [random vs random](https://chessboardjs.com/examples#5002) game using [chess.js](https://github.com/jhlywa/chess.js).

Since most of the chess implementation is going to be provided with `chess.js` we just need to create a `Chess` instance and provide a `playRandomMove()` to update the state when invoked.

```js
// main.js
const chess = new Chess();

function playRandomMove() {
  if (chess.isGameOver()) {
    return chess.ascii();
  }

  const moves = chess.moves();
  const move = moves[Math.floor(Math.random() * moves.length)];
  chess.move(move);
  return chess.ascii();
}
```

```objc
@interface ChessController () {
  JSVirtualMachine *_vm;
  JSContext *_context;
}
@end

@implementation ChessController
- (instancetype)init {
  self = [super init];
  if (self) {
    _vm = [[JSVirtualMachine alloc] init];
    _context = [[JSContext alloc] initWithVirtualMachine:_vm];
  }
  return self;
}

- (void)evaluate:(NSString *)filename {
  NSURL *sourceURL = [[[NSBundle mainBundle] bundleURL] URLByAppendingPathComponent:filename];
  NSError *error = nil;
  NSString *script = [NSString stringWithContentsOfURL:sourceURL
                                              encoding:NSUTF8StringEncoding
                                                 error:&error];
  if (error != nil) NSLog(@"Error: %@", [error description]);
  [_context evaluateScript:script withSourceURL:sourceURL];
}

- (void)setUp {
  [_context setExceptionHandler:^(JSContext *context, JSValue *exception) {
    NSLog(@"JS:Error: %@", exception);
  }];
  [self evaluate:@"chess.js"];
  [self evaluate:@"main.js"];
}

- (NSString *)playRandomMove {
  JSValue *moveFn = _context[@"playRandomMove"];
  JSValue *result = [moveFn callWithArguments:[NSArray array]];
  NSString *board = [result toString];
  return board;
}
@end
```

Next for the native UI we only require a simple text view that renders the latest chess board.
```objc
@interface ViewController () {
  ChessController *_chessCtrl;
  UITextView *_textVw;
}
@end

@implementation ViewController
- (void)viewDidLoad {
  [super viewDidLoad];
  
  CGSize winSize = self.view.frame.size;
  CGFloat size = MIN(winSize.width, winSize.height);
  CGRect frame = CGRectMake(0, 0, size, size);
  
  _textVw = [[UITextView alloc] initWithFrame:frame];
  [_textVw setCenter:[self.view center]];
  [_textVw setEditable:NO];
  [_textVw setFont:[UIFont monospacedSystemFontOfSize:20
            weight:UIFontWeightBold]];
  [self.view addSubview:_textVw];
  
  _chessCtrl = [[ChessController alloc] init];
  [_chessCtrl setUp];
  
  [NSTimer scheduledTimerWithTimeInterval:1
                                   target:self
                                 selector:@selector(playRandomMove)
                                 userInfo:nil
                                  repeats:YES];
}

- (void)playRandomMove {
  [_textVw setText:[_chessCtrl playRandomMove]];
}
@end
```


![random vs random](/assets/javascriptcore/00_chess.png)

### References

- [JavaScriptCore API](https://developer.apple.com/documentation/javascriptcore?language=objc)
- [JavaScriptCore C API](https://developer.apple.com/documentation/javascriptcore/c-javascriptcore-api?language=objc)
