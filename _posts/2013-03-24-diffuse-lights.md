---
layout: post
title:  "opengl-series: Changes with 06_diffuse_lighting"
date:   2013-03-24 23:28:54 +0530
categories: opengl ios
published: false
---

This post is in continuation with the opengl-series. 
visit here for notes for sample code up to 05_asset_instance

With the diffuse lighting sample code, we have to make some specific changes to make it run on iOS.

I’m assuming following configuration:

```
OpenGL version: OpenGL ES 2.0 IMGSGX535-73.16.1
GLSL version: OpenGL ES GLSL ES 1.0
Vendor: Imagination Technologies
Renderer: PowerVR SGX 535
Reference for GLSL 1.0.17
```

Man page for GLSL (for comparision between different GLSL versions)

## Fragment Shader

Apart from all the changes we have made to the frag shader so far, like, using precision, using attribute, varying keywords instead of in out. We will have to make some more adjustments.
``` cpp
mat3 normalMatrix = transpose(inverse(mat3(model)));
```

The transpose function is available since version 1.20, and the  inverse function is available since version 1.40, so we can’t use it. Instead we have two options. A. Calculate the normalMatrix from the C++ code and pass it as a uniform to the frag shader. B. Write our own transpose and inverse functions.

Although, with option B, we shall have consistent code, but since the code in frag shader is called for each pixel on the screen, I think it would be inefficient. Also, we can not ignore the fact that we already have a time tested maths library, which would be better than our own code.

In RenderInstance, I’ve added this line to use glm compute the normalMatrix for us
``` cpp
shaders->setUniform("normalMatrix", glm::transpose(glm::inverse(glm::mat3(inst.transform))));
```

Next, we have the usage of function clamp
``` cpp
brightness = clamp(brightness, 0, 1);
```

With version 1.0 we don’t have implicit conversions, so instead of 0 and 1, we have to explicitly say 0.0 and 1.0.

And, now we should be able to run our code.


## Gestures

Next big change is with gestures. If you run the Desktop app, you can observe that after moving the camera around and pressing the ’1′ key updates the light position to current camera position.

With the 05_asset_instance we used gestures as direct mapping with keyboard, but from this sample code I’ve tried to utilize the touch based gestures more, and also made them more natural to the touch based gestures.

### Long press WASD

The WASD gestures are now not directly mapped with the desktop version. Instead, W, A, S, and D when tapped for a longer duration now move the camera Up, Left, Down and Right respectively.

### Short press WASD

When tapped on WASD zones for a quick duration, launches a different action. In current case W, A, S, D maps to Update the light to current position; Change light to red; Change light to white; Change light to Green.


### Pinch to zoom

The zooming gesture, that were previously mapped to W and S key are now mapped to pinch gestures.


### Drag

The mouse movement is mapped to dragging on the screen.
