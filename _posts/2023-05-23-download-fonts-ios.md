---
layout: post
title:  "Downloading fonts on iOS"
date:   2023-05-15 21:58:00 +0200
categories: ios fonts
published: true
---

So it turns out you can download a bunch of fonts at runtime on a device. But not every font. There's a selected list that Apple has made available that you can find here [https://developer.apple.com/fonts/system-fonts/](https://developer.appl.e.com/fonts/system-fonts/). 

So how do you actually download these fonts? It's not as trivial as it could be. You need to dip your hands into `CoreText`. But no complains, it's a beautiful C API that works across all Apple platforms and you can even easily mix it your other cross-platform projects. 

And when I say API I mean just this one function:

```c
bool CTFontDescriptorMatchFontDescriptorsWithProgressHandler(
    CFArrayRef descriptors,
    CFSetRef mandatoryAttributes,
    CTFontDescriptorProgressHandler progressBlock
);
```

There are only two steps before this function can be called:

1. Create an attributes dictionary
1. Create a font descriptor with the attributes dictionary

Here's how it looks with Swift

```swift
    // Create an attributes dictionary
    let attrs = NSDictionary(
        object: fontName, 
        forKey: kCTFontNameAttribute as NSString
    )
    // Create font descriptor
    let desc = CTFontDescriptorCreateWithAttributes(
        attrs as CFDictionary
    )
    // Download font
    CTFontDescriptorMatchFontDescriptorsWithProgressHandler(
        [desc] as CFArray, nil) { state, _ in
        if state == .didFinish {
            // Font probably downloaded!
            DispatchQueue.main.async { 
                // notify listeners
            }
        }
        return true
    }
```

I've skipped a lot of other good things that can be done here, like error handling and tracking progress. But after a while the `state` would end up at `.didFinish` and by then if all went well a font would've been downloaded somewhere for our use.

So how do we get the said font? Simply like we get any other font:

```swift
let font = UIFont(name: fontName, size: size)
```

The only missing element is the value to pass in for `fontName`. 

Turns out you can't simply copy paste the *Font Name* from the [link](https://developer.apple.com/fonts/system-fonts/) above. For example, if you pass `"Apple Chancery"` as the `fontName`, the font API might give you back is actually `"Helvetica"`. What?!

The answer is hidden somewhere in the `CTFontDescriptor.h`. Take a look at the key we used when creating our attributes dictionary:

```c
/*!
    @defined    kCTFontNameAttribute
    @abstract   The PostScript name.
    @discussion This is the key for retrieving the PostScript name from 
    the font descriptor. When matching, this is treated more generically:
     the system first tries to find fonts with this PostScript name. If 
     none is found, the system tries to find fonts with this family name, 
     and, finally, if still nothing, tries to find fonts with this display 
     name. The value associated with this key is a CFStringRef. If 
     unspecified, defaults to "Helvetica", if unavailable falls back to 
     global font cascade list.
*/
const CFStringRef kCTFontNameAttribute;
```
So two things: One the system returns `"Helvetica"` when it can't find any better matching font, and second, the `fontName` has to be the *PostScript* name, whatever that means. 

So where could you find the *PostScript* name? One good answer is the **Font Book.app** that comes with your Mac.

In the info pane you can see the *PostScript* name for any font.

![Font Book]({{ site.url }}/assets/download-fonts-ios/fontbook.png)

So yes the correct value for `fontName` is `"Apple-Chancery"`.

## Conclusion
When does it make sense to use this API? I think for most apps that simply use a selected list of fonts it's probably simpler to just bundle the custom font. For document based apps where you can't possibly bundle many different fonts you can download them on demand. Also since this is not actually a font downloading service, but more like a font matching where the service finds you the best match for the provided attributes, it is probably sensible to assume that you won't always get the exact font you requested for.

## References

1. https://developer.apple.com/fonts/system-fonts/
1. https://developer.apple.com/documentation/coretext/1511433-ctfontdescriptormatchfontdescrip?language=objc
1. https://developer.apple.com/library/archive/samplecode/DownloadFont/Introduction/Intro.html#//apple_ref/doc/uid/DTS40013404