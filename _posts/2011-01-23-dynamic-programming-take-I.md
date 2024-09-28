---
layout: post
title: "Dynamic Programming: Take I"
date: 2011-01-23 00:00:00 +0530
categories: ios
published: true
---

> I mean, if 10 years from now, when you are doing something quick and dirty, you suddenly visualize that I am looking over your shoulders and say to yourself “Dijkstra would not have liked this”, well, that would be enough immortality for me.
- Edsger Wybe Dijkstra

Its all happens like: Imagine you live in a small sleepy town, roads are limited. Every weekend you go to the only restaurant in town, and seats are always available there. So, you’re very happily living, like the Shrek in his swamp.
Then, one day the big rock hits you: You have to move to the big city. You see unlimited roads, unlimited destination. Now, every weekend you spend hours, deciding upon restaurants, roads to take, calling up pals. Decisions every where!
Welcome to the dynamic world.
Why do we need to think dynamic? The first reason I can think is TIME. Of course, everyone loves a second saved. After all that is what evolution is about. We evolve, we become faster than ever.

Ok, enough shit. Lets code.

Here’s a factorial program thats the first program for everybody learning programming. This program prints all the factorials up to n:

```
#define TAB 4
int fact(int n)
{
	printf("%*.sfact(%d) = %d * fact(%d)\n",TAB*n," ",n,n,n-1);
	int f = 1;
	if(n != 1)
		f = n *fact(n-1);
	 printf("%*.sfact(%d) = %d\n",TAB*n," ",n,f);
	return f;
}

int main(int argc, char **argv)
{
	int n;
	for(scanf(" %d",&n);n; n--, printf("\n\n\n"))
		fact(n);
}
```
Here’s the output, for n = 4:

```
                fact(4) = 4 * fact(3)
            fact(3) = 3 * fact(2)
        fact(2) = 2 * fact(1)
    fact(1) = 1 * fact(0)
    fact(1) = 1
        fact(2) = 2
            fact(3) = 6
                fact(4) = 24

            fact(3) = 3 * fact(2)
        fact(2) = 2 * fact(1)
    fact(1) = 1 * fact(0)
    fact(1) = 1
        fact(2) = 2
            fact(3) = 6

        fact(2) = 2 * fact(1)
    fact(1) = 1 * fact(0)
    fact(1) = 1
        fact(2) = 2

    fact(1) = 1 * fact(0)
    fact(1) = 1
```

So, as you can see, with this approach, we are calculating the same thing many times. For example, in this example we have calculated the factorials for various numbers as:
```
fact(4): 1
fact(3): 2
fact(2): 3
fact(1): 4
```

Now, just imagine for n = 1000000, the calculations will be something like;
```
fact(1000000): 1
fact(999999): 2
fact(999998): 3
…
fact(3): 999998
fact(2): 999999
fact(1): 1000000
```

So, here’s where the DP comes into play. The theory from wikipedia says: dynamic programming is a method for solving complex problems by breaking them down into simpler subproblems...

Lets make out silly factorial program dynamic !!
Here’s one ways to do it: We will store the values already calculated in a array and from next time onwards, we’ll check if the factorial we need has been already calculated, we will use that value.

```
#include <stdio.h>
#define TAB 4
#define SOME_BIG_NO 100

int storage[SOME_BIG_NO] = {0};

void store(n,f)
{
	if(!storage[n])
		storage[n] = f;
}

int get_f(int n)
{
	return storage[n]?:0;
}

int fact(int n)
{
	int f = get_f(n);
	if(!f)
	{
		printf("%*.sfact(%d) = %d * fact(%d)\n",TAB*n," ",n,n,n-1);
		f = 1;
		if(n > 1)
			f = n *fact(n-1);
		store(n,f);
	}
	 printf("%*.sfact(%d) = %d\n",TAB*n," ",n,f);
	return f;
}

int main(int argc, char **argv)
{
	int n;
	for(scanf(" %d",&n);n; n--, printf("\n\n\n"))
		fact(n);
}
```

And magically the output reduces to ( again for n = 4);

```
                fact(4) = 4 * fact(3)
            fact(3) = 3 * fact(2)
        fact(2) = 2 * fact(1)
    fact(1) = 1 * fact(0)
    fact(1) = 1
        fact(2) = 2
            fact(3) = 6
                fact(4) = 24

            fact(3) = 6

        fact(2) = 2

    fact(1) = 1
```

Conclusion: This program was pretty lousy, like using a array sized SOME_BIG_NO, is never a efficient way of doing things, but, its a good way start think dynamic way, as to say the least. :)
