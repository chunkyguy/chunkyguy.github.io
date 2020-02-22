---
layout: post
title:  "C++: Typesafe programming"
date:   2014-08-03 23:28:54 +0530
categories: cpp
---

Lets say you have a function that needs angle in degrees as a parameter.

```cpp
void ApplyRotation(const float angle)
{
    std::cout << "ApplyRotation: " << angle << std::endl;
}
```
And an another function that returns a angle in radians
```cpp
float GetRotation()
{
    return 0.45f;
}
```

To fill out the missing piece, you write a radians to degree function

```cpp
float RadiansToDeg(const float angle)
{
    return angle * 180.0f / M_PI;
}
```

Then some place later, you use the functions like
```cpp
float angleRadians = GetRotation();
float angleDegrees = RadiansToDeg(angleRadians);
ApplyRotation(angleDegrees);
```

This is bad. The user of this code, who might be the person next to you, or yourself 10 weeks later, doesn’t knows what does it means by angle in functions `ApplyRotation` or `GetRotation`. Is it angle in radians or angle in degrees?

Yes, you can add a comment on top of this function about where is it a angle in degrees and where radians. But, that doesn’t actually stop the user from passing a value in whatever format they will.

The main problem with this piece of code is that it uses a float as a parameter, which is an implementation detail, and doesn’t passes any other information. In C++ we can do better.

Lets create new types.
```cpp
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
```

And update the functions as
```cpp
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
```

Now if we try to call it with following it just works.

```cpp
Radians angleRadians = GetRotation();
Degrees angleDegrees = RadiansToDeg(angleRadians);
ApplyRotation(angleDegrees);
```

Notice that, if we don’t specify the constructors as explicit the compiler will implicitly converts the value from float to corresponding types. Which isn’t what we want.

This is already starting to look good. The code is self documenting, and the user will have difficult times, if they try to use if differently than intended as most of the error checking is done by the compiler.

Another benefit is that, now we can have a member function that does the conversion like
```cpp
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
```

So, your calling code reduces to
```cpp
Radians angle = GetRotation();
ApplyRotation(angle.ToDegrees());
```

As you can see, we no longer have variable names that tell the type information, but the type that tells about itself.

On the final note we can make passing from value to passing as a const reference
```cpp
void ApplyRotation(const Degrees &angle)
{
    std::cout << "ApplyRotation: " << angle.value << std::endl;
}
```

This does two things. Firstly, no more copies when data gets passed around and secondly we can pass a temporary value as guaranteed by the C++ standard, like so

```cpp
ApplyRotation(GetRotation().ToDegrees());
ApplyRotation(Degrees(45.0f));
```
