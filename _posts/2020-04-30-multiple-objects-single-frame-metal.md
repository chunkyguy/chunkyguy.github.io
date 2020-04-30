---
layout: post
title:  "Multiple objects in single frame with Metal"
date:   2020-04-30 20:20:54 +0200
categories: metal
published: true
---

It's been [a long time](https://whackylabs.com/metal/2019/01/19/hello-metal/) since I touched Metal. [Recently](https://twitter.com/chunkyguy/status/1254785138040307721?s=20), I've been poking at Metal again. This brought me to an interesting challenge. So far I've been rendering single objects and when I try to render more than one item I face an interesting problem. Let's see if I can fix that.

# Problem

So, I've a teapot and a floor that I'm able to render independently.

![Floor]({{ site.url }}/assets/draw-two-things/floor.png)
![Teapot]({{ site.url }}/assets/draw-two-things/teapot.png)

Now I want to render them both together. Sounds simple right?

# Take 1: A simple loop

This is how my shader types looks like, nothing fancy in there:

```cpp
struct VertexIn {
  float4 position;
  float3 normal;
};

struct Uniforms {
  float4x4 mvMatrix;
  float3x3 nMatrix;
  float4x4 mvpMatrix;
};

vertex VertexOut vert_main(
  const device VertexIn *vertices [[buffer(0)]],
  constant Uniforms *uniforms [[buffer(1)]],
  uint vid [[vertex_id]]
) { ... }
```

And this is the pseudo algorithm:

```objc
// Renderer.m
- (void)renderScene
{
  for (WLActor *actor in scene.actors) {
    id<MTLCommandBuffer> cmdBuf; // set up
    id<MTLRenderCommandEncoder> command; // set up

    [actor render:command];

    [command endEncoding];
    [cmdBuf presentDrawable:drawable];
    [cmdBuf commit];
  }
}
```

```objc
// Actor.m
@interface WLActor ()
{
  id<MTLBuffer> _uniforms;
  id<MTLBuffer> _vertexBuffer;
  id<MTLBuffer> _indexBuffer;
}
@end

- (void)render:(id<MTLRenderCommandEncoder>)command
{
  // bind vertex, uniform, index data ...
  [command setVertexBuffer:_vertexBuffer offset:0 atIndex:0];
  [command setVertexBuffer:_uniforms offset:0 atIndex:1];
  [command drawIndexedPrimitives:MTLPrimitiveTypeTriangle
                      indexCount:[_indexBuffer length]/sizeof(WLInt16)
                       indexType:MTLIndexTypeUInt16
                     indexBuffer:_indexBuffer
               indexBufferOffset:0];
}
```

When running this, I get a strange flickering

![Failure 01]({{ site.url }}/assets/draw-two-things/failure_01.gif)

Turns out with Metal you need to wait for the drawing to finish before you can submit a new draw command. In simpler terms, we can not draw multiple times in a simple for loop, we need a blocking mechanism. I've seen the sample code trying to do this with a semaphore. Let's try that.

# Take 2: Asynchronous drawing with semaphores

```objc
// Renderer.m
- (void)renderScene
{
  dispatch_semaphore_wait(_drawSemaphore, DISPATCH_TIME_FOREVER);
  _drawIndex = (_drawIndex + 1) % scene.actors.count;

  id<MTLCommandBuffer> cmdBuf = [_cmdQueue commandBuffer];
  __block dispatch_semaphore_t sema = _drawSemaphore;
  [cmdBuf addCompletedHandler:^(id<MTLCommandBuffer> buffer) {
    dispatch_semaphore_signal(sema);
  }];

  id<MTLRenderCommandEncoder> command; // set up
  [self configureRenderCommand:command];
  [command setRenderPipelineState:_pipeline];

  WLActor *actor = [scene.actors objectAtIndex:_drawIndex];
  [actor render:command camera:scene.camera];

  [command endEncoding];
  [cmdBuf presentDrawable:drawable];
  [cmdBuf commit];
}
```

![Failure 02]({{ site.url }}/assets/draw-two-things/failure_02.gif)

Okay. The problem seems that every draw wipes the drawing surface clean. Can we try getting rid of that?

# Take 3: Overlapped drawing

This is how I'm currently creating the render pass. The `texture` here is coming from the `CAMetalDrawable`, which is our screen.

```objc
- (MTLRenderPassDescriptor *)renderPassDescWithTexture:(id<MTLTexture>)texture
{
  MTLRenderPassDescriptor *passDesc = [MTLRenderPassDescriptor renderPassDescriptor];
  passDesc.colorAttachments[0].texture = texture;
  passDesc.colorAttachments[0].loadAction = MTLLoadActionClear;
  passDesc.colorAttachments[0].storeAction = MTLStoreActionStore;
  passDesc.colorAttachments[0].clearColor = MTLClearColorMake(0.1, 0.1, 0.1, 1);
  passDesc.depthAttachment.texture = _depthTexture;
  passDesc.depthAttachment.clearDepth = 1.0f;
  passDesc.depthAttachment.loadAction = MTLLoadActionClear;
  passDesc.depthAttachment.storeAction = MTLStoreActionDontCare;
  return passDesc;
}
```

Here if I change the `loadAction` to `MTLLoadActionLoad`, I guess it should preserve the last frame

```objc
passDesc.colorAttachments[0].loadAction = MTLLoadActionLoad;
```

Result:

![Dirty draw]({{ site.url }}/assets/draw-two-things/draw-dirty-buffer.png)

This look better. Although I should mention that there's still a flicker mainly in the beginning. But the bigger problem is the red color background. I guess that is because we never clear the screen, so whatever color is there originally stays there forever. Another problem appears when the teapot starts moving, since we are not clearing the screen, the images just keep adding up.

![Additive images]({{ site.url }}/assets/draw-two-things/additive.png)

Can we fix this with the classic *if first time do this, else do this* flow? Let's see.

# Take 4: Clear and Draw

 Since we need to clear the screen after all the objects have been drawn to avoid overlapping. We need something like:

```objc
  for (WLActor *actor in scene.actors) {
    [actor render];
  }
  clearScreen();
```

But this seems like a problematic already. These commands are executed one after the other. So if our last command is clear, it would wipe the entire screen clear. Thinking about it as how a painter would paint. First they would clear what ever is on the canvas, then draw the background and then the foreground. So, if our `actors` are in draw order we need something like:

```objc
  clearScreen();
  for (WLActor *actor in scene.actors) {
    [actor render];
  }
```

Also this reveals another interesting insight. A `MTLCommandBuffer` is designed to render multiple draw commands. So, looking at our target pseudo code, each `[actor render]` is basically just a draw command. So far we have been trying to redundantly waste resources and duplicate the work of `MTLCommandBuffer` with our semaphore, and probably even made the code unnecessarily complicated. What we actually need is to create a command buffer, add draw commands to it and submit.

```objc
- (void)render
{
  id<MTLCommandBuffer> cmdBuf = [_cmdQueue commandBuffer];
  for (NSInteger i = 0; i < N; ++i) {
    id<MTLRenderCommandEncoder> command = // make
    [command endEncoding];
  }
  [cmdBuf presentDrawable:drawable];
  [cmdBuf commit];
}

```

So let's try implementing this. We can loop our draw command one extra step than the `actors.count` just for clearing the screen.

```objc
- (void)render
{
  id<MTLCommandBuffer> cmdBuf = [_cmdQueue commandBuffer];
  for (NSInteger i = 0; i < (scene.actors.count + 1); ++i) {

    MTLLoadAction loadAction = MTLLoadActionDontCare;
    WLActor *actor = nil;
    if (i == 0) {
      loadAction = MTLLoadActionClear;
    } else {
      actor = [scene.actors objectAtIndex:i - 1];
    }
    id<MTLRenderCommandEncoder> command = [self renderCommandWithTexture:texture
                                                              loadAction:loadAction
                                                                  cmdBuf:cmdBuf];
    [actor render:command camera:scene.camera];
    [command endEncoding];
  }
  [cmdBuf presentDrawable:drawable];
  [cmdBuf commit];
}
```

Result:

![Dirty draw]({{ site.url }}/assets/draw-two-things/success.gif)

Now, thinking again, do we actually need an explicitly render command just for clearing the screen? We can also simply set the first draw with `MTLLoadActionClear`

```objc
for (NSInteger i = 0; i < scene.actors.count; ++i) {
    MTLLoadAction loadAction = (i == 0) ? MTLLoadActionClear: MTLLoadActionLoad;
    WLActor *actor = [scene.actors objectAtIndex:i];
    // draw ..
}
```

# Recap

So what did we learn from this?

1. We need a `MTLCommandBuffer` for a render pass.
1. Each render pass can have as many `MTLRenderCommandEncoder` render commands.
1. The first render command should clear the screen with `MTLLoadActionClear`.
1. The following render command need to draw on top of existing image with `MTLLoadActionLoad`.
1. You don't need semaphore based blocking unless your app is GPU bound.

This was a fun exercise. Let me know on [twitter](http://twitter.com/chunkyguy) if you have any questions.