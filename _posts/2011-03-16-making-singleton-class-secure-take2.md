---
layout: post
title: "Making singleton class secure: Take 2"
date: 2011-03-16 00:00:00 +0530
categories: ios
published: true
---

Few days back I wrote something about the Making singleton class secure.
Its my personal experience, avoid Singleton approach as far as possible in the sense you avoid Pointers and class destructors and switch to Java from c++.
But, while I say this every now and then I feel that uncontrollable surge to put all the code in a singleton class and put up all sort of factories in there.

So, here I’ll discuss one of the common pitfalls you might fell into, if not cautious.

Ok you wrote a ReminderManager class as singleton.

```
+(ReminderManager *)defaultManager
{
	if(!_def_mgr)
		_def_mgr = [[self alloc] __init__];
	return _def_mgr;
}

/*Hold me committing the 8th deadly sin*/
-(id)init
{
	NSException *ex = [NSException exceptionWithName:@"ReminderManager" reason:@"Creating new instance of singleton class" userInfo:nil];
	@throw ex;
	return nil;
}

/*private init*/
-(id)__init__
{
	if(self = [super init])
	{
              _reminders = [[NSMutableArray alloc] init];
              for(int i = 0; i < DEF_REMINDERS; i++)
	      {
	          Reminder *rem = [[Reminder alloc] initWithName:@"<Blank>"];
	          [_reminders addObject:rem];
		  [rem release];
	     }
	}
	return self;
}
```

And, initialized few blank Reminders.
Since this manages all the Reminders, so lets say you added a method to tell what the next available ID is:

```
/*generate ID's for Reminders*/

-(NSInteger)nextID
{
	return [_reminders count];
}
```

Now, in the Reminder class you did something like:

```
-(id)initWithName:(NSString *)n
{
	if(self = [super init])
	{
		_identifier = [[ReminderManager defaultManager]nextID];
		self.name = n;
	}
	return self;
}
```

And happily when you compile and run you will soon see it running into some infinite loop. Why?
It turns out that while the ReminderManger is busy creating Reminders via:

```
		Reminder *rem = [[Reminder alloc] initWithName:@"<Blank>"];
```

The Reminder is in return calling the ReminderManger to get the _identifier, via;

```
		_identifier = [[ReminderManager defaultManager]nextID];
```

But, as you might have noticed that by this time the ReminderManager isn’t ready.
So, the [ReminderManager defaultManager] launches the __init__ and which starts creating more Reminders!
And hence the recursion.

The solution in fact turns out to be very simple. You need to architecture the ReminderManager as:

```
-(id)__init__
{
	if(self = [super init])
	{
		_reminders = [[NSMutableArray alloc] init];
	}
	return self;
}
```

And add a separate method to create Reminder‘s:

```
-(void)init_rems
{
	for(int i = 0; i < DEF_REMINDERS; i++)
	{
		Reminder *rem = [[Reminder alloc] initWithName:@"<Blank>"];
		[_reminders addObject:rem];
		[rem release];
	}
}
```

But, DON’T call this method from anywhere in the `[ReminderManager defaultManager]`. Keep it away from its reach!
Then, at some better location, probably at the App Launching method, do something like;

```
[[ReminderManager defaultManager]init_rems];
```

Problem Solved :)

