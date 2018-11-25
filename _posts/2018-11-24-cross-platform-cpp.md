---
layout: post
title:  "Cross platform mobile development with Djinni"
date:   2018-11-24 00:54:00 +0200
categories: djinni cpp ios android
published: true
---

# Hello Djinni

## Motivation

One of my last years goal was to make a Android app. It just occurred to me that even though I had been making app for such a long time, I had never made an Android. The app that I eventually made was far from being anything but glorious. It was just a empty with a label saying **Hello World**. Nonetheless, it was a milestone of my career. I should also mention, that being a Java illiterate I wrote that app in C++, so most my time was actually spent fighting with the JNI.

To continue the story, this year I want to make a cross platform app. Like one codebase that runs on both iOS and Android. I'll be using C++ as much as I can. I've heard [good](https://slack.engineering/libslack-the-c-library-at-the-foundation-of-our-client-application-architecture-97470b5ef9b3) [things](https://pspdfkit.com/blog/2016/a-pragmatic-approach-to-cross-platform/) about the [Djinni from Dropbox](https://github.com/dropbox/djinni), so I'll try that.

## Hello Djinni

I've already seen the [CppCon 2017](https://www.youtube.com/watch?v=ssqhz_1pPI4) and [CppCon 2014](https://bit.ly/djinnivideo) videos already. I found [this awesome](http://mobilecpptutorials.com) tutorial which I would be following and see how it goes.

The first part is setting up. If you already have Android studio set up, good, otherwise be prepared to spend half a day setting up the system. But, on the bright side everything works as documented, no random issues. The real game begins when everything is setup.

For testing the water, I cloned the Djinni from the repo and tried running it for a simple interface:

```
photoapp = interface +c {
  static create(): photoapp;
  get_photoapp(): string;
}
```

With a simple command

```
../djinni/src/run \
   --java-out $java_out \
   --java-package $java_package \
   --ident-java-field mFooBar \
   --cpp-out $cpp_out \
   --cpp-namespace $namespace \
   --jni-out $jni_out \
   --ident-jni-class NativeFooBar \
   --ident-jni-file NativeFooBar \
   --objc-out $objc_out \
   --objc-type-prefix $objc_prefix \
   --objcpp-out $objc_out \
   --idl $djinni_file
```
And after downloading a bunch of more stuff, it chokes on some random `Unresolved dependencies` error. Retrying with sudo does not help either. From what I understand, the Djinni is build with Scala and there is this sbt that builds the projects. When running the packaged `./src/build`, the installer hangs on some dependency issues. Next step could be try building directly with [sbt](https://www.scala-sbt.org/) with 

```
$ brew install sbt@1
```

Next, I would try running the installer directly.

```
$ sbt
```

This, launches the interactive shell. After loading the setting for the current Djinni project, the shell waits for me provide inputs. 

```
> help
  tasks                                   Lists the tasks defined for the current project.
```

This sounds promosing. 

```
  compile          Compiles sources.
[success] Total time: 32 s, completed Nov 24, 2018 3:22:17 PM
```

Finally, the war with installing Djinni seems to be over. Next up making the actual app. 

## Setting up the structure

The first question I've is to figure out how the file system looks like with a cross platform app. This is how the example app seems to be laid out.


```
photoapp.djinni 	; Djinni input IDL file
generated-src/		; Djinni output
handwritten-src/	; Shared implementation
android/			; Android app
ios/				; iOS app
```

I like this, except I would like to wrap all common parts in one directory. Something like:

```
shared/photoapp.djinni 		; Djinni input IDL file
shared/generated-src/		; Djinni output
shared/handwritten-src/		; Shared implementation
android/					; Android app
ios/						; iOS app
```

After a simple run, the `ls -R` looks like:

```
shared

./shared:
generated-src   photoapp.djinni run_djinni.sh

./shared/generated-src:
cpp  java jni  objc

./shared/generated-src/cpp:
photoapp.hpp

./shared/generated-src/java:
com

./shared/generated-src/java/com:
whackylabs

./shared/generated-src/java/com/whackylabs:
photoapp

./shared/generated-src/java/com/whackylabs/photoapp:
Photoapp.java

./shared/generated-src/jni:
NativePhotoapp.cpp NativePhotoapp.hpp

./shared/generated-src/objc:
PAPhotoapp+Private.h  PAPhotoapp+Private.mm PAPhotoapp.h
```

My first struggle is what editor to open the files in. I know at some point I'm going to create a Xcode project, so I can do that right now. The problem I have with editing standalone files is that they do not get any Xcode magic, no autocompletion or even good syntax highlighting. It is almost same as editing in a Notepad. We can think of the codebase composed of 3 independent components:

1. PhotoAppLib: C++ library
2. PhotoApp iOS: iOS client
3. PhotoApp Android: Android client

With that in place, this is how the structure looks like:

```
shared/photoapp.djinni 		; Djinni input IDL file
shared/generated-src/		; Djinni output
shared/lib/					; Shared implementation
android/					; Android app
ios/						; iOS app
```

## Setting up the iOS client

Tackling this part first since I've know all ins and outs of Xcode. Also I believe that with every cross platform development everyone has one favorite client, for me that is iOS and Android would be more or less a side product.

A way to set up the whole project could be to simply wrap the library in its own which comes bundled with a bunch of header files. Or another way could be to include all the files required by each clients. I personally like the latter to avoid dealing with link time issues. All the issues would be discovered at compile time and after the compiler is done the rest should sail smoothly.

With all the necessary files in the project, when I do a build I get the error `DJICppWrapperCache+Private.h` not found. And indeed when I look in my project I don't find that file. Maybe there is something I missed somewhere, because I do see these files somewhere in the `support-lib` for Djinni source, but somehow these did not made into the autogenerated project. Could be that I was supposed to bring these in manually, so let's do that. 

Turns out that was indeed the missing puzzle. This is how the iOS project looks like

![iOS Project](https://i.imgur.com/MmTAgcV.png)

And here is the text on screen

![iOS client](https://i.imgur.com/AqlsBzl.png)

## Setting up the Android client

This would be even more interesting to set up, since I've almost 0 experience with Android Studio or even how Java build system and runtime works. But, it seems the out of the box for C++ from Android has greatly improved since the last time I experimented with it. Last time I had to download and install the NDK and it was so much painful as if the Android people did not want anyone to even write C++. This time the set up is almost as clean as in Xcode, you just simply tick the correct boxes and a hello world app template pops up. Nice!

And on top of it the build system is now a rather familiar Cmake. I really love this new Android development environment. This is my `CmakeLists.txt`:

```
cmake_minimum_required(VERSION 3.4.1)

# Add shared files
file(
        GLOB photoapp_sources
        ../../shared/support-lib/jni/*.cpp
        ../../shared/generated-src/jni/*.cpp
        ../../shared/src/*.cpp
)

# Provide include files
include_directories(
        ../../shared/support-lib/
        ../../shared/support-lib/jni/
        ../../shared/src/
        ../../shared/generated-src/cpp/
        ../../shared/generated-src/jni/
)

# Create library from the shared files
add_library(
        photoapplib
        SHARED
        ${photoapp_sources}
)

# Link library
target_link_libraries(photoapplib)
```

And then after reading the same string from the shared library and rendering it on a `TextView` we get

![Android screenshot](https://i.imgur.com/xGDuB9b.png)

## Thanks

So how do I feel about Djinni so far? I would say, I did like it a lot, if I would every have to do a cross platform mobile app, I would love to do it with Djinni. I also did love the fact that how much improvements have been made in the Android IDE to support C++.

I'll probably expand this project further to the regular PhotoApp that I try to make with every toolkit that I try, which involves fetching some data, parsing it and rendering it on screen. I'll also post my notes here on this blog.

The entire [project is available](https://github.com/chunkyguy/PhotoApp/tree/master/x-platform-djinni) for referencing. 

Thanks for reading! 
