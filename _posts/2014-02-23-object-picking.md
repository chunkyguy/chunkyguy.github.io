---
layout: post
title:  "Experiment 14: Object Picking"
date:   2014-02-23 23:28:54 +0530
categories: opengl
---

This article is part of the Experiments with OpenGL series

The source code is available at https://github.com/chunkyguy/EWOGL

Every game at some point requires the user to interact with the 3D world. Probably in the form of a mouse click event firing bullets at targets or like a touch drag on a mobile device to rotate the view on the screen. This is called object picking, because most probably we are picking objects in our 3D space using mouse or touch screen from the device space.

How I like to picture this problem is, that we have a point in the device coordinate system and we would like to ray trace it back to some relevant 3D coordinate system, like the world space or probably the model space.

In the fixed function pipeline days there was a gluUnProject function which helped in this ray tracing, and still most developers like to use it. Even the GLKit provides a similar GLKMathUnproject function. Basically, this function requires a modelview matrix, a projection matrix, a viewport or the device coordinates and a 3D vector in device coordinate and it returns back the vector transformed to the object space.

With modern OpenGL, we don’t need to use that function. First of all it requires us to provide all the matrices seperately, and secondly, it returns the vector in model space, but our needs could be different, maybe we need it in some other space. Or we don’t need the vector at all, rather, just a bool flag to indicate whether the object can be picked or not.

At the core, we just need two things: a way to transform our touch point from device space to the clip space and an inverse of model-view-projection matrix to transform it from the clip space to the object’s model space.

Lets assume we have a scene with two rotating cubes and we need to handle the touch events on the screen and test if the touch point a collision with any of the objects in the scene. If the collision does happens, we just change the color of that cube.


First problem is to convert the touch point from iPhone’s coordinates system to the OpenGL’s coordinate system. Which is easy as it just means we need to flip the y-coordinate

``` objc
  /* calculate window size */
  GLKVector2 winSize = GLKVector2Make(CGRectGetWidth(self.view.bounds), CGRectGetHeight(self.view.bounds));

  /* touch point in window space */
  GLKVector2 point = GLKVector2Make(touch.x, winSize.y-touch.y);
Next, we need to transform the point to a normalized device coordinates space, or to range of [-1, 1]


  /* touch point in viewport space */
  GLKVector2 pointNDC = GLKVector2SubtractScalar(GLKVector2MultiplyScalar(GLKVector2Divide(point, winSize), 2.0f), 1.0f)
```

Next question that we need to tackle is, how to calculate a 3D vector from a device space, as the device is a 2D space. We need to remember that after the projection transformations are applied the depth of the scene is reduces to the range of [-1, 1].
So, we can use this fact and calculate the 3D positions at both the depth locations.

``` objc
  /* touch point in 3D for both near and far planes */
  GLKVector4 win[2];
  win[0] = GLKVector4Make(pointNDC.x, pointNDC.y, -1.0f, 1.0f);
  win[1] = GLKVector4Make(pointNDC.x, pointNDC.y, 1.0f, 1.0f);
```

Then, we need to calculate the inverse of the model-view-projection matrix

``` objc
  GLKMatrix4 invMVP = GLKMatrix4Invert(mvp, &success);
```

Now, we have all the things required to trace our ray
``` objc
  /* ray at near and far plane in the object space */
  GLKVector4 ray[2];
  ray[0] = GLKMatrix4MultiplyVector4(invMVP, win[0]);
  ray[1] = GLKMatrix4MultiplyVector4(invMVP, win[1]);
```

Remember the values could be in the homogenous coordinates system, to convert them back to human-imaginable cartesian coordinate system we need to divide by the w component.

``` objc
  /* covert rays from homogenous coordsys to cartesian coordsys */
  ray[0] = GLKVector4DivideScalar(ray[0], ray[0].w);
  ray[1] = GLKVector4DivideScalar(ray[1], ray[1].w);
```

We don’t need the start and end of the ray, but we need the ray in form of

```
 R = o + dt
```

Where o is the ray origin and d is the ray direction and t is the variable.

We calculate the ray direction as
``` objc
  /* direction of the ray */
  GLKVector4 rayDir = GLKVector4Normalize(GLKVector4Subtract(ray[1], ray[0]));
```

Why is it normalized? We will take look at that later.

Now we have the ray in the object space. We can simply to a sphere intersection test. We must already know the radius of the bounding sphere, if not we can easily calculate it. Like for a cube, I’m assuming the radius to be equal to the half edge of any side.

For detailed information regarding a ray-sphere intersection test I recommend this article, but I’ll go through the minimum that we need to accomplish our goal.

Let points on surface of sphere of radius r is given by

```
x^2 + y^2 + z^2 = r^2
P^2 - r^2 = 0; where P = {x, y, z}
```
All points on sphere where ray hits should obey

```
(o + dt)^2 - r^2 = 0
o^2 + (dt)^2 + 2odt - r^2 = 0
f(t) = (d^2)t^2 + (2od)t + (o^2 - r^2) = 0
```

This is a quadratic equation in form

```
 ax^2 + bx + c = 0
```

And, it makes sense, because every ray can hit the sphere a maximum two points, first when it goes in the sphere and second when it leaves the sphere.

The determinant of the quadratic equation is calculated as

```
det = b^2 - 4ac
```

If determinant is < 0 it means no roots or the ray misses the sphere
If determinant is equal 0 means one root or the ray started from the inside of the sphere or it just touches it.
If determinant is > 0 means we have two roots, or the perfect case of ray passing through the sphere.

In our equation the values of a, b, c are

```
a = d^2 = 1; as d is a normalized vector and dot(d, d) = 1
b = 2od
c = o^2 - r^2
```

So, if we have ray with direction normalized we can simply test if it hits our sphere in model space with

``` objc
- (BOOL)hitTestSphere:(float)radius
        withRayOrigin:(GLKVector3)rayOrigin
         rayDirection:(GLKVector3)rayDir
{
  float b = GLKVector3DotProduct(rayOrigin, rayDir) * 2.0f;
  float c = GLKVector3DotProduct(rayOrigin, rayOrigin) - radius*radius;
  printf("b^2-4ac = %f x %f = %f\n",b*b, 4.0f*c, b*b - 4*c);
  return b*b >= 4.0f*c;
}
```

Hope this helps you in understanding the object picking concept. Don’t forget to checkout the full code from the repository linked at the top of this article.
