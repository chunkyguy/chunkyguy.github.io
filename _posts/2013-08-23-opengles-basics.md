---
layout: post
title:  "Experiments with OpenGLES: 01 The Basics"
date:   2013-08-23 23:28:54 +0530
categories: opengl ios
---

This article is part of the Experiments with OpenGL series.

I’ve decided to start a new series on my experiments with OpenGLES.

I’ll be targeting iOS as the platform and use C/Objective C as the language.

## Why C?

I’m planning to use C for many reasons:

1. It’s simplicity. I’m a professional C++ programmer, but for my experiments I don’t to fight with the language. I just want few things to just as quickly as possible.

2. I’m trying to think out of the Object Oriented box, and go back to the traditional functional programming. I know using all the Object Oriented languages for so many years has corrupted my thinking process. I’m also aware of the fact that as the code gets larger functional approach gets messier, but since these are just my experiments, I don’t expect the code to flow out into too many files.

3. I’ve recently joined the C-games group and that is bringing a lot of C nostalgia, the time when I was a new kid and learning programming.

## The Boilerplate.

To begin with I’ve written some boilerplate code, that just gets the things running. You can check out the latest code from this repo: https://github.com/chunkyguy/EWOGL

To summarize, here are the list of things I’m doing.

1. At app launch, create a UIView and add it to the main window.

2. Configure the UIView to be able to run OpenGL commands, using:

``` objc
+ (Class) layerClass {
return [CAEAGLLayer class];
}
```
3. Configure the EAGL.

4. Create the EAGL context, that manages the rendering to Core Animation layer.

5. Create a framebuffer with 2 renderbuffers, a color renderbuffer and a depth renderbuffer.

6. Set the viewport to view’s size.

7. Compile the shader.

8. Start the loop.

9. The loop then call Init (only once) and Update with delta time in millisecs and a Render function with compiler GLSL program.

The Init functions create all things that would be used by the Update/Render each frame.



References:

1. Apple’s OpenGL ES Guide.
