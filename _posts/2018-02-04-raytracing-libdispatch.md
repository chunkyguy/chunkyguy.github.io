---
layout: post
title:  "Raytracing with libdispatch"
date:   2018-02-04 14:35:00 +0200
categories: concurrency raytracing
published: true
---

### Problem

So, I decided to make the ray tracer render frames faster. It was getting significantly painful to make some change, build and run and wait for several minutes before the rendering completed.

Before any performance improvements, I profiled the time, it was around 10 minutes per frame.

![main thread](https://i.imgur.com/gYlhjjx.png)

This was when using a single thread for all the processing.

### Solution

The good thing about a ray tracer is that, it is actually very straightforward to make every ray being traced as purely reentrant which makes them ideal candidate for concurrency. The only thing that needs special consideration is the data structure holding the pixel information. As each ray can take different time to compute the final color value, we need a way to synchronize the ray coordinates with the final color value, to avoid any race conditions between 2 concurrent write operations.

To illustrate the problem, a ray which hits no objects will be much faster to compute the final value than the ray which hit multiple objects and bounces all over the place. The first solution to such a solution could be to guard the write operation to the image data with a [mutex](https://en.wikipedia.org/wiki/Lock_(computer_science)).

``` cpp
// data structure for pixel data
Film film;

for (int j = ny - 1; j >= 0; --j) {
    for (int i = 0; i < nx; ++i) {
        // spawn a new thread
        simd::float3 color = getColor(nx, ny, ns, i, j, camera, space);

        // write color value by acquiring a mutex lock
        film.updateColor(color, i, j);
        // release mutex
    }
}
```

The reason this implementation is just a pseudo-code is that this will never scale up, as this algorithm requires number of threads equal to number of pixels of the target image. Which as per [Amdahl's law](https://en.wikipedia.org/wiki/Amdahl%27s_law) might do more harm than good.

### libdispatch

The first solution that I actually tried was using [Apple's libdispatch](https://apple.github.io/swift-corelibs-libdispatch/). Grand Centeral Dispatch (GCD) or libdispatch is a high level concurrency library which moves the concurrency problem from threads to tasks. At the core of libdispatch lies the concept of queues. A queue can then run `N` number of tasks in parallel. With libdispatch, each operation can be thought of as a unit of work that can then be dispatched to any queue. The only challenge left then is to design the queue architecture and work units or operations to be performed.

For our case, if we think of each rendering of a pixel as an independent task then we can then simply schedule the tasks on a parallel queue. At the end of each trace, we need to update the shared data structure with color information. And once all the tasks are complete we are done!

The first thing we need is a serial queue `filmQueue` which is responsible for synchronizing write operations on `film`, which is just the pixel data store.

``` cpp
dispatch_queue_t filmQueue = dispatch_queue_create("com.whackylabs.srt", DISPATCH_QUEUE_SERIAL);
```

Next we can either make our own concurrent queue, or simply use one of the few system global concurrent. Let us call this parallel queue `rayQueue` which is the concurrent queue and manages the parallel execution of all the rays in the scene. With this task based approach we won't have to manage the thread pool ourselves.

``` cpp
dispatch_queue_t rayQueue = dispatch_get_global_queue(QOS_CLASS_BACKGROUND, 0);
```

Now we can already implement our algorithm:

```cpp
for (int j = ny - 1; j >= 0; --j) {
    for (int i = 0; i < nx; ++i) {
        dispatch_async(rayQueue, ^{
            float3 color = getColor(nx, ny, ns, i, j, camera, space);
            dispatch_async(filmQueue, ^{
                film.updateColor(color, i, j);
            });
        });
    }
}
```
With that in place, we now need a mechanism to notify us when the all the rays have evaluated a color, so that we can start processing the collected pixel data.

This is the job best suited for dispatch groups. You can think of dispatch group as a high level semaphore which waits for N resources and signals when all resources are released. To acquire the resource we have to call `dispatch_group_enter` and to release `dispatch_group_leave`. The signal is implemented with `dispatch_group_notify`.

Here's how this would look:

``` cpp
dispatch_group_t rayTask = dispatch_group_create();

for (int j = ny - 1; j >= 0; --j) {
    for (int i = 0; i < nx; ++i) {
        dispatch_group_enter(rayTask);
        dispatch_async(rayQueue, ^{
            float3 color = getColor(nx, ny, ns, i, j, camera, space);
            dispatch_async(filmQueue, ^{
                film.updateColor(color, i, j);
                dispatch_group_leave(rayTask);
            });
        });
    }
}

dispatch_group_notify(rayTask, filmQueue, ^{
    film.process();
});
```

If you run this code as is, the program would exit even before the first ray has returned. The reason being that our main program runs on the main thread, and there is nothing holding our main thread from finishing. We could either use the main queue which is a serial queue around the main thread as our `filmQueue`. But I don't like the idea, if we decide to have some better UI around our ray tracer, like a loading indicator or a progress bar, we would need the main thread for that.

For our current needs what we simply need is this one line at the end:

``` cpp
dispatch_main();
```

And that should keep the main thread busy forever. Which means our program never exits. That is also not very good. The fastest way around is then to explicitly exit the program when the image has been processed.

``` cpp
film.process();
dispatch_async(mainQueue, ^{
    exit(0);
});
```
With this change, I was able to reduce the run time from 10 minutes to 7 minutes 30 seconds. Not bad!

![concurrent background](https://i.imgur.com/dl65mc4.png)

But this is not all, remember when we created the `rayQueue` we passed in the `QOS_CLASS_BACKGROUND`. This flag dictates the quality of service class, which gives the system a hint on what kind of operations is the queue going to perform. Here I picked background, which isn't entirely true. By simply changing the quality of service class to `QOS_CLASS_USER_INITIATED`, the runtime goes down to around 4 minutes! Awesome!

![concurrent user initiated](https://i.imgur.com/EsqOAwG.png)


Finally since we are mutating the `film` within `filmQueue`, we need to capture the reference. The way it's done blocks is with the use of `__block` keyword.

``` cpp
__block Film film(nx, ny);
```
### Conclusion

Although the [free lunch is definitely over](http://www.gotw.ca/publications/concurrency-ddj.htm), but still it is amazing to realize how much performance gain can be achieved by simply using easy to use concurrency libraries such as libdispatch.

Maybe I'll give the C++ standard thread library a shot. I would like to benchmark the gain of using `std::async` someday, but I doubt if they would be as much given the fact that concurrency is much more about what lies in the implementation, which I don't think Apple cares for C++ standard as much as for their libdispatch.

The entire code is available at [github.com/chunkyguy/SimpleRayTracer](https://github.com/chunkyguy/SimpleRayTracer)

