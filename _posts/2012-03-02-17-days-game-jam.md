---
layout: post
title:  "17 Days Game Jam!"
date:   2012-03-02 23:28:54 +0530
categories: gamedev
---

### DAY 1: 0917 hours

So, here’s the plan:

I’ll coding a game from the scratch to the AppStore for next 17 days, and I’ll try to update this blog as often as possible, I’m thinking about at least once per day.

Here is the fact, my job profile and the human inside me doesn’t allows me to code for full 24 hours, although I wish I could, but I can’t that’s the fact of life, but I’m trying to target coding at least 4 hours on weekdays and 10 hours on weekends.

The following is the list of resources I’ll be using (hopefully):

Xcode 4.3
Inkscape
Paintbrush
sfxr (the flash version, I know there’s a cfxr port for mac too)
Garage band
And I’ll try to code everything in pure ObjectiveC with OpenGL ES 2.0.

### RAQ (Rarely asked questions):

1. Why 17 days?

Because according to one holy book I’ve read, the God in that religion created the whole universe in 17 days, I’m kidding if you didn’t noticed. I don’t know why I picked 17, it’s just a random number that sounds cool, putting it another way- because it’s the largest prime number known to me.

2. Why OpenGL ES 2.0?

Because I’m just learning OpenGL, and 2.0 is just awesome enough for me, also I’m planning to put some funky looking effects that come with GLSL.

Of the above mentioned tools, which ones are your favorite?
Franky saying I’ve never used OpenGL at such large scale, and I’ve created few rectangle with Inkscape. I’ve used Paintbrush a lot for annotating red circles and arrows on some screenshots. SFXR, I’ve played for few seconds and was the easiest thing to master. As for GarageBand, I’ve no experience of using it as such, but my half-baked musical knack should get me through it, hopefully.

OK, the question-answer session is over, and as a great Ninja Panda once said: Enough Talk, Let’s Code!

### Day 1: 1030 hours

Here is the overview of the gameplay:

It’s somewhere in some unknown jungle, there is this guy totally lost. The place is full of wild animals, that look cute on TV, but not when from 50 meters. We need to move it out of the jungle in the safest way and since it’s that phase of the year when there is lot of water scarcity, so watch out for the energy, before the dehydration puts you to sleep forever!

### Day 3: 2110 hours

I just got so much involved in creating the basic engine, that I forgot to update the blog.

So, here is the story so far: I’m able to create basics classes, (including the abstract classes and shaders), to run a simple demo of objects fading in and out, as it would happen in the game, when the day turns into the night and then day again and all the objects on the screen turn dark and bright.

### Day 4: 1806 hours

So, now I’m able to do the day and night effect to some satisfactory level.

The shader has been upgraded from a custom Diffuse shader to the almost Phong shader!

I’ve also written a Factory to load to load .OBJ files, first I tried using the .PLY files, as I was getting the normals with that, but later on I found that to create the OBJ with normals requires just a checkmark in Blender. And, then I found this interesting perl script to convert the OBJ files to C type .h files.

I found one problem, not actually an problem but an optimization, that it creates two arrays for vertices and normals, but Apple recommends using interleaved arrays, and even I love the concept, but as most probably I’m not going to write a full 3D game, so it doesn’t concerns me that much. I was just trying to test my shaders and routines to do the day-night effect.

Next, I’m going to start the main gameplay, the day-night effect was the sort of thing I wanted to prototype before jumping into the game. So, most probably you won’t see any funny looking cubes and spheres.

### Day 12: 0612 hours

I know it’s been a long time, so here’s the update. I’m late on schedule, and most of that is because of my ignorance about some facts that I should’d learned earlier. But, anyways the thing I recently learned about the OpenGL ES is that, it’s recommended, and not just simply recommended by very strongly recommended that we should always prefer to render a huge chunk of triangles at a given, so instead of pushing 2 triangle that you might be doing for a 2D game to create a quad, you must buffer the whole screen at a time a render it with one call.

So, this knowledge changed my whole design of the Game Engine I’d written, I was basically doing the same thing –  calling render for each object, But now I’ve redesigned the whole thing and now my each scene is divided into layers, where each layer encapsulates a bunch of objects to be rendered. For e.g., a gameplay scene might consist of one background layer where the sun, cloud, and the mountains are and the other action layer might contain our survivor man and all the animals of the Jungle, then on top of that could be the users panel that displays vital information like energy left, or time or score or the pause button, whatever, I think you got the idea, right?

So, how do we render the polygons per layer based? You might ask it if you’re used to keep all the vertex data with respect to the origin, like your square with two triangles might look something like:
```
GLfloat vertexData[] = {
 
-1.0, 1.0, 1.0,
 
-1.0, -1.0, 1.0,
 
1.0, 1.0, 1.0,
 
1.0, 1.0, 1.0,
 
-1.0, -1.0, 1.0,
 
1.0, -1.0, 1.0,
 
};
```

The simplest way I could think is, to do the model and view transformation on the CPU itself, and then send the data to the GPU.

To cut the long story short, here is the demo, the Grizzly Bears art I found somewhere on this website


Here you can see two same set of vertex data on different layers with modified View matrix and the light colors calculation (actually they are simply phasing with cos and sin respectively :) )

And another thing I did I think worth mentioning is writing a class to use texture altas. I’d heard nice things about both Texture Packer and zwoptex, and I tried both and both look good to me at the moment, and they good thing about both is they both export to Cocos2D type plist, and I love plist and cocoa loves plist, for one reason it can avoid you all the headache for parsing an XML or text file or whatever, and second thing great about a plist file from cocoa perspective is that there are constructors available for all core cocoa datastructures like NSArray, NSDictionary to initialize directly from a plist file. Right now I’m using zwoptex, let’s see how it goes.

So, next thing is doing some graphics work and adding them to the game.

### Day ∞:

I don’t remember what happened, the whole GameJam thing came down like Kony 2012, when my computer crashed and I went from corner to corner hunting the missing pieces.

This is how I’m feeling about the whole affair right now :(
