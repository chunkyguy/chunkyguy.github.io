---
layout: post
title:  "Experiment 13: Shadow mapping"
date:   2014-02-02 23:28:54 +0530
categories: opengl
---

This article is part of the Experiments with OpenGL series

The source code is available at https://github.com/chunkyguy/EWOGL

Shadows add a great detail to the 3D scene. For instance, it’s really hard to judge how high the object really is above the floor without a shadow, or how far away is the light source from the object.


Shadow mapping is one of the trick to create shadows for our geometry.

The basic idea it to render the scene twice. First from the light’s point of view, and save the result in a texture. Next, render from the desired point of view and compare the result with the saved result. If the part of geometry being rendered falls under shadow region, don’t apply lighting to it.

During setup routine, we need 2 framebuffers. One for each rendering pass. The first framebuffer needs to have a depth renderbuffer, and a texture attached to it. The second framebuffer is your usual framebuffer with color and depth.

The first pass is very simple. Switch the point of view to the light’s view and just draw the scene as quick as possible to the framebuffer. You just need to pass the vertex positions and the model-view-projection matrix to take those positions from the object space to the clip space. The result is what some like to call as the Shadow map.

The shadow map is just the 1-channel texture where blacker the color means, more close it is to the light. While whiter the color means the farther it is from light. And, a totally white color means, that part is not visible to the light.

With black and white I’m assuming the depth range is from 0.0 to 1.0, which is the default range I guess.

The second pass is also not very difficult. We just need a matrix that will take our vertex positions from object space to the clip space from the light’s view. Let’s call it the shadow matrix.

Once, we have the shadow matrix, we just need to compare the compare between the z of stored depth map and the z of our vertex’s position.

Creating of shadow matrix is also not very difficult. I calculate it like this:

``` objc
    GLKMatrix4 basisMat = GLKMatrix4Make(0.5f, 0.0f, 0.0f, 0.0f,
                                         0.0f, 0.5f, 0.0f, 0.0f,
                                         0.0f, 0.0f, 0.5f, 0.0f,
                                         0.5f, 0.5f, 0.5f, 1.0f);
    GLKMatrix4 shadowMat = GLKMatrix4Multiply(basisMat,                  
                            GLKMatrix4Multiply(renderer->shadowProjection,  
                            GLKMatrix4Multiply(renderer->shadowView ,mMat)));
```
The basis matrix is just another way to move the range from [-1, 1] to [0, 1]. Or from depth coordinate system to texture coordinate system.

This is how it can be calculated:


GLKMatrix4 genShadowBasisMatrix()
{
  GLKMatrix4 m = GLKMatrix4Identity;
  
  /* Step 2: scale by 1/2 
   * [0, 2] -> [0, 1]
   */
  m = GLKMatrix4Scale(m, 0.5f, 0.5f, 0.5f);

  /* Step 1: Translate +1 
   * [-1, 1] -> [0, 2]
   */
  m = GLKMatrix4Translate(m, 1.0f, 1.0f, 1.0f);

  return m;
}
The first run was pretty cool on the simulator, but not so good on the device.

That is something I learned from my last experiment. To try out on the actual device as soon as possible.

I tried playing around with many configurations, like using 32 bits for depth precision, using bilinear filtering with depth map texture, changing the depth range from 1.0 to 0.0 and many other I don’t even remember. I should say, changing the depth range from the default [0, 1] to [1, 0] was the best.

If you’re working on something, where the shadows needs to be projected far away from the object. Like on a wall behind, try this in the first pass, while creating the depth map:


glDepthRangef(1.0f, 0.0f);
And, in the second pass don’t forget to switch back to


glDepthRangef(0.0f, 1.0f);
Of course, you also need to update your shader code, so that the result 1.0 now means fail case while 0.0 means the pass case. Or you could even update the GL_TEXTURE_COMPARE_FUNC to GL_GREATER.

But, for majority cases where the shadow is attached to the geometry. The experiment’s code should work fine.


Another bit of improvement, that can be added is using the PCF (Percentage Closer Filtering) method. It helps with the anti-aliasing at the shadow edges.

The way PCF works is, we read the result from surrounding textures and average them.
Here’s an example of the improvement you can expect.

Hosted by imgur.com

Another shadow mapping method, that I would like to try in the future is with random sampling. Where instead of fixed offsets, we pick random samples around the fragment. I’ve heard this creates the soft shadow effect.

Shadow mapping, is a field of many hit and trials. There doesn’t seems to be one solution fit all. There are so many parameters that can be configured in so many ways. Like, you can try with culling front face, or setting polygon offset to avoid z-fighting. In the end, you just need to find the solution that fits your current needs.

The code right now only works for OpenGL ES3, as I was using many things available only to GLSL 3.00. But, since the trick is very trivial, I would in the future upgrade to code to work with OpenGL ES 2 as well. I see no reasons why it shouldn’t work.
