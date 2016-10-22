---
layout: post
title:  "Ludo for iOS"
date:   2013-10-28 23:28:54 +0530
categories: gamedev
---

Ludo for iOS
=============

Introduction
-------------

I’ll be creating a game from scratch for iPad. I’ll be using mainly C and OpenGLES and any other thing to get the job done.

The entire code is at https://github.com/chunkyguy/Ludo.

Watch my facebook page and twitter for updates.

Day 8

I’m so tired, been working from 10 AM. It’s now 4 AM, that means 18 hours with a 4 hours English Premier League break. I wanted to finish off the AI part, but I don’t think any I’m near that.

Maybe, I’m back to where I started with. Sometimes, ideas in my head just go in circles. One things sounds awesome. And, after I’ve spent hours on it, it seems silly.

Hosted by imgur.com

I was trying a new approach to save/load the data. At one point I even thought of adding sqlite for backend. But, finally I’ve decided to use custom binary files. One won’t be working on more than one game simultaneously, so we can safely read/write to a single file.

Another thing I was trying was, a new flow for the data. It was based on a giant moves queue, something like we have inbox in our email. The moves queue would cache all the moves coming from different places. But, in the end I commented it out. The code is still there in LD_MoveQueue. Maybe, I’ll implement that later.

The animation is still pending. The gamecenter integration is still pending. So many bad news. The only good news today was ManU defeated Arsenal, that means the gap between Liverpool and Arsenal is now just 2 points! YAY!

Day 7

So, after so many years, I mean days of hard work is paying off. Today, the first ever human vs 3 AI match was conducted.

I’ve added a game selector screen, where you can pick the kind of intelligence each player is going to have.

Hosted by imgur.com

Currently, as there is no animation, you won’t have any idea what is going on. You play a move and in the split second, it’s your turn again. But, you can guess by the changed locations of the other player’s pieces.

So, next up is animation.

Day 6

I moved the main gameplay code to a separate file called LD_Game.m. Again, it is 99% C right now and after the prototyping will be over it’ll be 100% C.

Another thing that I modified, was to move all the rendering oriented code to another directory. It’s called Prototype, as all the rendering is being done with UIKit for now.

So, GUI wise there is not much difference. But the code is now much modular.

Also, I’ve added some GameCenter code, and the flow of the code is becoming more of GameCenter oriented. Like, for example now every thing is happening based on user actions.

Later, I’m planning to some synchronous animations. Like, say you tap on the dice and the dice rolling animation runs synchronously. At the end the user will again be able to update the game state.

Day 5

Worked on finishing the basic rules of the game. There were some bug with the path mapping, now fixed.

To work with testing, I’ve added few compile time cheatcodes. One of them is the autoplay mode, where the computer plays the next player automatically. Another is the dice, it won’t generate a random number, but the number you’ve selected from the UI. See the image for details.

Hosted by imgur.com

Day 4

Made the code more asynchronous. Building towards network based play. Also, reduced the global vars, so that more context sensitivity could be achieved.



Day 3

Finally some movement has started in the game.



The main trick is to use a tileset map, which contains all the points for all the cells (15×15) and a path map that has all valid positions any piece can take.

Creating the tileset map was just running two for loops, but creating a path map needed some work, as the path for any piece is in clockwise direction in a complete circle. I took the easiest path and wrote a small code to generate the valid path for each piece type.

Also, in the codebase you might notice I’ve added Lua. The reason for that is that I’m thinking of using Lua for all the configuration files. This would save me some time, as I won’t have to recompile the code everytime I tweak some values.

In the final release though, we can get away with Lua and use only static config files.

Day 2

Day started with me spending time rendering planes and lines to get a board on the screen. And I was successful for a while



But, then I thought of saving the awesome rendering for later and first just get the
thing work as quick as possible. And, what’s the quickest way to design UI for iOS than
the Interface Builder.

So, with the help of the UIKit magic, here is our Ludo board;



Day 1

The game will target iPad. I’m writing all the code from scratch*.

I’ll be using C*, the filenames are .m because I’m GLKMath library, which is again C based but for some reasons doesn’t compiles well with plain C files.

The game is based on popular game called Ludo. I’m not sure about the name, so for the time being lets call it Ludo. I’m hoping to make it online multiplayer, with computer AI. For now, there are just 2 Cubes revolving around the center of the screen

*as much as possible.



The latest code is available at:https://github.com/chunkyguy/Ludo

Posted in OpenGL, _dev, _games, _iOS	| 1 Comment
Embedding Freetype for iOS projects
Posted on September 11, 2013 by chunkyguy
Freetype is a font library written in C. From OpenGL’s point of view, it creates alpha texture on the fly for any font size for any string.

Step 1: Embedding the original freetype source to the code is some real pain in the ass. Fortunate for us, one guy has done that task and released the code on github. https://github.com/cdave1/freetype2-ios. So, step 1 is to check out that code.



Step 2: Create your new project and add the project downloaded from step 1 to your project.



Check that the Xcode must have created two schemes for you.



Step 3: Go to the your Project Settings, select your target and click on ‘Build Phases’. Under ‘Target Dependencies’ add the static library from the freetype2 project.



This is going to build the static library for you, without you having to explicitly build it.

Step 4: Go to ‘Link Binary with Libraries’ and add the static library.



This will build the libFreetype2.a and put it inside the products directory.

Step 5: Next, we need to add the freetype header files in the products directory. Why? Because, the Xcode projects are configured to look for header files inside the products directory automatically, without us having to specify the search path. Also, if in future we make some changes to the freetype project header files (very unlikely) the changes will be automatically updated in our products directory.

Now, open the freetype2 Project settings and go to ‘Build Phases’. Click ‘Add Build Phase’ and ‘Add Copy Files’



Step 6: Change the ‘Destination’ to ‘Products Directory’. Now, this is the part where Xcode sucks, we need to copy our header files from the freetype2 project, but doing so flattens the copied directory structure.

One way to fix this is to remove the include files from the freetype2 project and add them again as ‘Folder reference’



Step 7: Drag the include folder from freetype2 project to the ‘Copy Files’ phase.



If you’ve followed everything correct to this point, you project should just build fine. It should compile the freetype target, copy the header files and then compile and link your project.



The products directory should look something like



For final testing, try adding the following lines of code to your main.m file and compile the code:

1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
#import <UIKit/UIKit.h>
#import "AppDelegate.h"
#import <ft2build.h>
#import <freetype/freetype.h>
#import <assert.h>
 
void test_freetype() {
    FT_Library library;
    FT_Error error = FT_Init_FreeType(&library);
    assert(!error);
    FT_Done_FreeType(library);
}
 
int main(int argc, char *argv[])
{
    test_freetype();
 
    @autoreleasepool {
        return UIApplicationMain(argc, argv, nil, NSStringFromClass([AppDelegate class]));
    }
}
Posted in _dev, _iOS	| Tagged freetype, iOS, iPad, iphone, project, xcode	| 0 Comments
Experiment 4: OBJLoader
Posted on September 7, 2013 by chunkyguy
This article is part of the Experiments with OpenGL series.

Frankly the motivation behind this experiment was that I was bored of rendering cubes and triangles. So, I decided to render something interesting, maybe a ninja!



OBJ is one of the most popular file format for 3D meshes. I should most convenient instead of popular. You can literally parse it with scanf, thats what I did.

Although, it does contains information for normals, but I used the same trick I learned from iPhone 3D Programming, one of the best books for getting into OpenGL ES. Being the lazy, shameless scum I am, I also used the OBJ files provided by the code that accompanies the book.

I didn’t just stopped there, I even used the algorithm described by Philip for calculating normals from vertex positions.

I seriously recommend to get that book and read it cover to cover, but still I’ll describe the algorithm here.

Let us consider a triangle suface



The easiest way to calculate the surface normal for this triangle is to find two edge vectors, lets say AB and AC as

AB = B – A

AC = C – A



And take their cross product.

N = AB x AC



This was high school stuff, but now the trick part. For each vertex position that is shared among more than one triangles, we calculate the average and calculate the final normal. So, if two triangles T1 and T2  with surface normals N1 and N2 respectively share a common vertex P, we just take their average N.



I’m not sure if it is better than parsing the normals from the OBJ file itself, but it’s good for the lazy days.

Anyways, checkout the code from the repository and thanks to Philip Rideout and artists William Vaughan and Christophe Desse for blessing non creative ass like me with some cool meshes.

Posted in OpenGL, _dev	| 0 Comments
Experiment 3: Reflection.
Posted on September 3, 2013 by chunkyguy
This article is part of the Experiments with OpenGL series.

This experiment started with just one thing in mind, the stencil buffer.

At this point of the series, you must be asking, why am I bothering myself with such trivial experiments. I wish to explain few things here:

1. These experiments are going to build up in building blocks forms. Each experiment is going to grow over each other, so, I’m keeping initial blocks as minimalistic as possible.

2. I’m trying to use C, to keep things as simple as possible. Also, this is the most C I’ve used in years. Years of Object Oriented philosophy has spoiled my brain. I spend great deal of time figuring out best ways possible to arrange my data structures. I’m focusing more on computer side, not the human side. Trying to keep the data packed, aligned, optimized, not fragmenting the memory unnecessarily. In my opinion C is one the best programming language a programmer can have, I’m just relearning it.

3. I’m also working on finishing my game, so I intend to keep each experiment at this stage at minimum level. One that I can finish off in couple of hours. Honestly, I don’t even test them thoroughly, I’m keeping the fixes for days when the experiments get more complicated and issues will start to surface. And there is also a chance that someday I might come back and improve all these experiments to a better level.

OK, so getting back to the experiment. This experiment is all about using a stencil buffer. I thought of doing a reflection on a random surface.



Posted in OpenGL, _dev, _iOS	| 0 Comments
Experiment 2: A rotating cube.
Posted on September 1, 2013 by chunkyguy
This article is part of the Experiments with OpenGL series.

The experiment 2 started with an idea of extending the old experiment to 3D.

I just used the cube mesh we get with the Xcode’s OpenGL Game template. I also added the Transform code from the Hideous Engine.

In this experiment, I made things more simpler with all the framebuffer code all in the main file and a file Types.h for all the data-structures used in the experiment.

Although, the original idea was to implement a sort Camera effect, where I can zoom, pan things around but I stopped as soon as the cube started rotating on the screen. But, there is a small camera structure that I’m planning to extend more in further experiments.

Again, the code is live on https://github.com/chunkyguy/EWOGL and don’t take my words for it, for I may have added things to it, checkout the code for yourself.

 


Posted in OpenGL, _dev	| 0 Comments
Experiments With OpenGL ES
Posted on September 1, 2013 by chunkyguy
This is the main index for all my experiments.

This is the link to the repository for all the experiments. https://github.com/chunkyguy/EWOGL

1. OGLBasic: The Boilerplate. Just basic code to get a triangle on the screen.

2. Camera: The rotating cube.

3. Reflection: Rotating cube inside a triangle.

4. OBJLoader: Loading an OBJ file.

Posted in OpenGL, _dev	| 0 Comments
Experiments with OpenGLES: 01 The Basics
Posted on August 23, 2013 by chunkyguy
This article is part of the Experiments with OpenGL series.

I’ve decided to start a new series on my experiments with OpenGLES.

I’ll be targeting iOS as the platform and use C/Objective C as the language.

Why C?

I’m planning to use C for many reasons:

1. It’s simplicity. I’m a professional C++ programmer, but for my experiments I don’t to fight with the language. I just want few things to just as quickly as possible.

2. I’m trying to think out of the Object Oriented box, and go back to the traditional functional programming. I know using all the Object Oriented languages for so many years has corrupted my thinking process. I’m also aware of the fact that as the code gets larger functional approach gets messier, but since these are just my experiments, I don’t expect the code to flow out into too many files.

3. I’ve recently joined the C-games group and that is bringing a lot of C nostalgia, the time when I was a new kid and learning programming.

The Boilerplate.

To begin with I’ve written some boilerplate code, that just gets the things running. You can check out the latest code from this repo: https://github.com/chunkyguy/EWOGL

To summarize, here are the list of things I’m doing.

1. At app launch, create a UIView and add it to the main window.

2. Configure the UIView to be able to run OpenGL commands, using:

1
2
3
+ (Class) layerClass {
return [CAEAGLLayer class];
}
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

Posted in OpenGL, _dev, _iOS	| 0 Comments
OpenGL Texture Coordinates.
Posted on July 2, 2013 by chunkyguy
Let’s talk about OpenGL Texture Coordinates. Whenever we need to render a bitmap image using OpenGL, we need to pass in something called the texture coordinates to the GPU.

I’ll be using flexible pipeline, but the approach can be easily used for fixed pipeline as well, as there is no shader magic involved. We’ll more focusing on calculating the texture coordinates.

For this tutorial I will be shamelessly ripping off this awesome webgl tutorial to render textures to a rotating cube, and tweak it to display a flat square with a texture. Scroll down for the entire code and the live demo link.

Let’s start with basic, rendering entire texture.

Let’s say this is the image I wish to render

Cool Guy

The texture coordinates start at origin top-left of the screen.

Cool Guy

And in OpenGL we render a quad as TRIANGLE_STIP as A->B->C->D

Cool Guy

So, texture coordinates for an entire texture is set as:

1
2
3
4
5
6
7
8
var textureCoords = [
 
0.0, 1.0,  //A
1.0, 1.0,  //B
0.0, 0.0,  //C
1.0, 0.0  //D
 
];
Cool Guy

Now if we wish to render a portion of the original texture, we can simply map it to points we wish, keeping in mind the texture coordinates above.

Let’s say we wish to render only the first half of the texture, we can set texture coordinates as:

1
2
3
4
5
6
var textureCoords = [
0.0, 1.0,
0.5, 1.0,
0.0, 0.0,
0.5, 0.0
];
Cool Guy

As you can see we are just rendering half the width. Similarly, we can render only the next half by:

1
2
3
4
5
6
var textureCoords = [
0.5, 1.0,
1.0, 1.0,
0.5, 0.0,
1.0, 0.0
];
Cool Guy

What if we just wish to render the logo on our cool guy’s shirt. Try a few experimenting around, you can find the texture coordinates, just keep in mind the way coordinate-system works for textures.

One way to guess the texture coordinates is assume the range of x-min, x-max and y-min, y-max. Like for above requirement, after few trial-n-errors I calculated the coordinates to be in range:

1
2
3
4
x-min = 0.1
x-max = 0.43
y-min = 0.4
y-max = 0.57
So, the texture coordinates becomes:

1
2
3
4
5
6
var textureCoords = [
0.1, 0.57,
0.43, 0.57,
0.1, 0.4,
0.43, 0.4
];
Cool Guy

Now, in real life you seldom need to calculate the points yourself. There are plenty of applications that do that for you. One of my favorite is Zwoptex. You just need to drop your images into these sprite-sheet maker apps, and they give you a single texture, with an associated data file, that contains the texture coordinates.

Here’s an older article on converting data from Zwoptex to texture-coordinates.

Another, cool thing you can do with texture coordinates is to repeat them along a given bounds. First you need to set the correct parameters when you bind your texture.

1
2
gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_WRAP_S, gl.REPEAT);
gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_WRAP_T, gl.REPEAT);
Next, set the texture coordinates values greater than 1.

For example, this set of texture coordinates will repeat the entire texture twice on both X and Y axis.

1
2
3
4
5
6
var textureCoords = [
0.0, 2.0,
2.0, 2.0,
0.0, 0.0,
2.0, 0.0
];
Cool Guy

Yet, another interesting parameters are MIRRORED_REPEAT. It will mirror the repeat.

1
2
gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_WRAP_S, gl.MIRRORED_REPEAT);
gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_WRAP_T, gl.MIRRORED_REPEAT);
Cool Guy

And, finally we have GL_CLAMP_TO_EDGE, for scenarios where you don’t wish the texture to repeat at all but just extend all the way to the nearest edge.

1
2
gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_WRAP_S, gl.CLAMP_TO_EDGE);
gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_WRAP_T, gl.CLAMP_TO_EDGE);
Cool Guy

Now is the time for you to play with the code. Get the live demo here (if you’ve webgl enabled browser)

And the code
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
35
36
37
38
39
40
41
42
43
44
45
46
47
48
49
50
51
52
53
54
55
56
57
58
59
60
61
62
63
64
65
66
67
68
69
70
71
72
73
74
75
76
77
78
79
80
81
82
83
84
85
86
87
88
89
90
91
92
93
94
95
96
97
98
99
100
101
102
103
104
105
106
107
108
109
110
111
112
113
114
115
116
117
118
119
120
121
122
123
124
125
126
127
128
129
130
131
132
133
134
135
136
137
138
139
140
141
142
143
144
145
146
147
148
149
150
151
152
153
154
155
156
157
158
159
160
161
162
163
164
165
166
167
168
169
170
171
172
173
174
175
176
177
178
179
180
181
182
183
184
185
186
187
188
189
190
191
192
193
194
195
196
197
198
199
200
201
202
203
204
205
206
207
208
209
210
211
212
213
214
215
216
217
218
219
220
221
222
223
224
225
226
227
228
229
230
231
232
233
234
235
236
237
238
239
240
241
242
243
244
245
246
247
248
249
250
251
252
253
254
255
256
257
258
259
260
261
262
263
264
265
266
267
268
269
270
271
272
273
<html>
 
<head>
<title>Texture Coordinates</title>
<meta http-equiv="content-type" content="text/html; charset=ISO-8859-1">
 
<script type="text/javascript" src="glMatrix-0.9.5.min.js"></script>
<script type="text/javascript" src="webgl-utils.js"></script>
 
<script id="shader-fs" type="x-shader/x-fragment">
    precision mediump float;
 
    varying vec2 vTextureCoord;
 
    uniform sampler2D uSampler;
 
    void main(void) {
        gl_FragColor = texture2D(uSampler, vec2(vTextureCoord.s, vTextureCoord.t));
    }
</script>
 
<script id="shader-vs" type="x-shader/x-vertex">
    attribute vec3 aVertexPosition;
    attribute vec2 aTextureCoord;
 
    uniform mat4 uMVMatrix;
    uniform mat4 uPMatrix;
 
    varying vec2 vTextureCoord;
 
 
    void main(void) {
        gl_Position = uPMatrix * uMVMatrix * vec4(aVertexPosition, 1.0);
        vTextureCoord = aTextureCoord;
    }
</script>
 
 
<script type="text/javascript">
 
    var gl;
 
    function initGL(canvas) {
        try {
            gl = canvas.getContext("experimental-webgl");
            gl.viewportWidth = canvas.width;
            gl.viewportHeight = canvas.height;
        } catch (e) {
        }
        if (!gl) {
            alert("Could not initialise WebGL, sorry :-(");
        }
    }
 
 
    function getShader(gl, id) {
        var shaderScript = document.getElementById(id);
        if (!shaderScript) {
            return null;
        }
 
        var str = "";
        var k = shaderScript.firstChild;
        while (k) {
            if (k.nodeType == 3) {
                str += k.textContent;
            }
            k = k.nextSibling;
        }
 
        var shader;
        if (shaderScript.type == "x-shader/x-fragment") {
            shader = gl.createShader(gl.FRAGMENT_SHADER);
        } else if (shaderScript.type == "x-shader/x-vertex") {
            shader = gl.createShader(gl.VERTEX_SHADER);
        } else {
            return null;
        }
 
        gl.shaderSource(shader, str);
        gl.compileShader(shader);
 
        if (!gl.getShaderParameter(shader, gl.COMPILE_STATUS)) {
            alert(gl.getShaderInfoLog(shader));
            return null;
        }
 
        return shader;
    }
 
 
    var shaderProgram;
 
    function initShaders() {
        var fragmentShader = getShader(gl, "shader-fs");
        var vertexShader = getShader(gl, "shader-vs");
 
        shaderProgram = gl.createProgram();
        gl.attachShader(shaderProgram, vertexShader);
        gl.attachShader(shaderProgram, fragmentShader);
        gl.linkProgram(shaderProgram);
 
        if (!gl.getProgramParameter(shaderProgram, gl.LINK_STATUS)) {
            alert("Could not initialise shaders");
        }
 
        gl.useProgram(shaderProgram);
 
        shaderProgram.vertexPositionAttribute = gl.getAttribLocation(shaderProgram, "aVertexPosition");
        gl.enableVertexAttribArray(shaderProgram.vertexPositionAttribute);
 
        shaderProgram.textureCoordAttribute = gl.getAttribLocation(shaderProgram, "aTextureCoord");
        gl.enableVertexAttribArray(shaderProgram.textureCoordAttribute);
 
        shaderProgram.pMatrixUniform = gl.getUniformLocation(shaderProgram, "uPMatrix");
        shaderProgram.mvMatrixUniform = gl.getUniformLocation(shaderProgram, "uMVMatrix");
        shaderProgram.samplerUniform = gl.getUniformLocation(shaderProgram, "uSampler");
    }
 
    var mvMatrix = mat4.create();
    var mvMatrixStack = [];
    var pMatrix = mat4.create();
 
    function mvPushMatrix() {
        var copy = mat4.create();
        mat4.set(mvMatrix, copy);
        mvMatrixStack.push(copy);
    }
 
    function mvPopMatrix() {
        if (mvMatrixStack.length == 0) {
            throw "Invalid popMatrix!";
        }
        mvMatrix = mvMatrixStack.pop();
    }
 
 
    function setMatrixUniforms() {
        gl.uniformMatrix4fv(shaderProgram.pMatrixUniform, false, pMatrix);
        gl.uniformMatrix4fv(shaderProgram.mvMatrixUniform, false, mvMatrix);
    }
 
 
    function degToRad(degrees) {
        return degrees * Math.PI / 180;
    }
 
    var positionBuffer;
    var textureCoordBuffer;
    
    function initBuffers() {
      positionBuffer = gl.createBuffer();
        gl.bindBuffer(gl.ARRAY_BUFFER, positionBuffer);
        vertices = [
            -1.0, -1.0,  1.0,
             1.0, -1.0,  1.0,
            -1.0,  1.0,  1.0,
            1.0,  1.0,  1.0
        ];
        gl.bufferData(gl.ARRAY_BUFFER, new Float32Array(vertices), gl.STATIC_DRAW);
        positionBuffer.itemSize = 3;
        positionBuffer.numItems = 4;
 
        textureCoordBuffer = gl.createBuffer();
        gl.bindBuffer(gl.ARRAY_BUFFER, textureCoordBuffer);
        var textureCoords = [
          0.0, 1.0,
          1.0, 1.0,
          0.0, 0.0,
          1.0, 0.0
        ];
        gl.bufferData(gl.ARRAY_BUFFER, new Float32Array(textureCoords), gl.STATIC_DRAW);
        textureCoordBuffer.itemSize = 2;
        textureCoordBuffer.numItems = 4;
    }
 
    function handleLoadedTexture(texture) {
        gl.bindTexture(gl.TEXTURE_2D, texture);
        //gl.pixelStorei(gl.UNPACK_FLIP_Y_WEBGL, true);
        gl.texImage2D(gl.TEXTURE_2D, 0, gl.RGBA, gl.RGBA, gl.UNSIGNED_BYTE, texture.image);
        gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_MAG_FILTER, gl.NEAREST);
        gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_MIN_FILTER, gl.NEAREST);
		gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_WRAP_S, gl.REPEAT);
        gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_WRAP_T, gl.REPEAT);
        gl.bindTexture(gl.TEXTURE_2D, null);
    }
 
 
    var neheTexture;
 
    function initTexture() {
        neheTexture = gl.createTexture();
        neheTexture.image = new Image();
        neheTexture.image.onload = function () {
            handleLoadedTexture(neheTexture)
        }
 
        neheTexture.image.src = "images/coolguy.png";
    }
 
    function drawScene() {
        gl.viewport(0, 0, gl.viewportWidth, gl.viewportHeight);
        gl.clear(gl.COLOR_BUFFER_BIT | gl.DEPTH_BUFFER_BIT);
 
        mat4.perspective(45, gl.viewportWidth / gl.viewportHeight, 0.1, 100.0, pMatrix);
 
        mat4.identity(mvMatrix);
 
        mat4.translate(mvMatrix, [0.0, 0.0, -5.0]);
 
        gl.bindBuffer(gl.ARRAY_BUFFER, positionBuffer);
        gl.vertexAttribPointer(shaderProgram.vertexPositionAttribute, positionBuffer.itemSize, gl.FLOAT, false, 0, 0);
 
        gl.bindBuffer(gl.ARRAY_BUFFER, textureCoordBuffer);
        gl.vertexAttribPointer(shaderProgram.textureCoordAttribute, textureCoordBuffer.itemSize, gl.FLOAT, false, 0, 0);
 
        gl.activeTexture(gl.TEXTURE0);
        gl.bindTexture(gl.TEXTURE_2D, neheTexture);
        gl.uniform1i(shaderProgram.samplerUniform, 0);
 
        setMatrixUniforms();
        gl.drawArrays(gl.TRIANGLE_STRIP, 0, positionBuffer.numItems);
    }
 
 
    var lastTime = 0;
 
    function animate() {
        var timeNow = new Date().getTime();
        if (lastTime != 0) {
            var elapsed = timeNow - lastTime;
 
            xRot += (90 * elapsed) / 1000.0;
            yRot += (90 * elapsed) / 1000.0;
            zRot += (90 * elapsed) / 1000.0;
        }
        lastTime = timeNow;
    }
 
 
    function tick() {
        requestAnimFrame(tick);
        drawScene();
   //     animate();
    }
 
 
    function webGLStart() {
        var canvas = document.getElementById("lesson05-canvas");
        initGL(canvas);
        initShaders();
        initBuffers();
        initTexture();
 
        gl.clearColor(0.5, 0.5, 0.5, 1.0);
        gl.enable(gl.DEPTH_TEST);
 
//        drawScene();
        tick();
    }
 
 
</script>
 
 
</head>
 
 
<body onload="webGLStart();">
    <canvas id="lesson05-canvas" style="border: none;" width="500" height="500"></canvas>
</body>
 
</html>
view rawgistfile1.html hosted with ❤ by GitHub
Posted in OpenGL, _dev	| Tagged coordinates, opengl, texture, WebGL	| 0 Comments
opengl-series: Changes with 06_diffuse_lighting
Posted on March 24, 2013 by chunkyguy
This post is in continuation with the opengl-series. 
visit here for notes for sample code up to 05_asset_instance

With the diffuse lighting sample code, we have to make some specific changes to make it run on iOS.

I’m assuming following configuration:

1
2
3
4
OpenGL version: OpenGL ES 2.0 IMGSGX535-73.16.1
GLSL version: OpenGL ES GLSL ES 1.0
Vendor: Imagination Technologies
Renderer: PowerVR SGX 535
Reference for GLSL 1.0.17

Man page for GLSL (for comparision between different GLSL versions)

1. Fragment Shader

Apart from all the changes we have made to the frag shader so far, like, using precision, using attribute, varying keywords instead of in out. We will have to make some more adjustments.

1
mat3 normalMatrix = transpose(inverse(mat3(model)));
The transpose function is available since version 1.20, and the  inverse function is available since version 1.40, so we can’t use it. Instead we have two options. A. Calculate the normalMatrix from the C++ code and pass it as a uniform to the frag shader. B. Write our own transpose and inverse functions.

Although, with option B, we shall have consistent code, but since the code in frag shader is called for each pixel on the screen, I think it would be inefficient. Also, we can not ignore the fact that we already have a time tested maths library, which would be better than our own code.

In RenderInstance, I’ve added this line to use glm compute the normalMatrix for us

1
shaders->setUniform("normalMatrix", glm::transpose(glm::inverse(glm::mat3(inst.transform))));
Next, we have the usage of function clamp

1
brightness = clamp(brightness, 0, 1);
With version 1.0 we don’t have implicit conversions, so instead of 0 and 1, we have to explicitly say 0.0 and 1.0.

And, now we should be able to run our code.


First run
2. Gestures

Next big change is with gestures. If you run the Desktop app, you can observe that after moving the camera around and pressing the ’1′ key updates the light position to current camera position.

With the 05_asset_instance we used gestures as direct mapping with keyboard, but from this sample code I’ve tried to utilize the touch based gestures more, and also made them more natural to the touch based gestures.

2.1 Long press WASD

The WASD gestures are now not directly mapped with the desktop version. Instead, W, A, S, and D when tapped for a longer duration now move the camera Up, Left, Down and Right respectively.

2.2 Short press WASD

When tapped on WASD zones for a quick duration, launches a different action. In current case W, A, S, D maps to Update the light to current position; Change light to red; Change light to white; Change light to Green.


2.3 Pinch to zoom

The zooming gesture, that were previously mapped to W and S key are now mapped to pinch gestures.


2.4 Drag

The mouse movement is mapped to dragging on the screen.


Posted in OpenGL, _general_things, _iOS	| 0 Comments
opengl-series iOS migration notes
Posted on March 17, 2013 by chunkyguy
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