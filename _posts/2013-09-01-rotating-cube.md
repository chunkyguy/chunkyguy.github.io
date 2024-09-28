---
layout: post
title:  "Experiment 2: A rotating cube"
date:   2013-09-01 23:28:54 +0530
categories: opengl ios
published: false
---

This article is part of the Experiments with OpenGL series.

The experiment 2 started with an idea of extending the old experiment to 3D.

I just used the cube mesh we get with the Xcode’s OpenGL Game template. I also added the Transform code from the Hideous Engine.

In this experiment, I made things more simpler with all the framebuffer code all in the main file and a file Types.h for all the data-structures used in the experiment.

Although, the original idea was to implement a sort Camera effect, where I can zoom, pan things around but I stopped as soon as the cube started rotating on the screen. But, there is a small camera structure that I’m planning to extend more in further experiments.

Again, the code is live on https://github.com/chunkyguy/EWOGL and don’t take my words for it, for I may have added things to it, checkout the code for yourself.
