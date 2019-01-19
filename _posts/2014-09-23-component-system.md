---
layout: post
title:  "Component System using C++ Multiple Inheritance"
date:   2014-09-23 23:28:54 +0530
categories: concurrency
---

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
``` cpp
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
```

Now, in the traditional Actor model, if we have an Actor class with both RenderComponent and PhysicsComponent like:
``` cpp	
class Actor {
public:
    void Update(const int dt);
    void Render();
};

    Actor ninja;
    Actor hiddenBonus;
    Actor background;
```

Then it would inevitably add a PhysicsComponent to background actor and a RenderComponent to a hiddenBonus actor. So, the user of the code has to keep a mental track of what functions shouldn’t be invoked on what objects. Which is as awful in reality, as it sounds in theory.

We could use the same approach we used when designing the Component System with Objective-C, by having a function on fat GameObject that could be used to enable a particular component. Think of each GameObject as an collection of many components. Where each component know how to deal with one thing, like rendering, physics, AI, networking, and so on. When creating a new GameObject we need a method to specify what components should be enabled for it. The fat interface technique we’ve already discussed in that last experiment. Let’s try something new, we’re using C++ after all. Let’s start with breaking the components as separate classes:
``` cpp
class RenderComponent {
public:
    void Render();
};

class PhysicsComponent {
public:
    void Update(const int dt);
};
```

Then we can easily create composite components using such core components, such as:
``` cpp
class RenderAndPhysicsComponent : public RenderComponent, public PhysicsComponent
{};
```
Now, we can update our game scene’s interface as:
``` cpp
    RenderAndPhysicsComponent ninja;
    PhysicsComponent hiddenBonus;
    RenderComponent background;
```

This definitely serves our immediate needs, but there is some problem. This solution doesn’t scales well. Whenever we come up with a new component, we’ve to update your related base class accordingly and create a mix for each possible basic component interfaces. For five core components, we would get a total of C(5,1) + C(5,2) + C(5,3) + C(5,4) + c(5,5) = 5 + 10 + 10 + 5 + 1 = 31 total component interfaces combinations possible.

In C++, we can actually do better. This is because C++ has one thing that other languages just don’t want. It’s called multiple inheritance.

Instead of making the a composite collection class a direct collection of many components, we can instead have a single GameObject class that could inherit from as many components as desired. And, further to avoid having the fat abstraction issue, we can have the GameObject to be inherited at compile time from only the components that we actually need for that particular object.

How do we design such a GameObject? One correct answer is using variadic templates:
``` cpp
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
```

This GameObject has nothing of its own. A host class that wishes to use a instance of this object has to supply a concrete class as a template parameter. This has given infinite flexibility to our GameObject.

We can now update our interface to easily use the core components to construct our GameObject instances that fit our needs:
``` cpp
    GameObject<RenderComponent, PhysicsComponent> ninja;
    GameObject<PhysicsComponent> hiddenBonus;
    GameObject<RenderComponent> background;
```

If that seems to verbose to write every time, we can in fact use typedefs to create some most common types.
``` cpp
    typedef GameObject<RenderComponent> Sprite;
    typedef GameObject<PhysicsComponent> Trigger;
    typedef GameObject<RenderComponent, PhysicsComponent> Actor;
    
    Actor ninja;
    Trigger hiddenBonus;
    Sprite background;
```

Now, I know that feels already too awesome, but we still have a problem. In practice we often need to communicate between components. Like say, we have a TransformComponent, that just provides us a model matrix based on the current transform of the GameObject.
``` cpp
class TransformComponent {
public:
    /** Get the model matrix of the transform to take things from model space to
     * the world space
     */
    GLKMatrix4 GetModelMatrix() const;
    
    GLKVector2 position;
    float rotation;
};
```

And, that model matrix then could be used by the RenderComponent for rendering the mesh.
``` cpp
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
```
So, how do go with that design?

To solve that, we actually need another component. Let’s call it CustomComponent. The entire purpose of CustomComponent is to have an Update function:
``` cpp
class CustomComponent {
public:
    void Update(const int dt);
    
protected:
    std::function<void(const int dt)> customUpdate;
};
```
Now we can use our CustomComponent to bind many other components with something like:
``` cpp
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
```
No matter how absurd this code looks, it works. We can improve this, and while doing that we shall also address another important issue with multiple inheritance, name clashing.

Since, our GameObject is now split into multiple files we can easily get name clashing. For example both RenderComponent and TransformComponent can have a member named
``` cpp
GLKMatrix4 modelMatrix;
```
Or, both PhysicsComponent and CustomComponent can have a member function named:

void Update(const int dt);
How do we resolve this issue? One way is to adopt a naming convention, like say every PhysicsComponent will have a ‘phy’ prefix appended before every member function and data member. That would reduce our code to:
``` cpp
void Ninja::Update(const int dt)
{
	/* from transform component */
	GLKMatrix4 modelMatrix = transGetModelMatrix();
	/* to render component */
    rendrSetModelMatrix(modelMatrix);
}
```

We all agree that following a naming convention is very hard. So this solution might not work in long term. A better way out is to encapsulate all the core logic out of the component to another core class. The TransformComponent could be reduced to:
``` cpp
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
```

With such pattern implemented, our same code would look like:
``` cpp
void Ninja::Update(const int dt)
{
    renderer_.SetModelMatrix(transform_.GetModelMatrix());
}
```

OK, with that taken care of, let’s now focus on the fitting the pieces together. Let’s begin with updating the commonly used GameObject configurations to proper C++ types:
``` cpp
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
```

Lets now implement the loop. The trickiest part of the loop is that we need to call Update() on every object that has a CustomComponent and Render() on objects that implement RenderComponent. Notice, so far we haven’t implemented any virtual functions, so can not depend on the runtime to select the correct function. What we actually need is something like:
``` cpp
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
```

What we actually need some sort of container class that hold all types of Components. The first thought that hits the brain is to have a std::vector of Component of every kind. And from our experience we already know this sort of patterns doesn’t scales well. But using the power of C++11 and tuples we implement that in a much better way. Let’s add a ContainerComponent as:
``` cpp
template <typename... Component>
class ContainerComponent {
protected:
    std::tuple<std::vector<Component*>...> children_;
};
```

As you can see, ContainerComponent is capable of holding any number of components of any size. Think of it as a class that can possess compile time flexibility to hold any kind of components and at runtime we can add and remove instances of those component classes.

Now we can update our plain Scene class to be a GameObject that has a ContainerComponent. The ContainerComponent is capable of holding two kind of Components, the CustomComponent and the RenderComponent:
``` cpp
class Scene : public GameObject <
	ContainerComponent<
		CustomComponent,
		RenderComponent
>> {
	/* ... */
};
```

The only important thing to remember is that the order matters. Because this is a tuple we have to remember the index of Component type.

Our new updated Load() now looks like:
``` cpp
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
```

The power of C++ compile time checks comes into play when we do something illegal, such as:
``` cpp
std::get<1>(children_).push_back(&hiddenBonus_);
```

The above expression will throw an error at compile time because the hiddenBonus_ does not has a RenderComponent and we’re trying to add it to a vector of RenderComponents.

So far so good. Let’s make the code more readable by removing the hard-coded integers. If we introduce a enum type as:
``` cpp
enum ContainerIndex {
    Custom = 0,
    Render = 1
};
```

We can do things like:
```  cpp
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
```

Now getting back to our loop. We can simply do things like:
``` cpp
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
```

Here is the result of this prototype:

Hosted by imgur.com

The yellow color is applied to the render-only background, while the cyan is the ‘render+physics’ ninja.

I haven’t implemented any physics elements, but in a real game with a Physics engine implemented the PhysicsComponent should look something like:
``` cpp
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
```

And any GameObject that listens to physics updates should look something like:
``` cpp
void Ball::Update(const int dt)
{
    transform_.SetPosition(PhysicsConverter::GetPixels2(physicsBody_->GetPosition()));
    renderer_.SetModelMatrix(transform_.GetModelMatrix());
}
```

And then the typical Scene class’s Update() should look like:

```  cpp
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
```

I hope this gives and idea of how can we use the enormous power of C++ to build an game engine based on component system. This also shows why C++ is such a great language for such a pattern when compared to something like Objective-C.

The entire code for this experiment is available at https://github.com/chunkyguy/CppComponentSystem. Please feel free to checkout and start playing around.
