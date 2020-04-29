---
layout: post
title:  "Hello Metal one more time"
date:   2019-01-19 20:26:00 +0200
categories: metal
published: true
---

If I were to say that I've never poked the Metal API, I would be lying. I've probably poked at the Metal API several times now since it was launched.

My feelings for Metal has been nothing less than a roller coster ride. When I first heard of it, I did not like it. At that time I was too much into OpenGL and the idealistic idea of one API for all. Until I saw the point made by Apple about Metal, that with Metal Apple would provide us developers more control over the GPU which now Apple was manufacturing in house. And then I did like it a bit, and I decided to take a look at the API at least once, and if I do not like it, I'll just ignore its exsistence and keeping using OpenGL.

That is when I realized that Metal was written in Objective C, and I was sad again. And it not due the fact that it was not in Swift which was not even a hype back then, I was sad that it was not in C or C++. Like I've said, I was very much in to the idea of one API rules for, so I was very much using C++ as much as possible for similar reasons.

Some time later Khronos people released Vulkan which would cover almost all non-Apple GPUs and the API was in C. I was super excited, more like a dream come true for me. The only issue was that I had zero devices where I could actually play with Vulkan. 

Later I learned about this thing called Molten VK which would provide the Vulkan API on top of Metal. Good, finally I can try Vulkan on my macOS right? No, it was still under progress and even after it were done, it was supposed to be a not-for-free framework. Still interesting to keep and eye on.

Meanwhile, Vulkan people and the Metal people were still rolling out version updates and here I was still stuck with OpenGL. One day I picked up my MacBook and started rendering the first triangle with Metal and ObjC. To be honest, I did like API a lot. I would have continued diving deeper if it were at least in C++.

Some time later, I finally got a Windows machine at home. The first thing I did after unboxing was to set up Vulkan, yay! That API felt very much like Metal with a bit more verbosity, maybe a bit too much verbose, but okay.

Apple had already announced Metal for macOS and I had plans about making something for macOS for a long time, and I got one more time interested in Metal. And since I was using Swift exclusively at work and did not mind working with ObjC for personal projects. At least with ObjC you can interact with C++.

Coming to present time. I was exploring different UIKit alternatives and I came across two which got me interested in Metal one more time. These are ComponentKit and Flutter. ComponentKit inspired me try writing something entirely in ObjC++, which had never ever occured to me. Although, I'm still confused where would I set the boundary, but I definately wanna try it just to get the feel. Flutter got me interested in Metal because of this ticket: [Flutter should use Metal instead of OpenGL on iOS](https://github.com/flutter/flutter/issues/18208). 

It is not exactly this ticket, but more like the idea that a lot of people are actually avoiding Metal, because of the obvious reasons I did, but Metal as an API in itself is really amazing from what I remember, and it makes total sense from Apple perspective to own their own GPU API as it would give them total freedom, and on top of Metal being ObjC has an advantage that it can exposed to Swift with zero hiccups. I mean it all makes sense, and to be honest I feel a bit bad for the team working on the Metal API.

So, today I'll try to render something on screen with Metal and get a feel of the API one more time. Here we go.

# Set up

I'll try to rewrite the hello metal one more time. I'll use swift because I also wanna see how easy is it these days developing Metal apps with swift. I do remember that one of selling point of Metal was to share model data between the app code (in ObjC) and metal shaders by using `simd` C type `struct`. Don't know how much of that would apply to swift.

# Hello Triangle

To have a triangle on screen, these are the steps we need:

1. Get the GPU
1. Allocate some memory buffer on the GPU and copy the triangle vertex data there.
1. Write a shader to read the vertex data from step above.
1. Set up the render pipeline with the shader.
1. Set up a command queue to send draw commands to the GPU.
1. Set up a command buffer where the actual commands will live.
1. Set up a command encoder to append commands to the command buffer.
1. Render!

# Set up device (GPU)

Getting the device is pretty simple actually.

``` swift
func setUp() {
    self.device = MTLCreateSystemDefaultDevice()
}
```

# Set up draw data

Metal supports two kind of resources: buffer and image data. Buffer is where we can put the vertex data and image data is where we can put the textures.

Writing this part of code with Swift is a bit awkward because of the ObjC bridge but still whatever works.

``` swift
let vertexData: [Float] = [
    0.0, 1.0, 0.0,
    -1.0, -1.0, 0.0,
    1.0, -1.0, 0.0
]
let length = vertexData.count * MemoryLayout.size(ofValue: vertexData[0])
vertexBuffer = device?.makeBuffer(bytes: vertexData, length: length)
```

# Write shader

Writing a metal shader is almost the same as GLSL. We need a vertex shader and fragment shader. 

For our use case, our vertex shader is very simple. It just reads an array of `float3` and returns a `float4` for each vertex of the geometry.

``` cpp
vertex float4 vsh_flat(const device packed_float3 *vertexArray [[ buffer(0) ]], unsigned int vid [[ vertex_id ]]) {
    return float4(vertexArray[vid], 1.0);
}
```

The fragment shader is even simpler. We simply return a white color for each pixel.
``` cpp
fragment half4 fsh_flat() {
    return half4(1.0);
}
```

# Set up pipeline
 
 With Metal we can compile all our shaders at build time. All the compiled shaders then can be accessed from the library at runtime.

 ``` swift
func setUp() {
    // ...
    let library = device?.makeDefaultLibrary()
    let vertShader = library?.makeFunction(name: "vsh_flat")
    let fragShader = library?.makeFunction(name: "fsh_flat")
    // ...
}
 ```

To create the render pipeline we just need to plug in the shaders and we are done.

``` swift
func setUp() {
    // ...
    let pipelineDescriptor = MTLRenderPipelineDescriptor()
    pipelineDescriptor.vertexFunction = vertShader
    pipelineDescriptor.fragmentFunction = fragShader
    pipelineDescriptor.colorAttachments[0].pixelFormat = pixelFormat
    pipeline = device.flatMap { try? $0.makeRenderPipelineState(descriptor: pipelineDescriptor) }
    // ...
}
```

# Set up command queue

Next, we get the command queue where we would submit all our command buffers. 

``` swift
private func setUp() {
    // ...
    commandQueue = device?.makeCommandQueue()
}
```

# Set up Command buffer

Think of one command buffer as one drawing frame. Which could even have multiple render passes if required. 

After a buffer is submitted to the queue, we can either wait for the GPU to finish rendering, or use a callback to get notification.

``` swift
func render (/* ... */, drawable: MTLDrawable) {        
    let commandBuffer = commandQueue?.makeCommandBuffer()
    // ...
    commandBuffer?.present(drawable)
    commandBuffer?.commit()
    commandBuffer?.waitUntilCompleted()
}
```

# Render pass

Metal way of making things like the render encoder is to use a descriptor object where all the configuration values can be set. This is very similar to Vulkan, but looks less verbose due to the C/C++ [aggregate initialization](https://en.cppreference.com/w/cpp/language/aggregate_initialization), also one of the  reasons why [ComponentKit uses C++](https://componentkit.org/docs/why-cpp.html).

``` swift
func render(drawableTexture: MTLTexture, /* ... */ ) {
    // ...
    let renderPassDescriptor = MTLRenderPassDescriptor()
    renderPassDescriptor.colorAttachments[0].texture = drawableTexture
    renderPassDescriptor.colorAttachments[0].loadAction = .clear
    renderPassDescriptor.colorAttachments[0].clearColor = MTLClearColor(red: 0.5, green: 0.5, blue: 0.5, alpha: 1.0)
    // ...
}
```

Commands can be submitted only via a command encoder. We need the `MTLRenderCommandEncoder` to encode draw commands. Only one encoder can be active at a time for a command buffer.

``` swift
func render(drawableTexture: MTLTexture, /* ... */ ) {
    // ...
    let renderEncoder = commandBuffer?.makeRenderCommandEncoder(descriptor: renderPassDescriptor!)
    renderEncoder?.setRenderPipelineState(pipeline!)
    // ...
}
```

Next step is submit the actual draw calls and close the encoder.

``` swift
func render(drawableTexture: MTLTexture) {
    // ...
    renderEncoder?.setVertexBuffer(vertexBuffer, offset: 0, index: 0)
    renderEncoder?.drawPrimitives(type: .triangle, vertexStart: 0, vertexCount: 3)
    renderEncoder?.endEncoding()
    // ...
}
```

# Set up drawable

Final step is bind the renderer with a drawing surface, in our case a `CAMetalLayer` or a `MTKView`. Here we simply call the `setUp` on the `RenderingEngine` at appropriate times.

``` swift
class Renderer {
    private let metalView: MTKView
    private let renderingEngine = RenderingEngine()
    
    init(metalView: MTKView) {
        self.metalView = metalView
    }
    
    func setUp() {
        renderingEngine.setUp()
        
        metalView.device = renderingEngine.device
        
        let metalLayer = metalView.layer as? CAMetalLayer
        assert(metalLayer != nil)
        
        metalLayer?.pixelFormat = renderingEngine.pixelFormat
        
        metalView.framebufferOnly = true
    }

    // ...    
}

class GameViewController: NSViewController {

    var renderer: Renderer?

    override func viewDidLoad() {
        super.viewDidLoad()
        
        let metalView = view as? MTKView
        assert(metalView != nil)
        
        renderer = Renderer(metalView: metalView!)
        assert(renderer != nil)
        
        renderer?.setUp()
    }

    // ...
}
```

# Render

After everything is in place, we can call the `render` to see our beautiful triangle on screen.

``` swift 
class Renderer {

    // ...
    
    func render() {
        let metalLayer = metalView.layer as? CAMetalLayer
        assert(metalLayer != nil)

        let drawable = metalLayer?.nextDrawable()
        assert(drawable != nil)
        
        renderingEngine.render(drawableTexture: drawable!.texture, drawable: drawable!)
    }
}

class GameViewController: NSViewController {

    // ...

    override func viewDidAppear() {
        super.viewDidAppear()
        
        renderer?.render()
    }
}
```

# Benefit?

![Hello Metal Triangle]({{ site.url }}/assets/metal/metal_triangle.png)

The code is also available on Github [github.com/chunkyguy/try-metal](https://github.com/chunkyguy/try-metal/tree/master/01_HelloMetal)