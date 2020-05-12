---
layout: post
title:  "Types sharing between Metal and App"
date:   2019-01-19 22:43:00 +0200
categories: metal cpp
published: true
---

One of the best things about Metal is the [Metal Shading Language](https://developer.apple.com/library/archive/documentation/Metal/Reference/MetalShadingLanguageGuide/Introduction/Introduction.html?language=objc#//apple_ref/doc/uid/TP40014364) which is a based on C++. What this means is that we can share data structures between the shader and app code. There are a few cases that we have to look out for though, which is what this post is about. 

# Using C++

Lets say we want to define a data structure that represents vertex data:

```cpp
// common.h

#include <simd/simd.h>

typedef struct
{
    simd::float4 position;
    simd::float4 color;
} Vertex;
```

The first problem we would have to tackle is that even though the simd types are the same in both C++ code and the Metal code but in C++ the types are part of `simd` namespace.

One way to fix this issue is to add `using namespace simd` when calling from C++. 

```cpp
// common.h

#ifdef __METAL_VERSION__
#else
#include <simd/simd.h>
using namespace simd;
#endif

typedef struct
{
    float4 position;
    float4 color;
} Vertex;
```

Now we can reuse the data structure in our C++ code, which looks even better thanks to the [C99 style member initialization](https://en.cppreference.com/w/c/language/struct_initialization).

```cpp
// renderer.cpp

#include "common.h"

 Vertex data[] = {
    {
        .position = { 0.0f, 1.0f, 0.0f, 1.0f },
        .color = { 1.0f, 0.0f, 0.0f, 1.0f }
    },
    {
        .position = { -1.0f, -1.0f, 0.0f, 1.0f },
        .color = { 0.0f, 1.0f, 0.0f, 1.0f }
    },
    {
        .position = { 1.0f, -1.0f, 0.0f, 1.0f },
        .color = { 0.0f, 0.0f, 1.0f, 1.0f }
    }
};
```

And on the shading side

```cpp
// shader.metal

#include <metal_stdlib>
#include "common.h"

using namespace metal;

vertex Vertex vertexShader(device Vertex *data [[buffer(0)]], uint vid [[vertex_id]])
{
    return data[vid];
}


fragment float4 fragShader(Vertex in [[stage_in]])
{
    return in.color;
}
```

This has a problem. Since we are returning a `struct` from the vertex shader function, we need to have an element with attribute `[[position]]`. From the official docs:

> If the return type of a vertex function is not void, it must include the vertex position. If the vertex return type is float4, then it always refers to the vertex position, and the [[position]] attribute must not be specified. If the vertex return type is a struct, it must include an element declared with the [[position]] attribute.

The fix is actually pretty simple, we can simply add the missing attribute to the `Vertex`

```cpp
// common.h

typedef struct
{
    float4 position [[position]];
    float4 color;
} Vertex;
```

This should solve most of the errors except that compiler might emit a warning when compiling the C++ code:

> Unknown attribute 'position' ignored

This could again be fixed with our beautiful preprocessor check

```cpp
// common.h

#ifdef __METAL_VERSION__
#define ATTRIB_POSITION [[position]]
#else
#include <simd/simd.h>
#define ATTRIB_POSITION
using namespace simd;
#endif

typedef struct
{
    float4 position ATTRIB_POSITION;
    float4 color;
} Vertex;
```

Not the prettiest looking code out there, but gets the job done. 

# Using Objective-C

With Objective-C, the same trick can be applied, since for all `simd` types a C variant is available.

```cpp
// common.h

#ifdef __METAL_VERSION__
#define ATTRIB_POSITION [[position]]
#else
#define ATTRIB_POSITION
#endif

typedef struct _Vertex
{
    vector_float4 position ATTRIB_POSITION;
    vector_float4 color;
} Vertex;
```

Nothing changes on the Metal side.