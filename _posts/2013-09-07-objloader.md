---
layout: post
title:  "Experiment 4: OBJLoader"
date:   2013-09-07 23:28:54 +0530
categories: gamedev ios
---

This article is part of the Experiments with OpenGL series.

Frankly the motivation behind this experiment was that I was bored of rendering cubes and triangles. So, I decided to render something interesting, maybe a ninja!

OBJ is one of the most popular file format for 3D meshes. I should most convenient instead of popular. You can literally parse it with scanf, thats what I did.

Although, it does contains information for normals, but I used the same trick I learned from iPhone 3D Programming, one of the best books for getting into OpenGL ES. Being the lazy, shameless scum I am, I also used the OBJ files provided by the code that accompanies the book.

I didn’t just stopped there, I even used the algorithm described by Philip for calculating normals from vertex positions.

I seriously recommend to get that book and read it cover to cover, but still I’ll describe the algorithm here.

Let us consider a triangle suface



The easiest way to calculate the surface normal for this triangle is to find two edge vectors, lets say AB and AC as
```
AB = B – A

AC = C – A
```


And take their cross product.
```
N = AB x AC
```


This was high school stuff, but now the trick part. For each vertex position that is shared among more than one triangles, we calculate the average and calculate the final normal. So, if two triangles T1 and T2  with surface normals N1 and N2 respectively share a common vertex P, we just take their average N.



I’m not sure if it is better than parsing the normals from the OBJ file itself, but it’s good for the lazy days.

Anyways, checkout the code from the repository and thanks to Philip Rideout and artists William Vaughan and Christophe Desse for blessing non creative ass like me with some cool meshes.
