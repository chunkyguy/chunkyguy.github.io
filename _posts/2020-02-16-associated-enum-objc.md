---
layout: post
title:  "Implementing enum with associated values in Objective-C"
date:   2020-02-16 14:14:00 +0200
categories: objc architecture
published: true
---

Swift has this amazing thing called [enum with associated value](https://docs.swift.org/swift-book/LanguageGuide/Enumerations.html#ID148). Enums are traditionally used to represent a type system with finite set of values. But in real life each of those enum types sometimes have a need to associate some data. And the data of one enum type might be very different than the rest. Imagine writing a parser where each `node` is of type `Node`. We usually want to deal with `Node` as the type for doing things like passing `Node` around, or generate an array of `Node`. We might also have methods on `Node`, such as `node.print()` but `Node` can never be instantiated directly as `n = Node()`. 

Swift programming guide provides a good example of this pattern with `Barcode` that I've enhanced here a bit for readability:

```swift
// Barcode.swift
enum Barcode {
  case upc(
    numberSystem: Int,
    manufacturer: Int,
    product: Int,
    check: Int
  )

  case qrCode(productCode: String)

  func print() {
    switch self {
    case let .upc(numberSystem, manufacturer, product, check):
      Swift.print("UPC: \(numberSystem), \(manufacturer), \(product), \(check).")
    case let .qrCode(productCode):
      Swift.print("QR code: \(productCode).")
    }
  }
}
```
```swift
// Main.swift
func print(barcode: Barcode) {
  barcode.print()
}

func printBarcodes() {
  print(barcode: .upc(
    numberSystem: 10,
    manufacturer: 20,
    product: 30,
    check: 40
  ))

  print(barcode: .qrCode(productCode: "asdfa"))
}
```

How can we bring this pattern to Objective-C? This is how I think of the problem. Every use case of an enum with associated value can be represented as a subclass pattern, where every subclass is only a level deep. And the base class is an abstract class that can not be instantiated. The Barcode example from above might look something like:

[![](https://mermaid.ink/img/eyJjb2RlIjoiY2xhc3NEaWFncmFtXG5cdEJhcmNvZGUgPHwtLSBCYXJjb2RlUVJcblx0QmFyY29kZSA8fC0tIEJhcmNvZGVVUENcbiAgQmFyY29kZSA6IC12b2lkIHByaW50XG5cblx0Y2xhc3MgQmFyY29kZVVQQyB7XG4gICAgICBudW1iZXJTeXN0ZW06IEludCxcbiAgICAgIG1hbnVmYWN0dXJlcjogSW50LFxuICAgICAgcHJvZHVjdDogSW50LFxuICAgICAgY2hlY2s6IEludFxuXHR9XG5cbiAgY2xhc3MgQmFyY29kZVFSIHtcbiAgcHJvZHVjdENvZGU6IFN0cmluZ1xuXHR9XG5cblx0XHRcdFx0XHQiLCJtZXJtYWlkIjp7InRoZW1lIjoiZGVmYXVsdCJ9LCJ1cGRhdGVFZGl0b3IiOmZhbHNlfQ)](https://mermaid-js.github.io/mermaid-live-editor/#/edit/eyJjb2RlIjoiY2xhc3NEaWFncmFtXG5cdEJhcmNvZGUgPHwtLSBCYXJjb2RlUVJcblx0QmFyY29kZSA8fC0tIEJhcmNvZGVVUENcbiAgQmFyY29kZSA6IC12b2lkIHByaW50XG5cblx0Y2xhc3MgQmFyY29kZVVQQyB7XG4gICAgICBudW1iZXJTeXN0ZW06IEludCxcbiAgICAgIG1hbnVmYWN0dXJlcjogSW50LFxuICAgICAgcHJvZHVjdDogSW50LFxuICAgICAgY2hlY2s6IEludFxuXHR9XG5cbiAgY2xhc3MgQmFyY29kZVFSIHtcbiAgcHJvZHVjdENvZGU6IFN0cmluZ1xuXHR9XG5cblx0XHRcdFx0XHQiLCJtZXJtYWlkIjp7InRoZW1lIjoiZGVmYXVsdCJ9LCJ1cGRhdGVFZGl0b3IiOmZhbHNlfQ)

With that in mind we can design our interface as:

```objc
// PLBarcode.h
@protocol PLBarcode <NSObject>
- (void)print;
@end

@interface PLBarcodeUPC : NSObject <PLBarcode>
+ (instancetype)barcodeWithNumberSystem:(NSInteger)numberSystem
                           manufacturer:(NSInteger)manufacturer
                                product:(NSInteger)product
                                  check:(NSInteger)check;
@end

 @interface PLBarcodeQR : NSObject <PLBarcode>
+ (instancetype)barcodeWithProductCode:(NSString *)productCode;
@end
```

Notice that we are not actually subclassing any implementation class from any common base `PLBarcode` class.  This keeps every implementation class completely independent of each other with no shared hierarchy. We can use `PLBarcode` as the type that can easily pass around:

```objc
// Main.m
- (void)printBarcode:(id<PLBarcode>)barcode
{
  [barcode print];
}

- (void)printBarcodes
{
  [self printBarcode:[PLBarcodeUPC barcodeWithNumberSystem:10
                                              manufacturer:20
                                                   product:30
                                                     check:40]];
  [self printBarcode:[PLBarcodeQR barcodeWithProductCode:@"asfsdf"]];
}
```

Another advantage of enums is that we don't have to deal with so many explicit types as we have here with `PLBarcodeUPC` and `PLBarcodeQR`. I can imagine adding more implementation types would require bombarding the clients of `PLBarcode` with many implementation types that they don't actually care about. Thinking again, what we need is the following:

1. A way to instantiate `PLBarcode` with unrelated data.
2. Ability to send messages to `PLBarcode` which should be handled by the implementation type.

With that in mind, we can solve this problem by moving all the implementation subclasses internally to `PLBarcode.m` and expose only typeless class methods in `PLBarcode.h`, like a factory pattern

```objc
// PLBarcode.h
@protocol PLBarcode <NSObject>
- (void)print;
@end

@interface PLBarcode : NSObject
+ (id<PLBarcode>)barcodeWithNumberSystem:(NSInteger)numberSystem
                            manufacturer:(NSInteger)manufacturer
                                 product:(NSInteger)product
                                   check:(NSInteger)check;

+ (id<PLBarcode>)barcodeWithProductCode:(NSString *)productCode;
@end
```

```objc
// PLBarcode.m
@interface __PLBarcodeUPC : NSObject <PLBarcode>
+ (instancetype)barcodeWithNumberSystem:(NSInteger)numberSystem
                           manufacturer:(NSInteger)manufacturer
                                product:(NSInteger)product
                                  check:(NSInteger)check;
@end

@interface __PLBarcodeQR : NSObject <PLBarcode>
+ (instancetype)barcodeWithProductCode:(NSString *)productCode;
@end

@implementation PLBarcode

+ (id<PLBarcode>)barcodeWithNumberSystem:(NSInteger)numberSystem
                            manufacturer:(NSInteger)manufacturer
                                 product:(NSInteger)product
                                   check:(NSInteger)check;
{
  return [__PLBarcodeUPC barcodeWithNumberSystem:numberSystem
                                  manufacturer:manufacturer
                                       product:product
                                         check:check];
}

+ (id<PLBarcode>)barcodeWithProductCode:(NSString *)productCode;
{
  return [__PLBarcodeQR barcodeWithProductCode:productCode];
}
@end
```

With that change the call site looks much cleaner:

```objc
- (void)exampleBarcode
{
  [self printBarcode:[PLBarcode barcodeWithNumberSystem:10
                                           manufacturer:20
                                                product:30
                                                  check:40]];
  [self printBarcode:[PLBarcode barcodeWithProductCode:@"asfsdf"]];
}
```

## Resources

1. [Class Clusters Pattern • **developer.apple.com**](https://developer.apple.com/library/archive/documentation/General/Conceptual/CocoaEncyclopedia/ClassClusters/ClassClusters.html)