---
layout: post
title:  "Writing first ray tracer"
date:   2017-12-30 13:02:00 +0200
categories: pbr
---

One of the many goals I had for 2017 were:

1. Writing a ray tracer.
2. Restarting the blog.

# Writing a ray tracer

Sometime when the trees were starting to shed their leaves for the winters, I was worried that I was falling  behind my goals for the year. I finally decided to order the book I wanted to read for many years now. The following day I placed an order for [Physically Based Rendering](http://www.pbrt.org), which arrived within a long and sleepless month.

As I was close to finishing the first chapter, I realized I had never written a ray tracer. Although, many people tend to claim that writing a ray tracer is the most natural and natural first step in the beautiful world of computer graphics, or like how easy it is to write a ray tracing algorithm, as it requires no prior knowledge of any graphics API or how the graphics pipeline functions, for that matter. Still, I had never written it. 

The closest I had ever been to understanding a ray tracer was when following the [dissection](http://fabiensanglard.net/rayTracing_back_of_business_card/) of the mysterious and amusing [ray tracer on a business card](http://eastfarthing.com/blog/2016-01-12-card/). As a follow up, I started following the articles on [https://www.scratchapixel.com](https://www.scratchapixel.com) about ray tracing, to get a more in depth understanding of how would one go in order to write a ray tracer from scratch.

Although, these blogs were tempting enough for me to start diving into writing my first ray tracer, but still I was too busy (read lazy) to actually do it. Then, one day I ran across the minibook [Ray tracing in one weekend](https://www.amazon.com/Ray-Tracing-Weekend-Minibooks-Book-ebook/dp/B01B5AODD8). Although the book was probably designed to be finished within a weekend, and rightfully so, still I took it really slow and maybe finished in about a month. 

For every chapter, I used to do my own side research as well. For example, when working on the camera, I would often go side track and read about the innards of a real life camera. How do things like appertures, depth of field are supposed to work? What is the role of a lens in camera, and so on. Also, while coding, I tried to write a bit modern-ish C++ and explore the Apple `simd.h` library for all my math needs, which interestingly has no or very less online presence. For example, there is no proper documentation except for the header files, and searching any thing with keywords _simd_ would return results from anything related to the SSE instruction sets to the architecture itself.

Eventually, today, just when the year is about to end, I was able to render the final target image from the book!

![ray traced spheres](https://i.imgur.com/s5W3SrK.png)

[https://github.com/chunkyguy/SimpleRayTracer](https://github.com/chunkyguy/SimpleRayTracer)


# Restarting the blog

Another thing I had planned to do for a very long time was restarting this blog. The death of this blog started with some miscommunication between me and my hosting service [Yahoo Small business India](https://india.yahoosmallbusiness.com). Unfortunately my service renew date coincided with their transition to Aabaco. So every time I tried to renew my service I received a generic and unhelpful try again later error. At the same time, I was also moving continents, literally speaking. Later when I was finally settled and tried to renew the service again to my horror I received the message from them that due my failure to renew they had wiped my data from the server. 

Eventually, I transferred the domain name and built the blog up from whatever scrap I could find. This entire process had left me a bit wretched to say the least. Also, at the same time I was having hard time finding something interesting to focus on that I could write about.

Finally, it was the goal of 2017 to restart the blog again, and if this post is published before the dawn of 2018 I believe 2017 was a great year after all for me!

I hope you had a great year too! Let's carry the positive spirit to 2018!
