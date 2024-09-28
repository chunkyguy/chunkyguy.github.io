---
layout: post
title:  "The Art of Designing Softwares"
date:   2014-09-07 23:28:54 +0530
categories: rant
---

When I was in college, my vision of a software was like, we have a problem and we need a solution, a program, a set of instructions, that has to be executed by the computer in order to solve that particular problem. For example, design a program to print a Fibonacci series using loops, recursion, implement with an array, a doubly linked list, an AVL tree.

Then there are these programming challenges like the Facebook Hacker Cup, or the Google CodeJam, that just focus on how strong are you at understanding and implementing common algorithms and data structures, or can you design your own data structures and algorithms?

And, there is nothing wrong with that, but just that writing softwares isn’t just about designing algorithms or data structures, as one eventually discovers.

When I got my first job as an iOS developer, and start writing softwares at the enterprise levels. First thing I realized was that, the solution space is no longer just a set of instructions in a single file, but it is split into hundreds if not thousands of files. I realized that writing softwares is no longer just about algorithms and data structures. In fact, almost all of the interesting algorithms and data structures are either already solved and ready to use for your existing code, or nobody really cares about them as the first priority. There is a high probability that the programming languages of your choice is already bundled with all common data structures and algorithms you’ll ever need.

As a software engineer, most of the time I was working on managing the flow of data and control. The actual work seemed more close to plumbing, where instead of fluids one manages the flow of electrons through the hardware. Initially I was somewhat shocked to see that the iOS framework provides just three data structures, NSArray, NSDictionary and NSSet*. Thats it!

* (maybe also NSMapTable, NSHashTable, NSCountedSet, NSPointerArray,… but let’s assume we haven’t heard of them)

## What is software engineering?

For me software engineering is more about understanding the problem at a higher level. It is half part how good you’re at designing the flow yourself and half part, how good are you at reusing the existing solutions written by others.

Part 1: The design space
The design space deals with problems like:

How do I want the UI to be?
How do I manage downloading of multiple files in background even after my application has been quit?
How do I keep the memory used by the application in control? What happens, when I actually run out of memory?
How do I confirm that the code is just wrote, is going to pass all edge cases at all times?
Part 2: The integration space
The integration space deals with problems like:

How do I use the code I had written some time back to solve a similar problem?
How do I design this code, so that it could be used next time I face a similar problem?
Has someone already tackled a problem similar to mine? If yes, how can I use their solution?
Of these two subspaces, the design space seems easier, because you’re taught this at college level and you’re in total control of the design. The integration space is hard, and by hard I mean really hard. At a first few attempts, you might seriously consider skipping the integration and try moving it to the comfortable design space, i.e., reinvent the wheel.

But, the reality is that you can’t always design your own solutions. Maybe you’re working with an organization that already has codebase of solutions that has to be used; Or your other teammate is solving the problem in parallel; Or maybe the solution is already written out years ago and has been used and maintained for a really long time. Not to mention the time constraints with the design space.

With my years of development, I’ve come to realization that almost all the problems are already solved or there are people already working on it. If you think, you are the only one working on it, you’re most definitely wrong. (If not, please add me to your project).

I bigger challenge then is, how to use the solution written by others, or by you some time ago? The main problem is that the problem could have been solved in some other context, or maybe it was not thought out enough. There could also be some flaws in the basic design, like the existing solution is not thread safe.

At the even higher level, a software can be written in two ways. Bottom-up or Top-down.

## Bottom-up approach

Think of a problem where you have to design a car, in software that is. One of the ways you can design a car is, break down it into smaller tasks.

Design the tire.
Design the body.
Design the engine.
Here each step is recursive in itself, as the designing the engine would be again broken down further until you reach an atomic task that can not be broken down any further. You start working on those pieces and at then assemble them all back into places, and you have a car.

This solution has many benefits like, you can easily assign tasks between team members, and can almost work independently on them. Also, it is easier to design for the future this way, as when you’re working on designing the tires, you’re just focused on designing the tires that should fit all known use cases.

The main problem with bottom-up approach is, you need to be a good future-seeker. Whatever solutions you’re designing, should fall into place perfectly. Say on the final assembly day you realize the sockets on the body are not big enough to hold the tires. Also, as one can tell, bottom-up approach is more time consuming, because you’re designing for the unknown future. Imagine, your car has to perform well on all sort of geographical terrains. Most of the time you might even end up thinking too forward and design for the year 2056, while your solution would become obsolete in next 3 months. Try talking to all those who wrote great solutions with NSThread before Apple rolled GCD. And don’t forget the time constraints, your boss, teammates, consumers are always pressurizing you to release it already.

## Top down approach

Think of the same problem of designing the car, but this time you’re working top down. You don’t actually care about the year 2056, but just this day. You start with a giant piece of metal and a empty notepad.

Then you start carving the metal block into the car. When you get to carving out the left tire, you might realize you’ve already done something similar. Then, you realize, yes sometime back you designed the right tire and this feels almost similar. So, you just apply the same procedure again and as your car comes to finish. With top-down approach often you might feel like the amount of work getting reduced with each iteration.

The problems with top-down approach are that, it’s really hard to work with a big team, because you’re blank at the beginning, so you can not assign tasks. Most of the time you need to start alone, and bring in more teammates as you progress.

The main differences between the two approaches are:

Top-down feels more like looking into the past while bottom-up is more into looking into the future.
Top-down solves the same problem in lesser time than bottom-up.
Top-down is not as future safe as bottom-up. Imagine a request to add a radio in the car. With bottom-up an engineer might have already thought about something similar and kept a ‘future’ slot, but with top-down you need to carve a hole and move internal stuff until you just get the radio working.
At the end of the day, top-down seems to taking you more closer to the end product, while bottom-up still seems uncertain until the final assembly.
What approach you pick actually depends on the task you’re solving. If you designing a framework or a library that has to be used by others or by you in the future, and have a lot of time at your hands, go for the bottom-up.

If you’re designing the end product, where time is limited. Maybe your product is eventually going to end up in garbage in 2 years, go with top down.

Another approach that can be tried is the hybrid approach, where you break the original task into smaller chunks using the bottom-up approach. Assign each sub problem to each teammate and then work on each sub problem with top-down approach in parallel until the final assembly.

