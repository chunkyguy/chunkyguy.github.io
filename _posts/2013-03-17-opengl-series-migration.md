---
layout: post
title:  "opengl-series iOS migration notes"
date:   2013-03-17 23:28:54 +0530
categories: gamedev ios
published: false
---

Today I spent few hours porting the Tom Dalling’s OpenGL sample code to work on iOS.

This is my current working repository, I’m still working on fixing few things. (Edit: The code is now merged with the original repository)

Even though I tried to keep things as consistent as possible, but for some reasons I had to change the code at few places, here I’m listing the thoughts behind that. I think it would help someone following the above tutorial for iOS, or for OpenGLES. The original main.cpp has been renamed to iOS_main.

1. The Loop:

The loop should be the most striking change. The main reason being that Apps on iDevices run in their own loop. We don’t have control over the loop, the system kind of passes the control to us when ever an event occurs. So, Update and Render are called directly from WLViewController.

2. Touch Events:

A new type eGesture is declared in iOS_main. For any kind of touch gesture, the RegisterGesture is called and the gesture state is saved. The gesture is then used in the Update.

Gesture include tapping on the four areas and tilting device up and down.

3. Linking glm:

I assume that you’ve installed glm using macports, as suggested by Tom in the first article. In any case you still have to go to Build Settings > Search Paths and provide glm’s location.

4. Code change:

All the code from original source remains the same, except when something doesn’t works for OpenGLES.

For example, in Program.cpp the MACRO hack for ATTRIB_N_UNIFORM_SETTERS is split into ATTRIB_SETTERS and UNIFORM_SETTERS.

Other than that, I think the articles can be easily followed. The code should work for iPhone and iPad in every orientation.

Here are my screenshots for each sample code:

Edit:
For Article 06: Diffuse Point Lightining migration and related gesture updates visit here.