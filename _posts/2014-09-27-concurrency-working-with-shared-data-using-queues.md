---
layout: post
title:  "Concurrency: Working with shared data using queues"
date:   2014-09-27 23:28:54 +0530
categories: concurrency
---

Concurrency: Working with shared data using queues
===================================================

All problems with concurrency basically boil down to two categories: Race conditions and deadlocks. Today let’s look into one specific problem under race conditions: Working with shared data.

If every thread had to deal with the only data provided exclusively to it, we won’t ever get into any sort of data race problems. In fact, this is one of the main features of functional programming languages like Haskell that advertises heavily about concurrency and parallelism.

But there’s nothing to be worried about, functional programming is more about adopting a design pattern, just as object oriented programming is. Yes, some programming languages offer more towards a particular pattern, but that doesn’t means that we can not adopt that pattern in any other language. We just have to restrict ourselves to a particular convention. For example, there are a plenty or large softwares written with C programming language adopting the object oriented pattern. And for us, both C++ and Swift are very much capable of adopting the functional programming paradigm.

Getting back to the problem of shared data. Let’s look into when is shared data a problem with multiple threads. Suppose we have an application where two or more threads are working on a shared data. If all each threads ever does is a read activity, we won’t have any data race problem. For example, you’re at the airport and checking out your flight status on the giant screen. And, you’re not the only one looking at the status screen. But, does it really matters how many others are reading the information from the same screen? Now suppose the scenario is a little more realistic. Every airliner has a control with which they can update the screen with new information, while a passenger is reading the screen, the situation can get a little difficult. At any given instance, one or more airliner can be updating the screen at same instance, which could result in some garbled text to the passengers reading the screen. The two or airliners are actually in a race condition for a single resource, the screen.

If you’re using Cocoa, you must be familiar with the NSMutableArray. Also, you must be familiar that NSMutableArray isn’t thread-safe as of this writing. So, the question is how do make use of NSMutableArray in a situation like this, where we have concurrent read and write operations going on?

Let’s start with a simple single threaded model of what we’re doing:

/**
 * Concurreny Experiments: Shared data
 * Compile: clang SharedData.m -framework Foundation && ./a.out
 */

#import <Foundation/Foundation.h>

@interface Airport : NSObject {
    NSMutableArray *airportStatusBoard;
    NSUInteger writeCount;
    NSUInteger readCount;
}
@end

@implementation Airport

- (id)init
{
    self = [super init];
    if (!self) {
        return nil;
    }
    
    airportStatusBoard = [[NSMutableArray alloc] init];
    writeCount = 0;
    readCount = 0;
    
    return self;
}

- (void)dealloc
{
    [airportStatusBoard release];
    [super dealloc];
}

- (void)addFlightEntry
{
    NSNumber *newEntry = @(random() % 10);
    [airportStatusBoard addObject:newEntry];
}

- (void) statusWriter
{
    NSLog(@"writing begin: %@", @(writeCount));
    [self addFlightEntry];
    NSLog(@"writing ends: %@", @(writeCount));
    writeCount++;
}

- (BOOL) statusReaderWithFlightNumber:(NSNumber *)searchFlightNum
{
    NSLog(@"reading begin: %@", @(readCount));
    
    BOOL found = NO;
    for (NSNumber *flightNum in airportStatusBoard) {
        if ([flightNum isEqualToNumber:searchFlightNum]) {
            found = YES;
            break;
        }
    }
    
    NSLog(@"reading ends: %@", @(readCount));
    
    readCount++;
    
    return found;
}

- (void) serialLoop
{
    srandom(time(0));
    while (![self statusReaderWithFlightNumber:@(7)]) {
        [self statusWriter];
    }
    
    NSLog(@"Flight found after %@ writes and %@ reads", @(writeCount), @(readCount));
}

@end

int main()
{
    @autoreleasepool {
        Airport *airport = [[Airport alloc] init];
        [airport serialLoop];
    }
    
    return 0;
}
Here’s the output for the above program on my machine:


2014-09-27 14:32:12.636 a.out[9601:507] reading begin: 0
2014-09-27 14:32:12.638 a.out[9601:507] reading ends: 0
2014-09-27 14:32:12.639 a.out[9601:507] writing begin: 0
2014-09-27 14:32:12.639 a.out[9601:507] writing ends: 0
2014-09-27 14:32:12.640 a.out[9601:507] reading begin: 1
2014-09-27 14:32:12.640 a.out[9601:507] reading ends: 1
2014-09-27 14:32:12.641 a.out[9601:507] writing begin: 1
2014-09-27 14:32:12.641 a.out[9601:507] writing ends: 1
2014-09-27 14:32:12.642 a.out[9601:507] reading begin: 2
2014-09-27 14:32:12.642 a.out[9601:507] reading ends: 2
2014-09-27 14:32:12.643 a.out[9601:507] Flight found after 2 writes and 3 reads

As you can observe, we’re performing sequential writes and reads until we find the required flight number in the array, and everything works out great. But, if we now try to run each write and read concurrently. To do that we create a queue for reading and writing. We perform all the reading and writing operation on a concurrent queue. We’re trying to implement the scenario where one passenger (you) is interested in reading the status board from top to bottom and check if their flight is listed. When they reach the end of the list and did not find the flight number they start again from the top. While, many airliner services are trying to update the single status board concurrently.

readWriteQueue = dispatch_queue_create("com.whackylabs.concurrency.readWriteQ", DISPATCH_QUEUE_CONCURRENT);

- (void)parallelLoop
{
    srandom(time(0));

    __block BOOL done = NO;
    while(!done) {
        
        /* perform a write op */
        dispatch_async(readWriteQueue, ^{
            [self statusWriter];
        });
        
        /* perform a read op */
        dispatch_async(readWriteQueue, ^{
            if ([self statusReaderWithFlightNumber:@(SEARCH_FLIGHT)]) {
                done = YES;
            }
        });
    }
    
    NSLog(@"Flight found after %@ writes and %@ reads", @(writeCount), @(readCount));
}
If you now try to execute this program, once in a while you might get the following error:


Terminating app due to uncaught exception 'NSGenericException', reason: '*** Collection <__NSArrayM: 0x7f8f88c0a5d0> was mutated while being enumerated.'

This is the tricky part about debugging race conditions, most of the time your application might to be working perfectly, unless you deploy it on some other hardware or you get unlucky, you won’t get the issues. One thing you can play around with is, deliberately sleeping a thread to cause race conditions with sprinkling thread sleep code around for debugging:

- (void) statusWriter
{
    [NSThread sleepForTimeInterval:0.1];
    writeCount++;
    NSLog(@"writing begin: %@", @(writeCount));        
    [self addFlightEntry];
    NSLog(@"writing ends: %@", @(writeCount));
}
If the above code doesn’t crashes on your machine on the first go, edit the sleep interval and/or give it a few more runs.

Now coming to the solution of such a race condition. The solution to such problem is in two parts. First is the write part. When multiple threads are trying to update the shared data, we need a way to lock the part where the actual update happens. It’s something like when multiple airliners are trying to update the single screen, we can figure out a situation like, only one airliner has the controls required to update the screen, and when it’s done, it releases the control to be acquired by any other airliner in queue.

Second, is the read part. We need to make sure that when we’re performing a read operation some other thread isn’t mutating the shared data as NSEnumerator isn’t thread-safe either.

A way to acquire such a lock with GCD is to use dispatch_barrier()

- (void) statusWriter
{
    dispatch_barrier_async(readWriteQueue, ^{
        
        [NSThread sleepForTimeInterval:0.1];
        
        NSLog(@"writing begin: %@", @(writeCount));
        
        [self addFlightEntry];
        
        NSLog(@"writing ends: %@", @(writeCount));
        writeCount++;
        
    });
}

- (BOOL) statusReaderWithFlightNumber:(NSNumber *)searchFlightNum
{
   __block BOOL found = NO;
    
    dispatch_barrier_async(readWriteQueue, ^{
        
        NSLog(@"reading begin: %@", @(readCount));
        
        for (NSNumber *flightNum in airportStatusBoard) {
            if ([flightNum isEqualToNumber:searchFlightNum]) {
                found = YES;
                break;
            }
        }
        
        NSLog(@"reading ends: %@", @(readCount));
        
        readCount++;
        
    });
    
    return found;
}
If you try to run this code now, you’ll observe that the app doesn’t crash anymore, but the app has become significantly slower.

The first performance improvement that we can make is to not return from the read code immediately. Instead, we can let the queue handle the completion:

- (void) statusReaderWithFlightNumber:(NSNumber *)searchFlightNum completionHandler:(void(^)(BOOL success))completion
{
    
    dispatch_barrier_async(readWriteQueue, ^{
        
        NSLog(@"reading begin: %@", @(readCount));
        
        BOOL found = NO;
        
        for (NSNumber *flightNum in airportStatusBoard) {
            if ([flightNum isEqualToNumber:searchFlightNum]) {
                found = YES;
                break;
            }
        }
        
        NSLog(@"reading ends: %@", @(readCount));
        
        readCount++;
       
        completion(found);
    });
}

- (void)parallelLoop
{
    srandom(time(0));
    
    __block BOOL done = NO;
    while(!done) {
        
        /* perform a write op */
        dispatch_async(readWriteQueue, ^{
            [self statusWriter];
        });
        
        /* perform a read op */
        dispatch_async(readWriteQueue, ^{
            [self statusReaderWithFlightNumber:@(SEARCH_FLIGHT) completionHandler:^(BOOL success) {
                done = success;
            }];
        });
    }
    
    NSLog(@"Flight found after %@ writes and %@ reads", @(writeCount), @(readCount));
}

Another thing to note is that, our code is no longer executing in parallel. As each dispatch_barrier() blocks the queue until all the pending tasks in the queue are complete. As is evident from the final log.

Flight found after 154 writes and 154 reads
In fact, the processing could be even worse than just running it sequentially, as the queue has to take care of the locking and unlocking overhead.

We can make the read operation as non-blocking again, as blocking the reads is not getting us any profit. One way to achieve that is to copy the latest data within a barrier and then use the copy to read the data while the other threads update the data.

- (BOOL) statusReaderWithFlightNumber:(NSNumber *)searchFlightNum
{
    
    NSLog(@"reading begin: %@", @(readCount));
    
    __block NSArray *airportStatusBoardCopy = nil;
    dispatch_barrier_async(readWriteQueue, ^{
        
        airportStatusBoardCopy = [airportStatusBoard copy];
        
    });
    
    
    BOOL found = NO;
    
    for (NSNumber *flightNum in airportStatusBoardCopy) {
        if ([flightNum isEqualToNumber:searchFlightNum]) {
            found = YES;
            break;
        }
    }

    if (airportStatusBoardCopy) {
        [airportStatusBoardCopy release];
    }
    
    NSLog(@"reading ends: %@", @(readCount));
    
    readCount++;
    
    return found;
}
But, this won’t solve the problem. Most of reads are probably a waste of time. What we actually need is a way to communicate between the write and read. The actual read should only happen after the data has been modified with some new write.

We can start by separating the read and write tasks. We actually need the read operations to be serial, as we are implementing the case where a passenger is reading the list top-down and when they reach at the end, they start reading from the beginning. Whereas, many airliners can simultaneously edit the screen, so they still need to be in a concurrent queue.

readQueue = dispatch_queue_create("com.whackylabs.concurrency.readQ", DISPATCH_QUEUE_SERIAL);
writeQueue = dispatch_queue_create("com.whackylabs.concurrency.writeQ", DISPATCH_QUEUE_CONCURRENT);

- (void)parallelLoop
{
    srandom(time(0));
    
    __block BOOL done = NO;
    while(!done) {
        
        /* perform a write op */
        dispatch_async(writeQueue, ^{
            [self statusWriter];
        });
        
        /* perform a read op */
        dispatch_async(readQueue, ^{
            [self statusReaderWithFlightNumber:@(SEARCH_FLIGHT) completionHandler:^(BOOL success) {
                done = success;
            }];
        });
    }
    
    NSLog(@"Flight found after %@ writes and %@ reads", @(writeCount), @(readCount));
}

Now, in the read operation we take a copy of the shared data. The way we can do it is by having an atomic property. Atomicity guarantees that the data is either have be successful or unsuccessful, there won’t be any in-betweens. So, in all cases we shall have a valid data all the time.

@property (atomic, copy) NSArray *airportStatusBoardCopy;
In each read operation, we simply copy the shared data, so don’t care about mutability anymore.

self.airportStatusBoardCopy = airportStatusBoard;
And further on we can use the copy to execute the read operations.

2014-09-27 16:20:33.145 a.out[12855:507] Flight found after 11 writes and 457571 reads
This is the queue level solution of the problem. In our next update we will take a look at how we can solve the problem at thread level using C++ standard thread library.
As always, the code for today’s experiment is available online at the github.com/chunkyguy/ConcurrencyExperiments.

Posted in concurrency, _dev	| Tagged cocoa, Concurrency, GCD, ObjC	| 0 Comments
Component System using C++ Multiple Inheritance
Posted on September 23, 2014 by chunkyguy
Few days back, I talked about designing Component System using Objective-C and message forwarding. Today, to carry the conversation forward, we shall see how can the same Component System be designed with C++, and also why C++ is a better language choice for such a design.

Before I begin, let’s copy-paste the abstract from our earlier post, on why and when do we actually need a Component System:

I wanted a component system where I’ve a GameObject that can have one or more components plugged in. To illustrate the main problems with the traditional Actor based model lets assume we’re developing a platformer game. We have these three Actor types:

1. Ninja: A ninja is an actor that a user can control with external inputs.
2. Background: Background is something like a static image.
3. HiddenBonus: Hidden bonus is a invisible actor that a ninja can hit.

Source: Swim Ninja
Source: Swim Ninja
Let’s take a look at all the components inside these actors.

Ninja = RenderComponent + PhysicsComponent
Background = RenderComponent
HiddenBonus = PhysicsComponent.

Here RenderComponent is responsible for just drawing some content on the screen.
while the PhysicsComponent is responsible for just doing physics updates like, collision detection.

So, in a nutshell from our game loop we need to call things like

void Scene::Update(const int dt)
{
    ninja.Update(dt);
    hiddenBonus.Update(dt);
}
 
void Scene::Render()
{
    background.Render();
    ninja.Render();
}
Now, in the traditional Actor model, if we have an Actor class with both RenderComponent and PhysicsComponent like:

	
class Actor {
public:
    void Update(const int dt);
    void Render();
};

    Actor ninja;
    Actor hiddenBonus;
    Actor background;
Then it would inevitably add a PhysicsComponent to background actor and a RenderComponent to a hiddenBonus actor. So, the user of the code has to keep a mental track of what functions shouldn’t be invoked on what objects. Which is as awful in reality, as it sounds in theory.

We could use the same approach we used when designing the Component System with Objective-C, by having a function on fat GameObject that could be used to enable a particular component. Think of each GameObject as an collection of many components. Where each component know how to deal with one thing, like rendering, physics, AI, networking, and so on. When creating a new GameObject we need a method to specify what components should be enabled for it. The fat interface technique we’ve already discussed in that last experiment. Let’s try something new, we’re using C++ after all. Let’s start with breaking the components as separate classes:

class RenderComponent {
public:
    void Render();
};

class PhysicsComponent {
public:
    void Update(const int dt);
};
Then we can easily create composite components using such core components, such as:

class RenderAndPhysicsComponent : public RenderComponent, public PhysicsComponent
{};
Now, we can update our game scene’s interface as:

    RenderAndPhysicsComponent ninja;
    PhysicsComponent hiddenBonus;
    RenderComponent background;
This definitely serves our immediate needs, but there is some problem. This solution doesn’t scales well. Whenever we come up with a new component, we’ve to update your related base class accordingly and create a mix for each possible basic component interfaces. For five core components, we would get a total of C(5,1) + C(5,2) + C(5,3) + C(5,4) + c(5,5) = 5 + 10 + 10 + 5 + 1 = 31 total component interfaces combinations possible.

In C++, we can actually do better. This is because C++ has one thing that other languages just don’t want. It’s called multiple inheritance.

Instead of making the a composite collection class a direct collection of many components, we can instead have a single GameObject class that could inherit from as many components as desired. And, further to avoid having the fat abstraction issue, we can have the GameObject to be inherited at compile time from only the components that we actually need for that particular object.

How do we design such a GameObject? One correct answer is using variadic templates:

template <typename... Component>
class GameObject : public Component... {
public:
    GameObject() :
    Component()...
    {}
    
    GameObject(Component... component) :
    Component(component)...
    {}
};
This GameObject has nothing of its own. A host class that wishes to use a instance of this object has to supply a concrete class as a template parameter. This has given infinite flexibility to our GameObject.

We can now update our interface to easily use the core components to construct our GameObject instances that fit our needs:

    GameObject<RenderComponent, PhysicsComponent> ninja;
    GameObject<PhysicsComponent> hiddenBonus;
    GameObject<RenderComponent> background;
If that seems to verbose to write every time, we can in fact use typedefs to create some most common types.

    typedef GameObject<RenderComponent> Sprite;
    typedef GameObject<PhysicsComponent> Trigger;
    typedef GameObject<RenderComponent, PhysicsComponent> Actor;
    
    Actor ninja;
    Trigger hiddenBonus;
    Sprite background;
Now, I know that feels already too awesome, but we still have a problem. In practice we often need to communicate between components. Like say, we have a TransformComponent, that just provides us a model matrix based on the current transform of the GameObject.

class TransformComponent {
public:
    /** Get the model matrix of the transform to take things from model space to
     * the world space
     */
    GLKMatrix4 GetModelMatrix() const;
    
    GLKVector2 position;
    float rotation;
};
And, that model matrix then could be used by the RenderComponent for rendering the mesh.

class RenderComponent {
public:

    void Render(const Uniform &uniform,
                const Camera &camera) const;

    GLKMatrix4 modelMatrix;
    Mesh mesh;
};

void RenderComponent::Render(const Uniform &uniform,
                             const Camera &camera) const
{
    GLKMatrix4 modelViewProjectionMatrix = GLKMatrix4Multiply(camera.GetProjectionMatrix(),
                                                              GLKMatrix4Multiply(camera.GetViewMatrix(),
                                                                                 modelMatrix));
    glUniformMatrix4fv(uniform.um4k_Modelviewprojection, 1, 0, modelViewProjectionMatrix.m);
    mesh.Draw();
}

So, how do go with that design?

To solve that, we actually need another component. Let’s call it CustomComponent. The entire purpose of CustomComponent is to have an Update function:

class CustomComponent {
public:
    void Update(const int dt);
    
protected:
    std::function<void(const int dt)> customUpdate;
};
Now we can use our CustomComponent to bind many other components with something like:

void Ninja::Load()
{
	/* ... */
    customUpdate_ = std::bind(&Ninja::Update, this, std::placeholders::_1);
}
void Ninja::Update(const int dt)
{
	/* from transform component */
	GLKMatrix4 modelMatrix = GetModelMatrix();
	/* to render component */
    SetModelMatrix(modelMatrix);
}

No matter how absurd this code looks, it works. We can improve this, and while doing that we shall also address another important issue with multiple inheritance, name clashing.

Since, our GameObject is now split into multiple files we can easily get name clashing. For example both RenderComponent and TransformComponent can have a member named

GLKMatrix4 modelMatrix;
Or, both PhysicsComponent and CustomComponent can have a member function named:

void Update(const int dt);
How do we resolve this issue? One way is to adopt a naming convention, like say every PhysicsComponent will have a ‘phy’ prefix appended before every member function and data member. That would reduce our code to:

void Ninja::Update(const int dt)
{
	/* from transform component */
	GLKMatrix4 modelMatrix = transGetModelMatrix();
	/* to render component */
    rendrSetModelMatrix(modelMatrix);
}
We all agree that following a naming convention is very hard. So this solution might not work in long term. A better way out is to encapsulate all the core logic out of the component to another core class. The TransformComponent could be reduced to:

class TransformComponent {
public:
    /** Create default Transform component */
    TransformComponent();
    
protected:
    Transform transform_;
};

class Transform {
public:
    /** Create default Transform */
    Transform(const GLKVector2 &position = GLKVector2Make(0.0f, 0.0f),
              const Degree rotation = 0.0f);

    GLKVector2 GetPosition() const;
    
    void SetPosition(const GLKVector2 &position);
    
    /** Get angle in degrees */
    float GetRotation() const;
    
    /** Set rotation in degrees
     * @param rotation Angle in degrees.
     */
    void SetRotation(const Degree rotation);
    
    /** Get the model matrix of the transform to take things from model space to
     * the world space
     */
    GLKMatrix4 GetModelMatrix() const;
    
private:
    GLKVector2 position_;
    Degree rotation_;
};
With such pattern implemented, our same code would look like:

void Ninja::Update(const int dt)
{
    renderer_.SetModelMatrix(transform_.GetModelMatrix());
}
OK, with that taken care of, let’s now focus on the fitting the pieces together. Let’s begin with updating the commonly used GameObject configurations to proper C++ types:

class Actor : public GameObject <TransformComponent, PhysicsComponent, RenderComponent, CustomComponent> {
public:
    void Load(const Transform &trans, const Mesh &mesh);
    void Update(const int dt);
};

class Sprite : public GameObject<TransformComponent, RenderComponent, CustomComponent> {
public:
    void Load(const Transform &trans, const Mesh &mesh);
    void Update(const int dt);
};

class Trigger : public GameObject<TransformComponent, PhysicsComponent, CustomComponent> {
public:
    void Load(const Transform &trans);
    void Update(const int dt);
};

void Scene::Load(const GLKVector2 &screen)
{
    LoadShader();
    LoadMesh();
    
    /* load entities */
    background_.Load(Transform(), meshMap_[0]);
    ninja_.Load(Transform(), meshMap_[1]);
    hiddenBonus_.Load(Transform());
}

Lets now implement the loop. The trickiest part of the loop is that we need to call Update() on every object that has a CustomComponent and Render() on objects that implement RenderComponent. Notice, so far we haven’t implemented any virtual functions, so can not depend on the runtime to select the correct function. What we actually need is something like:

void Scene::Update(const int dt)
{
    /* update all custom components */
}

void Scene::Render()
{
    glClear(GL_COLOR_BUFFER_BIT);
    
    /* bind all pieces to the GL pipeline */
    meshMem_.Enable();
    shader_.Enable();
    
    /* render all render components */
}
What we actually need some sort of container class that hold all types of Components. The first thought that hits the brain is to have a std::vector of Component of every kind. And from our experience we already know this sort of patterns doesn’t scales well. But using the power of C++11 and tuples we implement that in a much better way. Let’s add a ContainerComponent as:

template <typename... Component>
class ContainerComponent {
protected:
    std::tuple<std::vector<Component*>...> children_;
};
As you can see, ContainerComponent is capable of holding any number of components of any size. Think of it as a class that can possess compile time flexibility to hold any kind of components and at runtime we can add and remove instances of those component classes.

Now we can update our plain Scene class to be a GameObject that has a ContainerComponent. The ContainerComponent is capable of holding two kind of Components, the CustomComponent and the RenderComponent:

class Scene : public GameObject <
	ContainerComponent<
		CustomComponent,
		RenderComponent
>> {
	/* ... */
};
The only important thing to remember is that the order matters. Because this is a tuple we have to remember the index of Component type.

Our new updated Load() now looks like:

void Scene::Load(const GLKVector2 &screen)
{
    LoadShader();
    LoadMesh();
    
    /* load entities */
    background_.Load(Transform(), meshMap_[0]);
    std::get<0>(children_).push_back(&background_);
    std::get<1>(children_).push_back(&background_);
    
    ninja_.Load(Transform(), meshMap_[1]);
    std::get<0>(children_).push_back(&ninja_);
    std::get<1>(children_).push_back(&ninja_);

    hiddenBonus_.Load(Transform());
    std::get<0>(children_).push_back(&hiddenBonus_);
}
The power of C++ compile time checks comes into play when we do something illegal, such as:

std::get<1>(children_).push_back(&hiddenBonus_);
The above expression will throw an error at compile time because the hiddenBonus_ does not has a RenderComponent and we’re trying to add it to a vector of RenderComponents.

So far so good. Let’s make the code more readable by removing the hard-coded integers. If we introduce a enum type as:

enum ContainerIndex {
    Custom = 0,
    Render = 1
};
We can do things like:

void Scene::Load(const GLKVector2 &screen)
{
    LoadShader();
    LoadMesh();
    
    /* load entities */
    background_.Load(Transform(), meshMap_[0]);
    std::get<ContainerIndex::Custom>(children_).push_back(&background_);
    std::get<ContainerIndex::Render>(children_).push_back(&background_);
    
    ninja_.Load(Transform(), meshMap_[1]);
    std::get<ContainerIndex::Custom>(children_).push_back(&ninja_);
    std::get<ContainerIndex::Render>(children_).push_back(&ninja_);

    hiddenBonus_.Load(Transform());
    std::get<ContainerIndex::Custom>(children_).push_back(&hiddenBonus_);
}
Now getting back to our loop. We can simply do things like:

void Scene::Update(const int dt)
{
    /* update all custom components */
    for (auto customComponent : std::get<ContainerIndex::Custom>(children_)) {
        customComponent->Update(dt);
    }
}

void Scene::Render()
{
    glClear(GL_COLOR_BUFFER_BIT);
    
    /* bind all pieces to the GL pipeline */
    meshMem_.Enable();
    shader_.Enable();
    
    /* render all render components */
    for (const auto renderComponent : std::get<ContainerIndex::Render>(children_)) {
        renderComponent->Render(uniform_, camera_);
    }
}
Here is the result of this prototype:

Hosted by imgur.com

The yellow color is applied to the render-only background, while the cyan is the ‘render+physics’ ninja.

I haven’t implemented any physics elements, but in a real game with a Physics engine implemented the PhysicsComponent should look something like:

class PhysicsComponent {
public:
    PhysicsComponent();
    
protected:
    b2Body *physicsBody_;
};

class Physics {
public:
    bool Load(
              const GLKVector2 &screenSize,
              const GLKVector2 &gravity
              );
    
    b2World *GetWorld() const;
    
private:
    std::unique_ptr<b2World> physicsWorld_;    
};
And any GameObject that listens to physics updates should look something like:

void Ball::Update(const int dt)
{
    transform_.SetPosition(PhysicsConverter::GetPixels2(physicsBody_->GetPosition()));
    renderer_.SetModelMatrix(transform_.GetModelMatrix());
}
And then the typical Scene class’s Update() should look like:


bool GameScene::Load(const GLKVector2 &screen)
{
    /* enable physics */
    physics_.Load(screen, GLKVector2Make(0.0f, 0.0f));

	/* more loading ... */
}

void GameScene::Update(const int dt)
{
    /* update physics */
    physics_.GetWorld()->Step(1.0f/FPS, 10, 10);
    physics_.GetWorld()->ClearForces();
    
    /* update all custom components */
    for (auto customComponent : std::get<ContainerIndex::Custom>(children_)) {
        customComponent->Update(dt);
    }
    
    /* update camera */
    camera_.SetPan(ball_.GetTransform().GetPosition());
}

I hope this gives and idea of how can we use the enormous power of C++ to build an game engine based on component system. This also shows why C++ is such a great language for such a pattern when compared to something like Objective-C.

The entire code for this experiment is available at https://github.com/chunkyguy/CppComponentSystem. Please feel free to checkout and start playing around.

Posted in OpenGL, _dev, _games	| Tagged C, Design Patterns	| 0 Comments
Concurrency: Spawning Independent Tasks
Posted on September 21, 2014 by chunkyguy
First step in getting with concurrency is spawning tasks in background thread.

Lets say we’re building a chat application where a number of users chat in a common room. In this sort of scenario every user is probably chatting at the same instance. For sake of simplicity let’s say each user can only have a single character username. And obviously such a constraint is definitely going to outrage the users. So, pissed off as they are, they’re just printing their usernames over and over again.

To support such a functionality, we might need a function like such:

void PrintUsername(const char uname, const int count)
{
    for (int i = 0; i < count; ++i) {
        std::cout << uname << ' ';
    }
    std::cout << std::endl;
}
In a single threaded environment, whatever user invokes this method gets the control of the CPU and their username is going to get printed until the loop expires.

If there are two users in the chat room with usernames ‘X’ and ‘Y’, each wants to print their username 10 times. The calling code might look something like:


void SingleThreadRoom()
{
    PrintUsername('X', 10);
    PrintUsername('Y', 10);
}
The output is as expected:

X X X X X X X X X X 
Y Y Y Y Y Y Y Y Y Y 
Now, to make the app slightly better, let’s give both the users equal weights. Each user should be able to perform the same task in their own thread. Simplest way to spawn a new thread is by calling join()


void MultiThreadRoom()
{
    std::thread threadX(PrintUsername, 'X', 10);
    std::thread threadY(PrintUsername, 'Y', 10);
    
    threadX.join();
    threadY.join();
}
On my machine this outputs as:

YX  YX  YX  YX  YX  YX  YX  YX  YX  YX  

Calling join() on a thread attaches the thread to the calling thread. When the app launches we already get the main thread, and since we’re calling join() on the main thread, the two spawned threads get joined to the main thread.

If we need a thread for some task that is of kind fire-and-forget, we can call detach(). But, if we call detach() we’ve no control of the thread lifetime anymore. It’s something like the main thread doesn’t cares anymore.

For a program like:

void MultiThreadRoomDontCare()
{
    std::thread threadX(PrintUsername, 'X', 10);
    std::thread threadY(PrintUsername, 'Y', 10);
    
    threadX.detach();
    threadY.detach();
}

int main()
{
    MultiThreadRoomDontCare();
    std::cout << "MainThread: Goodbye" << std::endl;
    return 0;
}
The output on my machine is:

MainThread: Goodbye
XY  XY
As you can see the main thread doesn’t care if it’s child thread are still not finished.

Getting over to Swift, we don’t have to manually track the threads, but think in more higher terms as tasks and queues.

First problem we face when trying out async code in Playground is that the NSRunLoop isn’t running in Playground as it is with out native apps. The solution is to import the XCPlayground and call XCPSetExecutionShouldContinueIndefinitely() method. All thanks to the internet.

Here’s a test code that works:

import UIKit
import XCPlayground

XCPSetExecutionShouldContinueIndefinitely(continueIndefinitely: true)

NSOperationQueue.mainQueue().addOperation(NSBlockOperation(block: { () -> Void in
    println("Hello World!!")
}))

var done = "Done"
Since, we’re talking in terms of queues and tasks, we have to change the way of thinking. Now, we don’t care about threads anymore, only tasks. The task is to print the username. This task is going to be submitted to the queue and the queue internally shall manage the threads.

A code like:

func printUsername(uname:String)
{
        println(uname)
}

let chatQueue:NSOperationQueue = NSOperationQueue()

func multiThreadRoom()
{
    for (var i = 0; i < 5; ++i) {
        chatQueue.addOperation(NSBlockOperation(block: { () -> Void in
            printUsername("X")
        }))
        
        chatQueue.addOperation(NSBlockOperation(block: { () -> Void in
            printUsername("Y")
        }))
    }
}

multiThreadRoom()

println("Main Thread Quit")
Prints on my machine as:

X
Y
X
Y
X
Y
X
Y
X
Y
Main Thread Quit
This is the very basics of spawning concurrent independent tasks. If all your requirements are working on independent set of tasks, this is almost all you need to learn about concurrency to get the job done. I hope the real world were this simple ;)

As always, the entire code is also available online at: https://github.com/chunkyguy/ConcurrencyExperiments

Posted in concurrency, _dev	| Tagged C, Concurrency, Swift, Threads	| 0 Comments
Concurrency: Getting started
Posted on September 14, 2014 by chunkyguy
The time everybody was speculating for so long has finally arrived. The processors aren’t getting any faster. The processors are hitting the physical thresholds. If we try to run the processors, as they are with current technology, they’re getting over-heated. Moore’s law is breaking down. Quoting Herb Sutter, ‘The free lunch is over‘.

If you want your code to run faster on new generation of hardware, you have to consider concurrency. All modern computing devices have multi-cores in built. More and more languages, toolkits, libraries, frameworks are providing support of concurrency. Let’s talk about concurrency.

Starting from today, I’ll be experimenting with multithreading. I’ll be using the following technologies. First is the C++11 thread library that ships with the C++11 standard. Second is the libdispatch or Grand Central Dispatch (GCD) that is developed by Apple. Although, GCD isn’t a pure threading library. It’s more of a concurrency library that is built over threads. In GCD we normally think in terms of tasks and queues. It’s a level above threads. And, the third is NSOperation. NSOperation is another level up from GCD. NSOperation is pure OOP oriented concurrency API. I’ve used it quite often with Objective-C and I’m sure it works as smoothly with Swift.

Let’s start with our first experiment. Dividing a huge task into smaller chunks and let them run in parallel to each other.

So, first question, what task should we pick. Let’s pick sorting of a dataset. Lets say, we’ve an huge array and we want to sort it. We could do it using the famous Quicksort algorithm as:

    template<typename T>
    void QuickSort(std::vector<T> &arr, const size_t left, const size_t right)
    {
        if (left >= right) {
            return;
        }
        
        /* move pivot to left */
        std::swap(arr[left], arr[(left+right)/2]);
        size_t last = left;
        
        /* swap all lesser elements */
        for (size_t i = left+1; i <= right; ++i) {
            if (arr[i] < arr[left]) {
                std::swap(arr[++last], arr[i]);
            }
        }
        
        /* move pivot back */
        std::swap(arr[left], arr[last]);

        /* sort subarrays */
        QuickSort(arr, left, last);
        QuickSort(arr, last+1, right);
    }
The algorithm is pretty straightforward. Given a list of unsorted data, in each iteration we pick a pivot element and sort that element by moving all smaller elements on the left side and all larger elements on right. By the end of the iteration, our pivot element is in the sorted position with we have two unsorted subarrays.

This algorithm is a prefect pick for concurrency. As in each iteration we get our problem divided into two independent sub problems that we can easily execute in parallel.

Here’s my first approach:

    template<typename T>
    void QuickerSort(std::vector<T> &arr, const size_t left, const size_t right)
    {
        if (left >= right) {
            return;
        }
        
        /* move pivot to left */
        std::swap(arr[left], arr[(left+right)/2]);
        size_t last = left;
        
        /* swap all lesser elements */
        for (size_t i = left+1; i <= right; ++i) {
            if (arr[i] < arr[left]) {
                std::swap(arr[++last], arr[i]);
            }
        }
        
        /* move pivot back */
        std::swap(arr[left], arr[last]);
        
        /* sort subarrays in parallel */
        auto taskLeft = std::async(QuickerSort<T>, std::ref(arr), left, last);
        auto taskRight = std::async(QuickerSort<T>, std::ref(arr), last+1, right);
    }
And here’s my calling code:

template <typename T>
void UseQuickSort(std::vector<T> arr)
{
    clock_t start, end;
    start = clock();
    QuickSort(arr, 0, arr.size() - 1);
    end = clock();
    std::cout << "QuickSort: " << (end - start) * 1000.0/double(CLOCKS_PER_SEC) << "ms" << std::endl;
    assert(IsSorted(arr));
}

template <typename T>
void UseQuickerSort(std::vector<T> arr)
{
    clock_t start, end;
    start = clock();
    auto bgTask = std::async(QuickerSort<T>, std::ref(arr), 0, arr.size()-1);
    if (bgTask.wait_for(std::chrono::seconds(0)) != std::future_status::deferred) {
        while (bgTask.wait_for(std::chrono::seconds(0)) != std::future_status::ready) {
            std::this_thread::yield();
        }
        assert(IsSorted(arr));
        end = clock();
        std::cout << "QuickerSort: " << (end - start) * 1000.0/double(CLOCKS_PER_SEC) << "ms" << std::endl;
    }
}

void main()
{
    
    std::vector<int> arr;
    for (int i = 0; i < 100; ++i) {
        arr.push_back(rand()%10000);
    }

    UseQuickSort(arr);
    UseQuickerSort(arr);
}
I’ve added some code to profile how long does each of these tasks take to execute the data set. The dataset of 100 elements executes on my machine as:

QuickSort: 0.023ms
QuickerSort: 29.12ms
As you can see, the concurrent code takes a lot longer to execute than the single threaded one. One guess could be that the creating thread for each subproblem and them the extra load of context switching between the threads could be the reason behind the latency. Or you could also suggest that there are huge flaws in my implementation.

I accept, there could be huge flaws in my implementation, as this is my first take at core multi-threading with C++ thread library. But, still we can’t ignore the fact that we’re spawning a lot of threads with each iteration. The number of threads running concurrent at each iteration are getting added by a factor of 2. This implementation definitely needs to be improved.

This is where GCD shines. First of all, it’s easier to begin with. Keep in mind I’ve equal amount of experience with both the libraries. So, my code should be full of bugs and most probably I’m not using them in the best possible way. But, this is all part of the test and my experiments.

Anyhow, when I implement the same problem using Swift and GCD, like so:


import UIKit

// MARK: Common

func isSorted(array:[Int]) -> Bool
{
    for (var i = 1; i < array.count; ++i) {
        if (array[i] < array[i-1]) {
            println("Fail case: \(array[i]) > \(array[i-1])")
            return false
        }
    }
    return true
}

//MARK: Quick Sort

func quickSort(inout array:[Int], left:Int, right:Int)
{
    if (left >= right) {
        return;
    }
    
    /* swap pivot to front */
    swap(&array[left], &array[(left+right)/2])
    var last:Int = left;
    
    /* swap lesser to front */
    for (var i = left+1; i <= right; ++i) {
        if (array[i] < array[left]) {
            swap(&array[i], &array[++last])
        }
    }
    
    /* swap pivot back */
    swap(&array[left], &array[last])
    
    /* sort subarrays */
    quickSort(&array, left, last)
    quickSort(&array, last+1, right)
}

func useQuickSort(var array:[Int])
{
    let start:clock_t = clock();
    quickSort(&array, 0, array.count-1)
    let end:clock_t = clock();
    
    println("quickSort: \((end-start)*1000/clock_t(CLOCKS_PER_SEC)) ms")
    assert(isSorted(array), "quickSort: Array not sorted");
}

//MARK: Quicker Sort

let sortQueue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_BACKGROUND, 0)


func quickerSort(inout array:[Int], left:Int, right:Int)
{
    if (left >= right) {
        return;
    }
    
    /* swap pivot to front */
    swap(&array[left], &array[(left+right)/2])
    var last:Int = left;
    
    /* swap lesser to front */
    for (var i = left+1; i <= right; ++i) {
        if (array[i] < array[left]) {
            swap(&array[i], &array[++last])
        }
    }
    
    /* swap pivot back */
    swap(&array[left], &array[last])
    
    /* sort subarrays */
    dispatch_sync(sortQueue, { () -> Void in
        quickSort(&array, left, last)
    })
    dispatch_sync(sortQueue, { () -> Void in
        quickSort(&array, last+1, right)
    })
}

func useQuickerSort(var array:[Int])
{

    let start:clock_t = clock();

    dispatch_sync(sortQueue, { () -> Void in
        quickerSort(&array, 0, array.count-1)

        let end:clock_t = clock();
        println("quickerSort: \((end-start)*1000/clock_t(CLOCKS_PER_SEC)) ms")
        
        assert(isSorted(array), "quickerSort: Array not sorted");

    })
}

// MARK: Main

var arr:[Int] = []
for (var i = 0; i < 20; ++i) {
    arr.append(Int(rand())%100);
}


useQuickSort(arr)
useQuickerSort(arr)
I get the following result:

quickSort: 784 ms
quickerSort: 772 ms
You obviously tell that the same implementation in Swift is quite slower than in C++, but that’s not the point. We’re not experimenting C++ vs Swift, but single thread vs multiple threads. And using GCD, we see quite a small improvement, even though our test case is so small. This is most probably because we’re not manually handling the thread management ourselves anymore. We simply let the dispatch_queue handle the thread management.

The sunshine of todays experiment is that if we make good use of multi threading, we have a lot of potential for getting a large improvements. And, this is more than enough to motivate me to dive further down the concurrency rabbit hole.

The source code for this experiment is available at the GitHub repository: https://github.com/chunkyguy/ConcurrencyExperiments

Posted in concurrency, _dev	| Tagged C, Concurrency, Experiment, Swift	| 0 Comments
The Art of Designing Softwares
Posted on September 7, 2014 by chunkyguy
When I was in college, my vision of a software was like, we have a problem and we need a solution, a program, a set of instructions, that has to be executed by the computer in order to solve that particular problem. For example, design a program to print a Fibonacci series using loops, recursion, implement with an array, a doubly linked list, an AVL tree.

Then there are these programming challenges like the Facebook Hacker Cup, or the Google CodeJam, that just focus on how strong are you at understanding and implementing common algorithms and data structures, or can you design your own data structures and algorithms?

And, there is nothing wrong with that, but just that writing softwares isn’t just about designing algorithms or data structures, as one eventually discovers.

When I got my first job as an iOS developer, and start writing softwares at the enterprise levels. First thing I realized was that, the solution space is no longer just a set of instructions in a single file, but it is split into hundreds if not thousands of files. I realized that writing softwares is no longer just about algorithms and data structures. In fact, almost all of the interesting algorithms and data structures are either already solved and ready to use for your existing code, or nobody really cares about them as the first priority. There is a high probability that the programming languages of your choice is already bundled with all common data structures and algorithms you’ll ever need.

As a software engineer, most of the time I was working on managing the flow of data and control. The actual work seemed more close to plumbing, where instead of fluids one manages the flow of electrons through the hardware. Initially I was somewhat shocked to see that the iOS framework provides just three data structures, NSArray, NSDictionary and NSSet*. Thats it!

* (maybe also NSMapTable, NSHashTable, NSCountedSet, NSPointerArray,… but let’s assume we haven’t heard of them)

What is software engineering?

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

Bottom-up approach
Think of a problem where you have to design a car, in software that is. One of the ways you can design a car is, break down it into smaller tasks.

Design the tire.
Design the body.
Design the engine.
Here each step is recursive in itself, as the designing the engine would be again broken down further until you reach an atomic task that can not be broken down any further. You start working on those pieces and at then assemble them all back into places, and you have a car.

This solution has many benefits like, you can easily assign tasks between team members, and can almost work independently on them. Also, it is easier to design for the future this way, as when you’re working on designing the tires, you’re just focused on designing the tires that should fit all known use cases.

The main problem with bottom-up approach is, you need to be a good future-seeker. Whatever solutions you’re designing, should fall into place perfectly. Say on the final assembly day you realize the sockets on the body are not big enough to hold the tires. Also, as one can tell, bottom-up approach is more time consuming, because you’re designing for the unknown future. Imagine, your car has to perform well on all sort of geographical terrains. Most of the time you might even end up thinking too forward and design for the year 2056, while your solution would become obsolete in next 3 months. Try talking to all those who wrote great solutions with NSThread before Apple rolled GCD. And don’t forget the time constraints, your boss, teammates, consumers are always pressurizing you to release it already.

Top down approach
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

 

Posted in _dev, _general_things	| Tagged programming	| 0 Comments
Component System using Objective-C Message Forwarding
Posted on August 5, 2014 by chunkyguy
Ever since I started playing around with the Unity3D, I wanted to build one of my games with component system. And, since I use Objective-C most of the time, so why not use it.

First let me elaborate on my intention. I wanted a component system where I’ve a GameObject that can have one or more components plugged in. To illustrate the main problems with the traditional Actor based model lets assume we’re developing a platformer game. We have these three Actor types:

1. Ninja: A ninja is an actor that a user can control with external inputs.
2. Background: Background is something like a static image.
3. HiddenBonus: Hidden bonus is a invisible actor that a ninja can hit.

Source: Swim Ninja
Source: Swim Ninja
Let’s take a look at all the components inside these actors.

Ninja = RenderComponent + PhysicsComponent
Background = RenderComponent
HiddenBonus = PhysicsComponent.

Here RenderComponent is responsible for just drawing some content on the screen, while the PhysicsComponent is responsible for just doing physics updates like, collision detection.

So, in a nutshell from our game loop we need to call things like

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
Now, in the traditional Actor model, if we have an Actor class with both RenderComponent and PhysicsComponent like:

@interface Actor : NSObject

- (void)render;
- (void)update:(int)dt;

@end
Then it would inevitably add a PhysicsComponent to Background actor and a RenderComponent to a HiddenBonus actor.

In Objective-C we can in fact design the component system using the message forwarding.
( Don’t forget to checkout this older post on message forwarding ).

At the core, we can have a GameObject class like

@interface GameObject : NSObject

- (void)render;
- (void)update:(int)dt;

@end
But this doesn’t implements any of these methods, instead it forwards them to relevant Component classes. We can write our Component classes as

@interface PhysicsComponent : NSObject

- (void)update:(int)dt;

@end

@interface RenderComponent : NSObject

- (void)render;

@end
Our GameObject class gets reduced to

@interface GameObject : NSObject

+ (GameObject *)emptyObject;

- (void)enablePhysicsComponent;
- (void)enableRenderComponent;

@end
Our game loop initializes our GameObjects as

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
And updates and render them as

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
If you compile at this stage, you will get errors like:

‘No visible @interface for ‘GameObject’ declares the selector ‘update’.
‘No visible @interface for ‘GameObject’ declares the selector ‘render’.
In earlier days the code used to compile just like that. Later it became an warning, which was good for most of the developer who migrated to Objective-C from other compile time bound languages like C++. Objective-C is mainly a runtime bound language, but very few developers appreciate this fact and they complained a lot on why compilers don’t catch their errors early on, or more precisely, why it doesn’t works somewhat like C++. And I understand that feeling, because I too love C++. But, when I’m coding in Objective-C I use it like Objective-C. They are two completely different languages based on completely different ideology.

Moving forward in time, ARC came along. It made using Objective-C runtime behavior impossible. Now the ‘missing’ method declaration became an error. For this experimentation, let’s disable ARC entirely and see how it goes.

OK, good enough. The errors are now warnings. But we can at least move forward with our experiment.

Lets add the message forwarding machinery to our GameObject. Whenever a message is passed to an object in Objective-C, if the message is not implemented by the receiver object, then before throwing an exception the Objective-C runtime offers us an opportunity to forward the message to some other delegate object. The way this is done is quite simple, we just need to implement following methods.

First thing we need to handle is passing a selector availability test.

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
Next, we need to handle the method selector signature availability test.

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
After all the tests have been cleared, it time to implement the actual message forwarding.

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
If you run the code at this point, you should see update and render getting invoked for the corresponding component objects. So yay, our component system prototype is working!

Now let try to make it more concrete. Let’s work on the RenderComponent first. For testing in each render call, we just draw a Cube on the screen.

The best part about the Component System is that every component focuses on just one thing. For example, the RenderComponent just focuses on rendering. We just need to add methods that are required just for rendering.

Lets pass in the matrix information to the RenderComponent in form of a NSValue.

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
The conversion from GLKMatrix to NSValue can be done easily like:

    GLKMatrix3 nMat = GLKMatrix3InvertAndTranspose(GLKMatrix4GetMatrix3(modelViewMatrix), NULL);
    GLKMatrix4 mvpMat = GLKMatrix4Multiply(projectionMatrix, modelViewMatrix);
    [ninja setNormalMatrix:[NSValue value:&nMat withObjCType:@encode(GLKMatrix3)]];
    [ninja setModelViewProjectionMatrix: [NSValue value:&mvpMat withObjCType:@encode(GLKMatrix4)]];
Now using our message forwarding magic, in our main loop we just need to update the GameObjects with rendering enabled like so:

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


As you can already tell, how convenient Message Forwarding has made it for us to add our custom code in Components and just directly call them, even though the GameObject class doesn’t directly implements them.

I believe, you can extend this idea to add more Components and rapidly add and use the functionality, while keeping the code logically separated by Components.

The accompanied code is available at https://github.com/chunkyguy/OjbCComponentSystem. Since, this was just a rapid prototype test, and we have disable the ARC, the code could have potential memory leaks. Don’t use the code directly, use it just for reference purposes.

Posted in OpenGL, _dev, _games, _iOS	| Tagged Design Patterns, Message Forwarding, Objective C	| 0 Comments
C++: Typesafe programming
Posted on August 3, 2014 by chunkyguy
Lets say you have a function that needs angle in degrees as a parameter.

    void ApplyRotation(const float angle)
    {
        std::cout << "ApplyRotation: " << angle << std::endl;
    }
And an another function that returns a angle in radians

    float GetRotation()
    {
        return 0.45f;
    }
To fill out the missing piece, you write a radians to degree function

    float RadiansToDeg(const float angle)
    {
        return angle * 180.0f / M_PI;
    }
Then some place later, you use the functions like

    float angleRadians = GetRotation();
    float angleDegrees = RadiansToDeg(angleRadians);
    ApplyRotation(angleDegrees);
This is bad. The user of this code, who might be the person next to you, or yourself
10 weeks later, doesn’t knows what does it means by angle in functions ApplyRotation or GetRotation.
Is it angle in radians or angle in degrees?

Yes, you can add a comment on top of this function about where is it a angle in degrees and
where radians. But, that doesn’t actually stop the user from passing a value in whatever format
they will.

The main problem with this piece of code is that it uses a float as a parameter, which is
an implementation detail, and doesn’t passes any other information. In C++ we can do better.

Lets create new types.

    struct Degrees {
        explicit Degrees(float v) :
        value(v)
        {}
        
        float value;
    };
    
    struct Radians {
        explicit Radians(float v) :
        value(v)
        {}
        
        float value;
    };
And update the functions as

    Radians GetRotation()
    {
        return Radians(0.45f);
    }
    
    void ApplyRotation(const Degrees angle)
    {
        std::cout << "ApplyRotation: " << angle.value << std::endl;
    }
    
    Degrees RadiansToDeg(const Radians angle)
    {
        return Degrees(angle.value * 180.0f / M_PI);
    }
Now if we try to call it with following it just works.

    Radians angleRadians = GetRotation();
    Degrees angleDegrees = RadiansToDeg(angleRadians);
    ApplyRotation(angleDegrees);
Notice that, if we don’t specify the constructors as explicit the compiler will implicitly
converts the value from float to corresponding types. Which isn’t what we want.

This is already starting to look good. The code is self documenting, and the user will have
difficult times, if they try to use if differently than intended as most of the error checking
is done by the compiler.

Another benefit is that, now we can have a member function that does the conversion like

    struct Radians {
        explicit Radians(float v) :
        value(v)
        {}

        Degrees ToDegrees() const
        {
            return Degrees(value * 180.0f / M_PI);
        }

        float value;
    };
So, your calling code reduces to

    Radians angle = GetRotation();
    ApplyRotation(angle.ToDegrees());
As you can see, we no longer have variable names that tell the type information, but the
type that tells about itself.

On the final note we can make passing from value to passing as a const reference

    void ApplyRotation(const Degrees &angle)
    {
        std::cout << "ApplyRotation: " << angle.value << std::endl;
    }
This does two things. Firstly, no more copies when data gets passed around and secondly we
can pass a temporary value as guaranteed by the C++ standard, like so

    ApplyRotation(GetRotation().ToDegrees());
    ApplyRotation(Degrees(45.0f));
Posted in _dev	| Tagged C, tutorial	| 1 Comment
Getting started Cocos2d with Swift
Posted on June 24, 2014 by chunkyguy
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

Posted in _games, _iOS	| Tagged Cocos2d, Objective C, Swift, tutorial	| 0 Comments
Experiment 14: Object Picking
Posted on February 23, 2014 by chunkyguy
This article is part of the Experiments with OpenGL series

The source code is available at https://github.com/chunkyguy/EWOGL

Every game at some point requires the user to interact with the 3D world. Probably in the form of a mouse click event firing bullets at targets or like a touch drag on a mobile device to rotate the view on the screen. This is called object picking, because most probably we are picking objects in our 3D space using mouse or touch screen from the device space.

How I like to picture this problem is, that we have a point in the device coordinate system and we would like to ray trace it back to some relevant 3D coordinate system, like the world space or probably the model space.

In the fixed function pipeline days there was a gluUnProject function which helped in this ray tracing, and still most developers like to use it. Even the GLKit provides a similar GLKMathUnproject function. Basically, this function requires a modelview matrix, a projection matrix, a viewport or the device coordinates and a 3D vector in device coordinate and it returns back the vector transformed to the object space.

With modern OpenGL, we don’t need to use that function. First of all it requires us to provide all the matrices seperately, and secondly, it returns the vector in model space, but our needs could be different, maybe we need it in some other space. Or we don’t need the vector at all, rather, just a bool flag to indicate whether the object can be picked or not.

At the core, we just need two things: a way to transform our touch point from device space to the clip space and an inverse of model-view-projection matrix to transform it from the clip space to the object’s model space.

Lets assume we have a scene with two rotating cubes and we need to handle the touch events on the screen and test if the touch point a collision with any of the objects in the scene. If the collision does happens, we just change the color of that cube.


First problem is to convert the touch point from iPhone’s coordinates system to the OpenGL’s coordinate system. Which is easy as it just means we need to flip the y-coordinate


  /* calculate window size */
  GLKVector2 winSize = GLKVector2Make(CGRectGetWidth(self.view.bounds), CGRectGetHeight(self.view.bounds));

  /* touch point in window space */
  GLKVector2 point = GLKVector2Make(touch.x, winSize.y-touch.y);
Next, we need to transform the point to a normalized device coordinates space, or to range of [-1, 1]


  /* touch point in viewport space */
  GLKVector2 pointNDC = GLKVector2SubtractScalar(GLKVector2MultiplyScalar(GLKVector2Divide(point, winSize), 2.0f), 1.0f)
Next question that we need to tackle is, how to calculate a 3D vector from a device space, as the device is a 2D space. We need to remember that after the projection transformations are applied the depth of the scene is reduces to the range of [-1, 1].
So, we can use this fact and calculate the 3D positions at both the depth locations.


  /* touch point in 3D for both near and far planes */
  GLKVector4 win[2];
  win[0] = GLKVector4Make(pointNDC.x, pointNDC.y, -1.0f, 1.0f);
  win[1] = GLKVector4Make(pointNDC.x, pointNDC.y, 1.0f, 1.0f);
Then, we need to calculate the inverse of the model-view-projection matrix


  GLKMatrix4 invMVP = GLKMatrix4Invert(mvp, &success);
Now, we have all the things required to trace our ray


  /* ray at near and far plane in the object space */
  GLKVector4 ray[2];
  ray[0] = GLKMatrix4MultiplyVector4(invMVP, win[0]);
  ray[1] = GLKMatrix4MultiplyVector4(invMVP, win[1]);
Remember the values could be in the homogenous coordinates system, to convert them back to human-imaginable cartesian coordinate system we need to divide by the w component.


  /* covert rays from homogenous coordsys to cartesian coordsys */
  ray[0] = GLKVector4DivideScalar(ray[0], ray[0].w);
  ray[1] = GLKVector4DivideScalar(ray[1], ray[1].w);
We don’t need the start and end of the ray, but we need the ray in form of


 R = o + dt
Where o is the ray origin and d is the ray direction and t is the variable.

We calculate the ray direction as


  /* direction of the ray */
  GLKVector4 rayDir = GLKVector4Normalize(GLKVector4Subtract(ray[1], ray[0]));
Why is it normalized? We will take look at that later.

Now we have the ray in the object space. We can simply to a sphere intersection test. We must already know the radius of the bounding sphere, if not we can easily calculate it. Like for a cube, I’m assuming the radius to be equal to the half edge of any side.

For detailed information regarding a ray-sphere intersection test I recommend this article, but I’ll go through the minimum that we need to accomplish our goal.

Let points on surface of sphere of radius r is given by


x^2 + y^2 + z^2 = r^2
P^2 - r^2 = 0; where P = {x, y, z}
All points on sphere where ray hits should obey


(o + dt)^2 - r^2 = 0
o^2 + (dt)^2 + 2odt - r^2 = 0
f(t) = (d^2)t^2 + (2od)t + (o^2 - r^2) = 0
This is a quadratic equation in form


 ax^2 + bx + c = 0
And, it makes sense, because every ray can hit the sphere a maximum two points, first when it goes in the sphere and second when it leaves the sphere.

The determinant of the quadratic equation is calculated as


det = b^2 - 4ac
If determinant is < 0 it means no roots or the ray misses the sphere
If determinant is equal 0 means one root or the ray started from the inside of the sphere or it just touches it.
If determinant is > 0 means we have two roots, or the perfect case of ray passing through the sphere.

In our equation the values of a, b, c are


a = d^2 = 1; as d is a normalized vector and dot(d, d) = 1
b = 2od
c = o^2 - r^2
So, if we have ray with direction normalized we can simply test if it hits our sphere in model space with


- (BOOL)hitTestSphere:(float)radius
        withRayOrigin:(GLKVector3)rayOrigin
         rayDirection:(GLKVector3)rayDir
{
  float b = GLKVector3DotProduct(rayOrigin, rayDir) * 2.0f;
  float c = GLKVector3DotProduct(rayOrigin, rayOrigin) - radius*radius;
  printf("b^2-4ac = %f x %f = %f\n",b*b, 4.0f*c, b*b - 4*c);
  return b*b >= 4.0f*c;
}
Hope this helps you in understanding the object picking concept. Don’t forget to checkout the full code from the repository linked at the top of this article.

Posted in OpenGL, _algo, _dev	| Tagged EWOGL, ObjC, Object Picking	| 3 Comments
Experiment 13: Shadow mapping
Posted on February 2, 2014 by chunkyguy
This article is part of the Experiments with OpenGL series

The source code is available at https://github.com/chunkyguy/EWOGL

Shadows add a great detail to the 3D scene. For instance, it’s really hard to judge how high the object really is above the floor without a shadow, or how far away is the light source from the object.


Shadow mapping is one of the trick to create shadows for our geometry.

The basic idea it to render the scene twice. First from the light’s point of view, and save the result in a texture. Next, render from the desired point of view and compare the result with the saved result. If the part of geometry being rendered falls under shadow region, don’t apply lighting to it.

During setup routine, we need 2 framebuffers. One for each rendering pass. The first framebuffer needs to have a depth renderbuffer, and a texture attached to it. The second framebuffer is your usual framebuffer with color and depth.

The first pass is very simple. Switch the point of view to the light’s view and just draw the scene as quick as possible to the framebuffer. You just need to pass the vertex positions and the model-view-projection matrix to take those positions from the object space to the clip space. The result is what some like to call as the Shadow map.

The shadow map is just the 1-channel texture where blacker the color means, more close it is to the light. While whiter the color means the farther it is from light. And, a totally white color means, that part is not visible to the light.

With black and white I’m assuming the depth range is from 0.0 to 1.0, which is the default range I guess.

The second pass is also not very difficult. We just need a matrix that will take our vertex positions from object space to the clip space from the light’s view. Let’s call it the shadow matrix.

Once, we have the shadow matrix, we just need to compare the compare between the z of stored depth map and the z of our vertex’s position.

Creating of shadow matrix is also not very difficult. I calculate it like this:


    GLKMatrix4 basisMat = GLKMatrix4Make(0.5f, 0.0f, 0.0f, 0.0f,
                                         0.0f, 0.5f, 0.0f, 0.0f,
                                         0.0f, 0.0f, 0.5f, 0.0f,
                                         0.5f, 0.5f, 0.5f, 1.0f);
    GLKMatrix4 shadowMat = GLKMatrix4Multiply(basisMat,                  
                            GLKMatrix4Multiply(renderer->shadowProjection,  
                            GLKMatrix4Multiply(renderer->shadowView ,mMat)));
The basis matrix is just another way to move the range from [-1, 1] to [0, 1]. Or from depth coordinate system to texture coordinate system.

This is how it can be calculated:


GLKMatrix4 genShadowBasisMatrix()
{
  GLKMatrix4 m = GLKMatrix4Identity;
  
  /* Step 2: scale by 1/2 
   * [0, 2] -> [0, 1]
   */
  m = GLKMatrix4Scale(m, 0.5f, 0.5f, 0.5f);

  /* Step 1: Translate +1 
   * [-1, 1] -> [0, 2]
   */
  m = GLKMatrix4Translate(m, 1.0f, 1.0f, 1.0f);

  return m;
}
The first run was pretty cool on the simulator, but not so good on the device.

That is something I learned from my last experiment. To try out on the actual device as soon as possible.

I tried playing around with many configurations, like using 32 bits for depth precision, using bilinear filtering with depth map texture, changing the depth range from 1.0 to 0.0 and many other I don’t even remember. I should say, changing the depth range from the default [0, 1] to [1, 0] was the best.

If you’re working on something, where the shadows needs to be projected far away from the object. Like on a wall behind, try this in the first pass, while creating the depth map:


glDepthRangef(1.0f, 0.0f);
And, in the second pass don’t forget to switch back to


glDepthRangef(0.0f, 1.0f);
Of course, you also need to update your shader code, so that the result 1.0 now means fail case while 0.0 means the pass case. Or you could even update the GL_TEXTURE_COMPARE_FUNC to GL_GREATER.

But, for majority cases where the shadow is attached to the geometry. The experiment’s code should work fine.


Another bit of improvement, that can be added is using the PCF (Percentage Closer Filtering) method. It helps with the anti-aliasing at the shadow edges.

The way PCF works is, we read the result from surrounding textures and average them.
Here’s an example of the improvement you can expect.

Hosted by imgur.com

Another shadow mapping method, that I would like to try in the future is with random sampling. Where instead of fixed offsets, we pick random samples around the fragment. I’ve heard this creates the soft shadow effect.

Shadow mapping, is a field of many hit and trials. There doesn’t seems to be one solution fit all. There are so many parameters that can be configured in so many ways. Like, you can try with culling front face, or setting polygon offset to avoid z-fighting. In the end, you just need to find the solution that fits your current needs.

The code right now only works for OpenGL ES3, as I was using many things available only to GLSL 3.00. But, since the trick is very trivial, I would in the future upgrade to code to work with OpenGL ES 2 as well. I see no reasons why it shouldn’t work.