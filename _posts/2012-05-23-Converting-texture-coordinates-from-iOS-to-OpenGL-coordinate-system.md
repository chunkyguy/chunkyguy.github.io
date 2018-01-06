---
layout: post
title:  "Converting texture coordinates from iOS to OpenGL coordinate system"
date:   2012-05-23 23:28:54 +0530
categories: openGL
---

Using Sprite Sheet: Converting texture coordinates from iOS to OpenGL coordinate system
========================================================================================


One thing I realized at a very early stage is the importance of using a Texture Atlas or Sprite Sheet compared to individual images. If you haven’t yet, I recommend doing it now and realizing the benefits later.

For sanity sake let me tell you what a texture atlas or sprite sheet is, I assume they are the same thing, if not please let me know.

A sprite sheet or texture atlas is a smart way putting all your images in one giant image usually accompanied by a smart file. The giant image is the texture and the smart file is the atlas, this is how I see it.

For example if I’ve these three images from my under development game :



Then my sprite sheet might look like this:



“Big change”, you might say sarcastically, but it’s a big change, assume that as you game grows and you add more and more image and still you’ll just have one giant image to load into memory, that will reduce a lot of memory overhead, compared to the situation where you’re loading images one by one and releasing one by one.

And, if you would be smart enough you would design your sprite sheets in such a manner that one giant image has all the images required for the current scene on the screen. Say you keep one texture atlas for the settings screen, another for the main game and another for the main menu and load them when you switch the scenes.

OK, that’s enough theory about sprite sheets, google it if you want to know more. Let’s now dig into the important part creating and using sprite sheets.

When creating a sprite sheet, you’ve two options: the hard way or the smart way.

Hard way is you invest some hours writing some code, and create your own algorithm for creating sprite sheet, and believe me it’s not that tough. But, the smart way is more smarter, use any of the awesome apps out there, like Texture Packer, or Zwoptex.

For my pleasure I use Zwoptex, because I liked the Sci-fi name, you can pick anything.

Now, there are many file formats available like cocos2d(plist), sparrow(xml), corona(lua), again you’re free to pick any one, I’ve picked the Zwoptex Generic(plist), for one reason that it’s a plist and the other that I love the name :) .

And, again trust me when I say that most of code I’m going to explain will work for cocos2d format too.

Now lets dive into the using sprite sheet part. Remember when I said about the smart file that accompanies the giant image, that smart file is the plist file that we’re going to parse for all the information.

So, for the above example the plist file would look something like:

```
<!--?xml version="1.0" encoding="UTF-8"?-->

	frames

		mrseal.png

			aliases

			spriteColorRect
			{ { 64, 1}, {64, 148 } }
			spriteOffset
			{-4, -0}
			spriteSize
			{64, 148}
			spriteSourceSize
			{200, 150}
			spriteTrimmed

			textureRect
			{ { 0, 362}, { 64, 148 } }
			textureRotated

		snowman.png

			aliases

			spriteColorRect
			{ { 5, 0 }, { 120, 113 } }
			spriteOffset
			{-10, -0}
			spriteSize
			{120, 113}
			spriteSourceSize
			{150, 113}
			spriteTrimmed

			textureRect
			{ { 66, 362 }, { 120, 113 } }
			textureRotated

		snowview.png

			aliases

			spriteColorRect
			{ { 0, 0 }, { 480, 360 } }
			spriteOffset
			{0, -0}
			spriteSize
			{480, 360}
			spriteSourceSize
			{480, 360}
			spriteTrimmed

			textureRect
			{ { 0, 0 }, { 480, 360 } }
			textureRotated

	metadata

		version
		1.5.2
		format
		3
		size
		{512, 512}
		name
		Untitled
		premultipliedAlpha

		target

			name
			default
			textureFileName
			demo
			textureFileExtension
			.png
			coordinatesFileName
			demo
			coordinatesFileExtension
			.plist
			premultipliedAlpha
```

As, you can see that the structure basically consists of  a dictionary of two things, a frames and a metadata. The frames is the one we’re more interested in, it is again an array of dictionaries, with each dictionary having a key as the filename and the other data for the relative coordinated in the giant image.

Here’s my code for parsing that data:

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
-(GLKVector4)texCoordsForImageAtIndex:(NSInteger)indx{
    if(indx >= [imageNames_ count])
        return GLKVector4Make(0.0, 0.0, 0.0, 0.0);
 
    NSDictionary *frameDict = [imageValues_ objectAtIndex:indx];
    NSString *frameStr = [frameDict objectForKey:@"textureRect"];
    CGRect frame = CGRectFromString(frameStr);
    float u = frame.origin.x;
    float v = frame.origin.y;
    v += frame.size.height;
    v = fullSize_.height - v;
    float s = u + frame.size.width;
    float t = v + frame.size.height;
    return GLKVector4Make(u/fullSize_.width, v/fullSize_.height, s/fullSize_.width, t/fullSize_.height);
}
Now, let me explain the code, line by line.

This method takes a index and returns the openGL texture coordinates, there’s also an helper method that can achieve same thing with the image name, and most of the time I use this:

1
2
3
-(GLKVector4)texCoordsForImageNamed:(NSString *)name{
    return [self texCoordsForImageAtIndex:[imageNames_ indexOfObject:name]];
}
And I assume it’s just simple enough to understand.

Line 2-3: Just a normal exception check and return a GLKVector4 with everything set as 0.

Line 5: The frameDict is the dictionary corresponding to the image name, imageValues_ is just an array corresponding to the frames in the plist.

Line 6: Lets assume we want to render the snowview.png on the screen, so we get the textureRect from the dictionary, which the rectangle in the absolute space. Now, we just have to transform that to the openGL texture coordinate system.

Line 7: Convert the string into CGRect type.

Line 8-9: Get the origin point, and set it as u and v.

Line 10-11: We’ve already calculated the fullSize_ of the texture, the giant image as

1
2
3
4
5
    NSDictionary *metaDict = [texDict objectForKey:@"metadata"];
NSString *sizeStr = [metaDict objectForKey:@"size"];
NSString *fullFrameStr = [NSString stringWithFormat:@" { { 0,0 }, %@ }",sizeStr];
CGRect fullFrame = CGRectFromString(fullFrameStr);
fullSize_ = fullFrame.size;
Next, we just transform the origin y in openGL coordinate space, as the zwoptex plist and even the cocos2d plist format assumes the origin at top-left, while the openGL coordinate system assumes origin at the center of screen.

So, if originally

v = 0,

frame.size.height = 360

and the fullSize_.height = 1024,

then after the calculations,

v = 664

Line 12-13: We adjust the size of the frame in the openGL coordinate system, by adding the respective size to the origins.

So, for if a frame in iOS coordinate system is:

[0, 0, 480, 360]

We’ve the corresponding openGL coordinate system as:

[0, 664, 480, 1024]

To illustrate pictorially,  I created this image with Grapher:



Line 14: This is the final step, just to set the values in range 0 to 1, we divide the values by the size of the giant image, that should make it something like:

[0, 0.62, 0.46, 1]

Hope, this helps you in understanding the math and usage of texture mapping, for the entire time I was assuming the Zwoptex plist format, but the cocos2d plist format is almost similar, this code should be comfortable with that too with just a little tweak.