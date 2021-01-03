---
layout: post
title:  "Welcome to Jekyll!"
date:   2019-11-02 23:28:54 +0200
categories: jekyll update
published: false
---

# Understanding UIScrollView zooming

I'll attempt to expand the Ole Begemann's [CustomScrollView](https://oleb.net/blog/2014/04/understanding-uiscrollview/) by adding zooming to it. If you have not already read that article then please go and read that first because this would an extension to the project.

# A little math

Most of the work that we'd be doing is converting points between coordinate systems. That can either be done by figuring out the equations to transform points from one coordinate system to another, or we can use the established rules of linear algebra. To understand, let's the the example of translation.

If we have a `CGRect` and we want to move if by some offset, a rather simple way is using `CGRectOffset()`

```objc
CGRect frame = CGRectMake(10, 10, 100, 100);
CGPoint offset = CGPointMake(50, 50);
CGRect frameOut = CGRectOffset(frame, offset.x, offset.y); // {{60, 60}, {100, 100}}
```

Another way is to think of this a transforming points from one coordinate system to another where the *target coordinate system* has:

 ```
 x' = x + 50
 y' = y + 50
 ```
 
 Then, we can create a matrix that does this transformation.

```objc
GLKMatrix4 transform = GLKMatrix4MakeTranslation(offset.x, offset.y, 0);
```

The reason we're using a 4D matrix is because most of the popularly available linear algebra libraries provide 4D type system, also known as the [homogeneous coordinate system](https://en.wikipedia.org/wiki/Homogeneous_coordinates#Use_in_computer_graphics_and_computer_vision), which helps with transforming 3D vectors. For our 2D purposes we can always assume that `z` is always `0`.

Our above matrix is internally a 4x4 grid of `CGFloat` values:

```objc
{
    1, 0, 0, 50,
    0, 1, 0, 50, 
    0, 0, 1, 0,
    0, 0, 0, 1
}
```

Now with this matrix if we multiply any 4D vector, like `{10, 10, 0, 1}`, we would get `{60, 60, 0, 1}`. Notice the `w = 1` is important, if we use `w = 0`, we would get `{10, 10, 0, 1}`. *Tip: Use [this fancy app](http://matrixmultiplication.xyz) to see the matrix math in action* 

[!img]

With some basic understanding of linear algebra we can use that to perform the same calculation as above

```objc
GLKVector4 framesIn[2] = {
    GLKVector4Make(CGRectGetMinX(frame), CGRectGetMinY(frame), 0, 1), // {10, 10, 0, 1}
    GLKVector4Make(CGRectGetMaxX(frame), CGRectGetMaxY(frame), 0, 1) // {110, 110, 0, 1}
};
GLKVector4 framesOut[2] = {};
for (int i = 0; i < 2; ++i) {
    framesOut[i] = GLKMatrix4MultiplyVector4(transform, framesIn[i]);
}
CGRect frameOut = CGRectMake(
                        framesOut[0].x, 
                        framesOut[0].y,
                        framesOut[1].x - framesOut[0].x,
                        framesOut[1].y - framesOut[0].y); // {{60, 60}, {100, 100}}

```

This must feel like a lot of work compared to `CGRectOffset()`, but this gets easier when we have to multiple transformations. For example if we want to scale `{1.5, 1.5}` and then translate `{50, 50}` and then rotate by `75º`.

# UIScrollView Zoom

Coming back to adding `zoomScale` to `CustomScrollView`. We can think of zoom operation as transforming points to a *target coordinate system* where 

```
 x' = x * scaleX
 y' = y * scaleY
 ```

 ```objc
 GLKMatrix4 scaleTransform = GLKMatrix4MakeScale(scaleX, scaleY, 1);
 ```

If we simply transform 4 corners of our `frame` we might end up scaled up frame, but we want the final `frame` to still be the original size. For instance, if our original `frame` was `{{50, 100}, {50, 50}}` and we want to scale is up by 2x the resulting `frame` would end up being `{{100, 200}, {100, 100}}`.

[!img]

But we want the size to still be `{50, 50}`. So, for this case the final frame should be `{{180, 260}, {320, 480}}` to keep the centers aligned.

One way to do this is by 

```objc
CGRect wl_scrollZoom(CGRect frame, CGSize contentSize, CGFloat scale)
{
  // get center of the frame
  CGPoint frameCenter = CGPointMake(CGRectGetMidX(frame), CGRectGetMidY(frame));

  // create transformation matrix
  GLKMatrix4 transform = GLKMatrix4MakeScale(scale, scale, 1.0f);

  // convert center to new coordinate system
  GLKVector4 vecIn = GLKVector4Make(frameCenter.x, frameCenter.y, 0, 1);
  GLKVector4 vecOut = GLKMatrix4MultiplyVector4(transform, vecIn);

  // build frame from center and size
  CGFloat originX = vecOut.x - frame.size.width/2.0;
  CGFloat originY = vecOut.y - frame.size.height/2.0;
  return CGRectMake(originX, originY, frame.size.width, frame.size.height);
}
```

Next, we need to add constraints to avoid frame from going beyond `contentSize`
 
# Further Reading

Source code is available in my branch [github.com/chunkyguy/CustomScrollView/tree/zoom](https://github.com/chunkyguy/CustomScrollView/tree/zoom)