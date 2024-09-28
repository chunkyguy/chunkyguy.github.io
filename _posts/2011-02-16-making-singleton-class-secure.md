---
layout: post
title:  "Making singleton class secure"
date: 2011-02-16 00:00:00 +0530
categories: ios
published: true
---
There comes a time for a project, when you can’t resist using a Singleton class, for things that can’t be afforded to duplicate.
Say, for example I’m creating a World class and I want only a single world to exist in my entire project. So, next time anyone is referring to World, he’s referring to this:

```
static World *_world = nil;

@implementation World

+(World *)sharedWorld
{
	if(!_world)
		_world = [[self alloc] init];
	return _world;
}

-(id)init
{
	self = [super init];
	if(self)
	{
		//create world
		[self setup_world];
	}
	return self;
}
```

And then whenever someone wants to refer to world:

```
[[World sharedWorld] scheduleSnowfall];
```

Look cool? wait a sec. Suppose, at some random time somebody does something like:

```
World *newOne = [[World alloc] init];
[newOne scheduleSnowfall];
[newOne release];
```

This will create a new instance of the World and scheduleSnowfall in there. (Everybody living in this world will still be having sunshine).

So, the best fix is throwing an exception, thats like creating a new World totally illegal. How you do that?

```
+(World *)sharedWorld
{
	if(!_world)
		_world = [[self alloc] __secret_init];
	return _world;
}

-(id)init
{
	NSException *warning = [NSException exceptionWithName:@"World" reason:@"Illegally created: are you God?" userInfo:nil];
	@throw warning;
}

-(id)__secret_init
{
	self = [super init];
	if(self)
	{
		//create world
		[self setup_world];
	}
	return self;
}
```

So, now for anybody trying to be the next GOD, will see this exception ruining the dreams:

 *** Terminating app due to uncaught exception 'World', reason: 'Illegally created: are you God?'
