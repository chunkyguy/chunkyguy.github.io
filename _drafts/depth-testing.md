A better way to render objects is using depth testing. The idea is that we use another texture to hold the depth value for every pixel and later the GPU decides which pixel is needs to be displayed.


Here's a diagram I found on [Apple website](https://developer.apple.com/documentation/metal/calculating_primitive_visibility_using_depth_testing?language=objc) to illustrate this
![A flowchart showing the depth test sequence. The depth of each new fragment is tested against data in the depth texture. When the test succeeds, the new fragment is stored to the depth and color textures.]( {{ site.url }} /assets/depth-testing-flow.png "Depth Test Flow")


# References

1. https://developer.apple.com/documentation/metal/calculating_primitive_visibility_using_depth_testing?language=objc