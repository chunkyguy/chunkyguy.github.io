---
layout: post
title:  "OpenGL Texture Coordinates"
date:   2013-07-02 23:28:54 +0530
categories: opengl ios
---

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
``` js
var textureCoords = [
 
0.0, 1.0,  //A
1.0, 1.0,  //B
0.0, 0.0,  //C
1.0, 0.0  //D
 
];
```

Cool Guy

Now if we wish to render a portion of the original texture, we can simply map it to points we wish, keeping in mind the texture coordinates above.

Let’s say we wish to render only the first half of the texture, we can set texture coordinates as:
``` js
var textureCoords = [
0.0, 1.0,
0.5, 1.0,
0.0, 0.0,
0.5, 0.0
];
```

Cool Guy

As you can see we are just rendering half the width. Similarly, we can render only the next half by:
``` js
var textureCoords = [
0.5, 1.0,
1.0, 1.0,
0.5, 0.0,
1.0, 0.0
];
```

Cool Guy

What if we just wish to render the logo on our cool guy’s shirt. Try a few experimenting around, you can find the texture coordinates, just keep in mind the way coordinate-system works for textures.

One way to guess the texture coordinates is assume the range of x-min, x-max and y-min, y-max. Like for above requirement, after few trial-n-errors I calculated the coordinates to be in range:
``` js
x-min = 0.1
x-max = 0.43
y-min = 0.4
y-max = 0.57
```

So, the texture coordinates becomes:
``` js
var textureCoords = [
0.1, 0.57,
0.43, 0.57,
0.1, 0.4,
0.43, 0.4
];

```
Cool Guy

Now, in real life you seldom need to calculate the points yourself. There are plenty of applications that do that for you. One of my favorite is Zwoptex. You just need to drop your images into these sprite-sheet maker apps, and they give you a single texture, with an associated data file, that contains the texture coordinates.

Here’s an older article on converting data from Zwoptex to texture-coordinates.

Another, cool thing you can do with texture coordinates is to repeat them along a given bounds. First you need to set the correct parameters when you bind your texture.
``` js
gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_WRAP_S, gl.REPEAT);
gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_WRAP_T, gl.REPEAT);
```

Next, set the texture coordinates values greater than 1.

For example, this set of texture coordinates will repeat the entire texture twice on both X and Y axis.
``` js
var textureCoords = [
0.0, 2.0,
2.0, 2.0,
0.0, 0.0,
2.0, 0.0
];
```

Cool Guy

Yet, another interesting parameters are MIRRORED_REPEAT. It will mirror the repeat.
``` js
gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_WRAP_S, gl.MIRRORED_REPEAT);
gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_WRAP_T, gl.MIRRORED_REPEAT);
```

Cool Guy

And, finally we have GL_CLAMP_TO_EDGE, for scenarios where you don’t wish the texture to repeat at all but just extend all the way to the nearest edge.
``` js
gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_WRAP_S, gl.CLAMP_TO_EDGE);
gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_WRAP_T, gl.CLAMP_TO_EDGE);
```

Cool Guy

Now is the time for you to play with the code. Get the live demo here (if you’ve webgl enabled browser)

And the code

``` js
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
```