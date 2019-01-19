---
layout: post
title:  "Embedding Freetype for iOS projects"
date:   2013-09-13 23:28:54 +0530
categories: gamedev ios
---

Freetype is a font library written in C. From OpenGL’s point of view, it creates alpha texture on the fly for any font size for any string.

**Step 1**: Embedding the original freetype source to the code is some real pain in the ass. Fortunate for us, one guy has done that task and released the code on github. https://github.com/cdave1/freetype2-ios. So, step 1 is to check out that code.

**Step 2**: Create your new project and add the project downloaded from step 1 to your project.

Check that the Xcode must have created two schemes for you.

**Step 3**: Go to the your Project Settings, select your target and click on ‘Build Phases’. Under ‘Target Dependencies’ add the static library from the freetype2 project.



This is going to build the static library for you, without you having to explicitly build it.

**Step 4**: Go to ‘Link Binary with Libraries’ and add the static library.



This will build the libFreetype2.a and put it inside the products directory.

**Step 5**: Next, we need to add the freetype header files in the products directory. Why? Because, the Xcode projects are configured to look for header files inside the products directory automatically, without us having to specify the search path. Also, if in future we make some changes to the freetype project header files (very unlikely) the changes will be automatically updated in our products directory.

Now, open the freetype2 Project settings and go to ‘Build Phases’. Click ‘Add Build Phase’ and ‘Add Copy Files’



**Step 6**: Change the ‘Destination’ to ‘Products Directory’. Now, this is the part where Xcode sucks, we need to copy our header files from the freetype2 project, but doing so flattens the copied directory structure.

One way to fix this is to remove the include files from the freetype2 project and add them again as ‘Folder reference’



**Step 7**: Drag the include folder from freetype2 project to the ‘Copy Files’ phase.



If you’ve followed everything correct to this point, you project should just build fine. It should compile the freetype target, copy the header files and then compile and link your project.



The products directory should look something like



For final testing, try adding the following lines of code to your main.m file and compile the code:
``` objc
#import <UIKit/UIKit.h>
#import "AppDelegate.h"
#import <ft2build.h>
#import <freetype/freetype.h>
#import <assert.h>
 
void test_freetype() {
    FT_Library library;
    FT_Error error = FT_Init_FreeType(&library);
    assert(!error);
    FT_Done_FreeType(library);
}
 
int main(int argc, char *argv[])
{
    test_freetype();
 
    @autoreleasepool {
        return UIApplicationMain(argc, argv, nil, NSStringFromClass([AppDelegate class]));
    }
}
```

