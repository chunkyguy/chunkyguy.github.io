---
layout: post
title:  "Getting started Cocos2d with Swift"
date:   2014-06-24 23:28:54 +0530
categories: concurrency
published: false
---

I understand that everybody is super excited about the Swift Programming Language right? Although Apple has provided the Playground for getting acquainted with Swift but there is an even better way – by making a game.

If you’re already familiar with Cocos2d and want to make games using Cocos2d and Swift, then walk through there simple steps to get started with the Cocos2d project template you already have in your Xcode.

1. Create a new Cocos2d project: This is nothing new for you guys.



2. Give it a name: Why am I even bothering with these steps?



3. Add Frameworks: I’m not sure if this is a temporary thing, but for some reasons I was getting all sort of linking errors for missing frameworks and libraries. These are all the iOS Frameworks the Cocos2d uses internally. Click on the photo to enlarge and see what all frameworks you actually need to add.



Make sure you tidy them up in a group, because the Xcode beta will add them at top level of your nice structure. Again, I guess this is also a temporary thing as Xcode 5 already knows how to put them inside a nice directory.

4. Remove all your .h/m files: Next remove all the .m files that Cocos2d creates for you. Ideally they should be AppDelegate.h/m HelloWorldScene.h/m IntroScene.h/m and main.m.

Yes, you heard it right. We don’t need main.m anymore as Swift has no main function as an entry point. Don’t remove any Cocos2d source code. Best guess is, if it’s inside the Classes directory it’s probably your code.



5. Download the repository: https://github.com/chunkyguy/Cocos2dSwift

6. Add Swift code: Look for Main.swift, HelloWorldScene.swift and IntroScene.swift in the Classes directory and add them to your project by the usual drag and drop. While you’re adding them, you should get a dialog box from Xcode ‘Would you like to configure an Objective-C bridging header?’.



Say ‘Yes’. The Xcode is smart enough to guess that you’re adding Swift files to a project that already has Objective-C and C files.

The bridging header makes all your Objective-C and C code visible inside your Swift code. Well, in this case it’s actually the Cocos2d source code.

7. Locate the bridging header: It should have a name such as ‘YourAwesomeGame-Bridging-Header.h’. In it add the two lines we need to bring all the cocos2d code we need inside our Swift code.



In case you have some other custom Objective-C code you would like your Swift code to see, import it here.

8. That it folks! Compile and run.



There are some trivial code changes that you can read in repository’s README file.

Have fun!
