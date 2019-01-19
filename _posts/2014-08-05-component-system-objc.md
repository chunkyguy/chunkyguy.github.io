---
layout: post
title:  "Component System using Objective-C Message Forwarding"
date:   2014-08-05 23:28:54 +0530
categories: concurrency
---

Ever since I started playing around with the Unity3D, I wanted to build one of my games with component system. And, since I use Objective-C most of the time, so why not use it.

First let me elaborate on my intention. I wanted a component system where I’ve a GameObject that can have one or more components plugged in. To illustrate the main problems with the traditional Actor based model lets assume we’re developing a platformer game. We have these three Actor types:

1. Ninja: A ninja is an actor that a user can control with external inputs.
2. Background: Background is something like a static image.
3. HiddenBonus: Hidden bonus is a invisible actor that a ninja can hit.

Source: Swim Ninja
Source: Swim Ninja
Let’s take a look at all the components inside these actors.
```
Ninja = RenderComponent + PhysicsComponent
Background = RenderComponent
HiddenBonus = PhysicsComponent.
```
Here RenderComponent is responsible for just drawing some content on the screen, while the PhysicsComponent is responsible for just doing physics updates like, collision detection.

So, in a nutshell from our game loop we need to call things like
``` objc
- (void)loopUpdate:(int)dt
{
    [ninja update:dt];
    [bonus update:dt];
}

- (void)loopRender
{
    [background render];
    [ninja render];
}
```

Now, in the traditional Actor model, if we have an Actor class with both RenderComponent and PhysicsComponent like:
``` objc
@interface Actor : NSObject

- (void)render;
- (void)update:(int)dt;

@end
```

Then it would inevitably add a PhysicsComponent to Background actor and a RenderComponent to a HiddenBonus actor.

In Objective-C we can in fact design the component system using the message forwarding.
( Don’t forget to checkout this older post on message forwarding ).

At the core, we can have a GameObject class like
``` objc
@interface GameObject : NSObject

- (void)render;
- (void)update:(int)dt;

@end
```

But this doesn’t implements any of these methods, instead it forwards them to relevant Component classes. We can write our Component classes as
``` objc
@interface PhysicsComponent : NSObject

- (void)update:(int)dt;

@end

@interface RenderComponent : NSObject

- (void)render;

@end
```

Our GameObject class gets reduced to
``` objc 
@interface GameObject : NSObject

+ (GameObject *)emptyObject;

- (void)enablePhysicsComponent;
- (void)enableRenderComponent;

@end
```

Our game loop initializes our GameObjects as
``` objc
- (void)loadGameObjects
{
    ninja = [GameObject emptyObject];
    [ninja enablePhysicsComponent];
    [ninja enableRenderComponent];

    bonus = [GameObject emptyObject];
    [bonus enablePhysicsComponent];

    background = [GameObject emptyObject];
    [background enableRenderComponent];
}
```

And updates and render them as
``` objc
- (void)loopUpdate:(int)dt
{
    [ninja update:dt];
    [bonus update:dt];

}

- (void)loopRender
{
    [background render];
    [ninja render];
}
```

If you compile at this stage, you will get errors like:
```
‘No visible @interface for ‘GameObject’ declares the selector ‘update’.
‘No visible @interface for ‘GameObject’ declares the selector ‘render’.
```

In earlier days the code used to compile just like that. Later it became an warning, which was good for most of the developer who migrated to Objective-C from other compile time bound languages like C++. Objective-C is mainly a runtime bound language, but very few developers appreciate this fact and they complained a lot on why compilers don’t catch their errors early on, or more precisely, why it doesn’t works somewhat like C++. And I understand that feeling, because I too love C++. But, when I’m coding in Objective-C I use it like Objective-C. They are two completely different languages based on completely different ideology.

Moving forward in time, ARC came along. It made using Objective-C runtime behavior impossible. Now the ‘missing’ method declaration became an error. For this experimentation, let’s disable ARC entirely and see how it goes.

OK, good enough. The errors are now warnings. But we can at least move forward with our experiment.

Lets add the message forwarding machinery to our GameObject. Whenever a message is passed to an object in Objective-C, if the message is not implemented by the receiver object, then before throwing an exception the Objective-C runtime offers us an opportunity to forward the message to some other delegate object. The way this is done is quite simple, we just need to implement following methods.

First thing we need to handle is passing a selector availability test.
``` objc
- (BOOL)respondsToSelector:(SEL)aSelector
{
    if ( [super respondsToSelector:aSelector] ) {
        return YES;
    } else if(renderComponent && [renderComponent respondsToSelector:aSelector]) {
        return YES;
    }  else if(physicsComponent && [physicsComponent respondsToSelector:aSelector]) {
        return YES;
    }
    return NO;
}
```

Next, we need to handle the method selector signature availability test.
``` objc
- (NSMethodSignature*)methodSignatureForSelector:(SEL)selector
{
    NSMethodSignature* signature = [super methodSignatureForSelector:selector];
    if (signature) {
        return signature;
    }

    if (renderComponent) {
        signature = [renderComponent methodSignatureForSelector:selector];
        if (signature) {
            return signature;
        }
    }

    if (physicsComponent) {
        signature = [physicsComponent methodSignatureForSelector:selector];
        if (signature) {
            return signature;
        }
    }

    return nil;
}
```

After all the tests have been cleared, it time to implement the actual message forwarding.
``` objc
- (void)forwardInvocation:(NSInvocation *)anInvocation
{
    if ([renderComponent respondsToSelector: [anInvocation selector]]) {
        [anInvocation invokeWithTarget:renderComponent];
    } else if ([physicsComponent respondsToSelector: [anInvocation selector]]) {
        [anInvocation invokeWithTarget:physicsComponent];
    } else {
        [super forwardInvocation:anInvocation];
    }
}
```

If you run the code at this point, you should see update and render getting invoked for the corresponding component objects. So yay, our component system prototype is working!

Now let try to make it more concrete. Let’s work on the RenderComponent first. For testing in each render call, we just draw a Cube on the screen.

The best part about the Component System is that every component focuses on just one thing. For example, the RenderComponent just focuses on rendering. We just need to add methods that are required just for rendering.

Lets pass in the matrix information to the RenderComponent in form of a NSValue.
``` objc
@interface RenderComponent : NSObject

- (void)setNormalMatrix:(NSValue *)nMat;
- (void)setModelViewProjectionMatrix:(NSValue *)mvpMat;

- (void)render;

@end
@implementation RenderComponent

- (void)setNormalMatrix:(NSValue *)nMat;
{
    [nMat getValue:&_normalMatrix];
}

- (void)setModelViewProjectionMatrix:(NSValue *)mvpMat;
{
    [mvpMat getValue:&_modelViewProjectionMatrix];
}

- (void)render;
{
    glUniformMatrix4fv(uniforms[UNIFORM_MODELVIEWPROJECTION_MATRIX], 1, 0, _modelViewProjectionMatrix.m);
    glUniformMatrix3fv(uniforms[UNIFORM_NORMAL_MATRIX], 1, 0, _normalMatrix.m);

    glDrawArrays(GL_TRIANGLES, 0, 36);
}

@end
```

The conversion from GLKMatrix to NSValue can be done easily like:
``` objc
    GLKMatrix3 nMat = GLKMatrix3InvertAndTranspose(GLKMatrix4GetMatrix3(modelViewMatrix), NULL);
    GLKMatrix4 mvpMat = GLKMatrix4Multiply(projectionMatrix, modelViewMatrix);
    [ninja setNormalMatrix:[NSValue value:&nMat withObjCType:@encode(GLKMatrix3)]];
    [ninja setModelViewProjectionMatrix: [NSValue value:&mvpMat withObjCType:@encode(GLKMatrix4)]];
```

Now using our message forwarding magic, in our main loop we just need to update the GameObjects with rendering enabled like so:
``` objc
- (void)loopUpdate:(int)dt
{

    float aspect = fabsf(self.view.bounds.size.width / self.view.bounds.size.height);
    GLKMatrix4 projectionMatrix = GLKMatrix4MakePerspective(GLKMathDegreesToRadians(65.0f), aspect, 0.1f, 100.0f);

    GLKMatrix4 baseModelViewMatrix = GLKMatrix4MakeTranslation(0.0f, 0.0f, -4.0f);
    baseModelViewMatrix = GLKMatrix4Rotate(baseModelViewMatrix, _rotation, 0.0f, 1.0f, 0.0f);

    // Compute the model view matrix for the object rendered with GLKit
    GLKMatrix4 modelViewMatrix = GLKMatrix4MakeTranslation(0.0f, 0.0f, -1.5f);
    modelViewMatrix = GLKMatrix4Rotate(modelViewMatrix, _rotation, 1.0f, 1.0f, 1.0f);
    modelViewMatrix = GLKMatrix4Multiply(baseModelViewMatrix, modelViewMatrix);

    [ninja setNormalMatrix: NSStringFromGLKMatrix3(
     GLKMatrix3InvertAndTranspose(GLKMatrix4GetMatrix3(modelViewMatrix), NULL))];
    [ninja setModelViewProjectionMatrix: NSStringFromGLKMatrix4(GLKMatrix4Multiply(projectionMatrix, modelViewMatrix))];

    modelViewMatrix = GLKMatrix4MakeTranslation(0.0f, 0.0f, 1.5f);
    modelViewMatrix = GLKMatrix4Rotate(modelViewMatrix, _rotation, 1.0f, 1.0f, 1.0f);
    modelViewMatrix = GLKMatrix4Multiply(baseModelViewMatrix, modelViewMatrix);

    [background setNormalMatrix: NSStringFromGLKMatrix3(
                                                   GLKMatrix3InvertAndTranspose(GLKMatrix4GetMatrix3(modelViewMatrix), NULL))];
    [background setModelViewProjectionMatrix: NSStringFromGLKMatrix4(GLKMatrix4Multiply(projectionMatrix, modelViewMatrix))];

    [ninja update:dt];
    [bonus update:dt];
}
```

As you can already tell, how convenient Message Forwarding has made it for us to add our custom code in Components and just directly call them, even though the GameObject class doesn’t directly implements them.

I believe, you can extend this idea to add more Components and rapidly add and use the functionality, while keeping the code logically separated by Components.

The accompanied code is available at https://github.com/chunkyguy/OjbCComponentSystem. Since, this was just a rapid prototype test, and we have disable the ARC, the code could have potential memory leaks. Don’t use the code directly, use it just for reference purposes.
