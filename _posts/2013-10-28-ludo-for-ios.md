---
layout: post
title:  "Ludo for iOS"
date:   2013-10-28 23:28:54 +0530
categories: gamedev
published: false
---

## Introduction

I’ll be creating a game from scratch for iPad. I’ll be using mainly C and OpenGLES and any other thing to get the job done.

The entire code is at https://github.com/chunkyguy/Ludo.

Watch my facebook page and twitter for updates.

### Day 8

I’m so tired, been working from 10 AM. It’s now 4 AM, that means 18 hours with a 4 hours English Premier League break. I wanted to finish off the AI part, but I don’t think any I’m near that.

Maybe, I’m back to where I started with. Sometimes, ideas in my head just go in circles. One things sounds awesome. And, after I’ve spent hours on it, it seems silly.

I was trying a new approach to save/load the data. At one point I even thought of adding sqlite for backend. But, finally I’ve decided to use custom binary files. One won’t be working on more than one game simultaneously, so we can safely read/write to a single file.

Another thing I was trying was, a new flow for the data. It was based on a giant moves queue, something like we have inbox in our email. The moves queue would cache all the moves coming from different places. But, in the end I commented it out. The code is still there in LD_MoveQueue. Maybe, I’ll implement that later.

The animation is still pending. The gamecenter integration is still pending. So many bad news. The only good news today was ManU defeated Arsenal, that means the gap between Liverpool and Arsenal is now just 2 points! YAY!

### Day 7

So, after so many years, I mean days of hard work is paying off. Today, the first ever human vs 3 AI match was conducted.

I’ve added a game selector screen, where you can pick the kind of intelligence each player is going to have.

Hosted by imgur.com

Currently, as there is no animation, you won’t have any idea what is going on. You play a move and in the split second, it’s your turn again. But, you can guess by the changed locations of the other player’s pieces.

So, next up is animation.

### Day 6

I moved the main gameplay code to a separate file called LD_Game.m. Again, it is 99% C right now and after the prototyping will be over it’ll be 100% C.

Another thing that I modified, was to move all the rendering oriented code to another directory. It’s called Prototype, as all the rendering is being done with UIKit for now.

So, GUI wise there is not much difference. But the code is now much modular.

Also, I’ve added some GameCenter code, and the flow of the code is becoming more of GameCenter oriented. Like, for example now every thing is happening based on user actions.

Later, I’m planning to some synchronous animations. Like, say you tap on the dice and the dice rolling animation runs synchronously. At the end the user will again be able to update the game state.

### Day 5

Worked on finishing the basic rules of the game. There were some bug with the path mapping, now fixed.

To work with testing, I’ve added few compile time cheatcodes. One of them is the autoplay mode, where the computer plays the next player automatically. Another is the dice, it won’t generate a random number, but the number you’ve selected from the UI. See the image for details.

### Day 4

Made the code more asynchronous. Building towards network based play. Also, reduced the global vars, so that more context sensitivity could be achieved.

### Day 3

Finally some movement has started in the game.

The main trick is to use a tileset map, which contains all the points for all the cells (15×15) and a path map that has all valid positions any piece can take.

Creating the tileset map was just running two for loops, but creating a path map needed some work, as the path for any piece is in clockwise direction in a complete circle. I took the easiest path and wrote a small code to generate the valid path for each piece type.

Also, in the codebase you might notice I’ve added Lua. The reason for that is that I’m thinking of using Lua for all the configuration files. This would save me some time, as I won’t have to recompile the code everytime I tweak some values.

In the final release though, we can get away with Lua and use only static config files.

### Day 2

Day started with me spending time rendering planes and lines to get a board on the screen. And I was successful for a while



But, then I thought of saving the awesome rendering for later and first just get the thing work as quick as possible. And, what’s the quickest way to design UI for iOS than
the Interface Builder.

So, with the help of the UIKit magic, here is our Ludo board;



### Day 1

The game will target iPad. I’m writing all the code from scratch*.

I’ll be using C (as much as possible), the filenames are .m because I’m GLKMath library, which is again C based but for some reasons doesn’t compiles well with plain C files.

The game is based on popular game called Ludo. I’m not sure about the name, so for the time being lets call it Ludo. I’m hoping to make it online multiplayer, with computer AI. For now, there are just 2 Cubes revolving around the center of the screen.



The latest code is available at https://github.com/chunkyguy/Ludo

