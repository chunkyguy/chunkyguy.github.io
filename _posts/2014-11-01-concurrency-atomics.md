---
layout: post
title:  "Concurrency: Atomics"
date:   2014-11-01 11:28:54 +0530
categories: concurrency
---

I think we have covered most of the core concurrency concepts. With the
current knowledge we are good enough to tackle all real world
concurrency related problems. But this doesn’t means that we’ve covered
everything the thread libraries have to offer. And by thread libraries I
mean just the C++ standard thread library and libdispatch. For example,
we’re yet to see `std::future`, `std::promise`, `std::packaged_task`,
`dispatch_barier`, `dispatch_source` in action.

Today let’s focus on the low level of the libraries, atomics. Atomics
are the lowest level that we can work with a thread library. Atomic
operations provide the lowest level guarantee of ordering of operations.
An operation is atomic if there is a guarantee that the operation would
be never be left by any thread in an indeterminate state.

To make sure that the operation is atomic, internally the runtime could
be either not switch the thread while an atomic operation is underway or
maybe it simply uses a lock or whatever innovation the technology has to
offer. As a user of the library, you just get the guarantee that atomic
operation are indivisible.

To facilitate atomicity the C++ standard library offers a few atomic
types. All the types offer a `is_lock_free()` function to test if the
operation is done really not using any locks. Only exception is
`std::atomic_flag` which is the always lock free.

### Atomic Types

C++ provides a lot of atomic types. You can say that for almost all the
fundamental types have an atomic equivalent. (And by fundamental types I
mean bool and integers, no floating points, as you’ll see in a moment
why). You can use them directly or as `std::atomic<>` template
specialization. Here’s an example

``` cpp
std::atomic_bool b1;
std::atomic<bool> b2;
```

This same pattern is applied to all other fundamental types. Although,
you can use them as your usual fundamental types, there are few
operations that are not allowed. First striking constraint disallowed is
copy and assign operation. This code won’t compile

``` cpp
void TryCopy()
{
    std::atomic<bool> b1(false);
    std::atomic<bool> b2 = b1;
}
```

Then how do we use these atomic types? This brings us to `load()` and
`store()` operations. All atomic types except `atomic_flag` offer `load()`,
`store()`, `exchange()`, `compare_exchange_weak()` and
`compare_exchange_strong()` operations. Let’s take a look at what do
they do.

### load and store

If you’ve ever done a bit of assembly, you must be familiar with load
and store operations. Load retrieves the data while store saves the
data.

``` cpp
void DoLoad()
{
    std::atomic<int> i(10);
    std::atomic<int> j(i.load());
    std::cout << j << std::endl; // prints 10
}

void DoStore(const int n)
{
    std::atomic<int> i(0);
    i.store(n);
    std::cout << i << std::endl; // prints whatever n is
}
```

### exchange

Apart from the basic load and store, you also get a bunch of exchange
operations. An exchange operation does exactly what you’d expect, store
a new value and return the old value.

``` cpp
void DoExchange(const int n)
{
    std::atomic<int> i(50);
    int j = i.exchange(n);
    std::cout << i << " " << j << std::endl; // j = i; i = n;
}
```

An exchange operation is basically a 3 step operation. First it loads
the data, second it updates the data with new data, and third it stores
the new data back. And this should explain why floating points are left
out from fundamental types. [Floating point types are not deterministic
at
comparison](https://randomascii.wordpress.com/category/floating-point/page/2/).

And since we’re dealing with the lowest level of concurrency operations.
Sometimes you want a greater control over the execution. Maybe you need
a stronger guarantee that the 3 step exchange was indeed done
successfully before the running thread’s just ran out of time.

For such grained control the C++ standard library offers two more
exchange operations. `compare_exchange_weak()` and
`compare_exchange_strong()`.

The `compare_exchange_weak()` returns `false`, if the exchange wasn’t
successful. This could be because if the running thread’s time just ran
out and was kicked out by the scheduler before it could finish the
steps. This is called as *spurious failure*.

``` cpp
void DoExchangeWeak(const int desired)
{
    int expected = 50;
    std::atomic<int> i(expected);
    bool success = i.compare_exchange_weak(expected, desired);
    std::cout << std::boolalpha << success << " " << desired << " "
                << i << " " << expected << std::endl; // true 100 100 50
}
```

So if you want the exchange to run successfully every time, you probably
need to put this operation under a loop. So that whenever the operation
fails, you keep trying until it succeeds.

``` cpp
while (i.compare_exchange_weak(expected, desired) != true) {
}
```

Or, you can simply use `compare_exchange_strong()` which is guaranteed
to eliminate all spurious failures.

``` cpp
bool success = i.compare_exchange_strong(expected, desired);
```

Both of these functions return false whenever the expected value is not
same as the stored value. That is, whenever the comparison fails, and in
that case the expected updates to whatever was the actual value. For
example:

``` cpp
void DoExchangeWeak(const int desired)
{
    int expected = 0;
    std::atomic<int> i(5);
    bool success = i.compare_exchange_weak(expected, desired);
    std::cout << std::boolalpha << success << " "
    << desired << " " << i << " " << expected << std::endl;
    // false 1 5 5
}
```

For all fundamental types that support `+=`, `-+`, `|=`, `&=`, and `^=`, the
atomic types have equivalent `fetch_add`, `fetch_sub`, `fetch_or`,
`fetch_and`, `fetch_xor` operation available.

``` cpp
void DoFetchAdd(const int n)
{
    std::atomic<int> i(100);
    int j = i.fetch_add(n);
    std::cout << i << " " << j << std::endl;
    //for n = 50; output: 150, 100 => j = i; i += n;
}
```

Finally, if you want your custom type to work as an atomic type, you can
do that guaranteed that your custom type don’t do anything fancy. What
that means in practical world is that your type should work with
`memcpy()` and `memcmp()`. That is plain C types, no virtual table lookups.
Here’s trivial example:

``` cpp
struct MyType {
    int a;
    int b;
};

std::ostream &operator<<(std::ostream &os, const MyType &t)
{
    os << "{" << t.a << ", " << t.b << "}";
    return os;
}

void DoCustomExchange()
{
    std::atomic<MyType> a;
    a.store({10, 20});

    MyType b = {11, 21};
    MyType c = a.exchange(b);

    std::cout << "a: " << a << std::endl; // a: {11, 21}
    std::cout << "b: " << b << std::endl; // b: {11, 21}
    std::cout << "c: " << c << std::endl; // c: {10, 20}
}
```

There’s big part dealing with memory ordering that we’ve simply skipped
for now, but we shall come back to it later. Let’s now focus on the most
important atomic type `atomic_flag`.

### atomic_flag

Forget whatever that has been said about atomic types so far. None of
that applies to `std::atomic_flag`. `std::atomic_flag` is different. You
can say that `std::atomic_flag` is the core of the threading library. Its
like the atom of the universe. Let’s start exploring `std::atomic_flag`.

Let’s consider scenario. On your social network you get a lot of LOL
text that you just can’t understand. So, you decide to write a program
to convert that text either into a full uppercase or full lower case.

``` cpp
class UnLOLText {

public:
    UnLOLText(const std::string &name) :
    username_(name),
    modify_(0)
    {
        srand((unsigned int)time(0));
    }

    void ToUpper()
    {
        while (modify_ < username_.size()) {
            username_[modify_] = toupper(username_[modify_]);
            modify_++;
        }
    }

    void ToLower()
    {
        while (modify_ < username_.size()) {
            username_[modify_] = tolower(username_[modify_]);
            modify_++;
        }
    }

    void Reset()
    {
        modify_ = 0;
    }

    friend std::ostream &operator<<(std::ostream &os, const UnLOLText &txt);

private:
    std::size_t modify_;
    std::string username_;
};

std::ostream &operator<<(std::ostream &os, const UnLOLText &txt)
{
    os << txt.username_;
    return os;
}

void Scene1()
{
    UnLOLText txt("hEy How aRe yoU dOinG!");
    txt.ToUpper();
    std::cout << "Serial upper: " << txt << std::endl;
    txt.Reset();
    txt.ToLower();
    std::cout << "Serial lower: " << txt << std::endl;
    txt.Reset();

}
```

Output:

```
Serial upper: HEY HOW ARE YOU DOING!
Serial lower: hey how are you doing!
```

The amount of such LOL text you receive is huge. So obviously you want
to unLOL the the text concurrently.

``` cpp
class UnLOLText {
public:
    UnLOLText(const std::string &name) :
    username_(name),
    modify_(0),
    flag(ATOMIC_FLAG_INIT)
    {
        srand((unsigned int)time(0));
    }

    void ToUpper()
    {
        while(flag.test_and_set(std::memory_order_seq_cst)) {
        }

        while (modify_ < username_.size()) {
            username_[modify_] = toupper(username_[modify_]);
            modify_++;
            thread_sleep();
        }
        flag.clear(std::memory_order_release);
    }

    void ToLower()
    {
        while(flag.test_and_set(std::memory_order_seq_cst)) {
        }

        while (modify_ < username_.size()) {
            username_[modify_] = tolower(username_[modify_]);
            modify_++;
            thread_sleep();
        }
        flag.clear(std::memory_order_release);
    }

    void Reset()
    {
        modify_ = 0;
    }

    friend std::ostream &operator<<(std::ostream &os, const UnLOLText &txt);

private:
    void thread_sleep()
    {
        std::this_thread::sleep_for(std::chrono::microseconds(rand()%5+1));
    }

    size_t modify_;
    std::string username_;
    std::atomic_flag flag;
};

void Scene2()
{
    UnLOLText txt("hEy How aRe yoU dOinG!");

    std::thread tab1(&UnLOLText::ToUpper, &txt);
    std::thread tab2(&UnLOLText::ToLower, &txt);

    tab1.join();
    tab2.join();

    std::cout << "Concurrent random: " << txt << std::endl;

    txt.Reset();
}
```

Using the `std::atomic_flag` you can set the flag as soon as one of the
threads select a routine and then clear it only after the entire
modification is done. So, using `std::atomic_flag` you’re randomly
selecting a thread and blocking all the rest.

This almost sounds like what `std::mutex` does right? In fact, using
`std::atomic_flag` you can implement your own mutex object.

``` cpp
class CustomMutex {
public:
    CustomMutex() :
    flag(ATOMIC_FLAG_INIT)
    {}

    void lock()
    {
        while(flag.test_and_set(std::memory_order_seq_cst)) {
        }
    }

    void unlock()
    {
        flag.clear(std::memory_order_release);
    }

private:
    std::atomic_flag flag;
};

class UnLOLText {
public:
    UnLOLText(const std::string &name) :
    username_(name),
    modify_(0)
    {
        srand((unsigned int)time(0));
    }

    void ToUpper()
    {
        mutex_.lock();

        while (modify_ < username_.size()) {
            username_[modify_] = toupper(username_[modify_]);
            modify_++;
            thread_sleep();
        }

        mutex_.unlock();
    }

    void ToLower()
    {
        mutex_.lock();

        while (modify_ < username_.size()) {
            username_[modify_] = tolower(username_[modify_]);
            modify_++;
            thread_sleep();
        }

        mutex_.unlock();
    }

    void Reset()
    {
        modify_ = 0;
    }

    friend std::ostream &operator<<(std::ostream &os, const UnLOLText &txt);

private:

    void thread_sleep()
    {
        std::this_thread::sleep_for(std::chrono::microseconds(rand()%5+1));
    }

    size_t modify_;
    std::string username_;
    CustomMutex mutex_;
};
```

And since you’re so far, you can even just reap the benefits of
`std::lock_guard` for locking and unlocking the mutex for you. Remember,
`std::lock_guard` is based on RAII principles, so you get a
exception-safe guarantee that no matter what the rest of the code does
(except deadlock) your mutex will get unlocked.

``` cpp
class UnLOLText {
public:

    UnLOLText(const std::string &name) :
    username_(name),
    modify_(0)
    {
        srand((unsigned int)time(0));
    }

    void ToUpper()
    {
        std::lock_guard<CustomMutex> lock(mutex_);
        while (modify_ < username_.size()) {
            username_[modify_] = toupper(username_[modify_]);
            modify_++;
            thread_sleep();
        }
    }

    void ToLower()
    {
        std::lock_guard<CustomMutex> lock(mutex_);
        while (modify_ < username_.size()) {
            username_[modify_] = tolower(username_[modify_]);
            modify_++;
            thread_sleep();
        }
    }

    void Reset()
    {
        modify_ = 0;
    }

    friend std::ostream &operator<<(std::ostream &os, const UnLOLText &txt);

private:
    void thread_sleep()
    {
        std::this_thread::sleep_for(std::chrono::microseconds(rand()%5+1));
    }

    size_t modify_;
    std::string username_;
    CustomMutex mutex_;
};
```

Big deal, right? We just reinvented something that is already provided
by the C++ standard library. And I guarantee they’ve a better
implementation of the mutex. So, what can we extra out of working at
such low level?

With `std::atomic_flag` you must’ve noticed we use
`std::memory_order_seq_cst` and `std::memory_order_release`, what are
they? They specify the memory ordering of operations. Let take the red
pill and follow down the memory ordering hole.

### Memory ordering

This is probably the weirdest topic you’ll encounter as a programmer, as
this will put some doubts over your knowledge of how you thought
instructions execute at the lower level.

First lets take a look at all types of memory orderings possible. Memory
orderings are classified for 3 major classes of operations

1.  **Store** : `seq_cst`, `release` and `relaxed`
2.  **Load** : `seq_cst`, `acquire`, `consume` and `relaxed`
3.  **Exchange** : `seq_cst`, `acq_rel`, `acquire`, `release`, `consume`, `relaxed`

If we think in terms of different memory models available, we can
classify these operations as:

1.  **Default** : `seq_cst`
2.  **Unordered** : `relaxed`
3.  **Lock based** : `acquire`, `consume`, `release` and `acq_rel`

You’re already familiar with sequentially consistent (`seq_cst`) for this
is what you’ve been doing all your life. You see a piece of code and you
follows through the lines of code top to bottom, because that’s how the
execution works, right? Let’s see.

Let’s say there’s new trend that every awesome software company is
following. They have placed a coffee machine and a fedora hat machine at
the entrance lobby. And they require every employee to have a fedora hat
on their heads or a cup of coffee in their hands to enter the office.
They certainly like whe the employee gets both the items. According to a
survey this allegedly increases the hip level of the employee and brings
more energy and productivity in the office. Say each of these items
increment the employee’s hip level by 1.

So company A tries to implement this with the default memory model. They
noticed that some employees prefer wearing the hat first and then
holding the coffee, while other hold the coffee first and then wear the
hat. So they have two security systems, one that waits until employee
wears a hat and then it checks if the employee also has a coffee. The
second one does completely opposite, it first waits for the employee to
hold the coffee and then checks if the employee also has a hat on. After
both the security systems have reported back, the doors decides
whether to grant entry or not.

``` cpp
namespace defult {
    std::atomic<bool> hasHat;
    std::atomic<bool> hasCoffee;
    std::atomic<int> hipLevel;
    
    void WearHat()
    {
        std::this_thread::sleep_for(std::chrono::seconds(rand()%3+1));
        hasHat.store(true, std::memory_order_seq_cst);
    }

    void HoldCoffee()
    {
        std::this_thread::sleep_for(std::chrono::seconds(rand()%3+1));
        hasCoffee.store(true, std::memory_order_seq_cst);
    }

    void CheckHatAndCoffee()
    {
        while (!hasHat.load(std::memory_order_seq_cst)) {
            /* wait till employee gets a hat */
        }

        if (hasCoffee.load(std::memory_order_seq_cst)) {
            hipLevel++;
        }
    }

    void CheckCoffeeAndHat()
    {
        while (!hasCoffee.load(std::memory_order_seq_cst)) {
            /* wait till employee gets a coffee */
        }

        if (hasHat.load(std::memory_order_seq_cst)) {
            hipLevel++;
        }
    }

    void EmployeeEnter()
    {
        hasHat = false;
        hasCoffee = false;
        hipLevel = 0;

        std::thread a(WearHat);
        std::thread b(HoldCoffee);
        std::thread c(CheckHatAndCoffee);
        std::thread d(CheckCoffeeAndHat);

        a.join();
        b.join();
        c.join();
        d.join();

        if (hipLevel == 0) {
            std::cout << "Entry denied" << std::endl;
        } else {
            std::cout << "Entry granted with hip level: " << hipLevel << std::endl;
        }
    }
}
```

If you follow the code, you’ll notice that there is no way an employee
can be denied an entry. No matter how long an employee takes to get a
coffee or a hat, as soon as he does one thing the observing security
personnel will check the other item, if they have it good, otherwise
whenever they get the other item the second security will activate and
this time employee will definitely pass the test, as they already have
the first item.

Company B follows the unordered memory model.

``` cpp
namespace unordered {
    std::atomic<bool> hasHat;
    std::atomic<bool> hasCoffee;
    std::atomic<int> hipLevel;

    void GetThings()
    {
        hasHat.store(true, std::memory_order_relaxed);
        hasCoffee.store(true, std::memory_order_relaxed);
    }

    void CheckCoffeeAndHat()
    {
        while (!hasCoffee.load(std::memory_order_relaxed)) {
            /* wait till employee gets a coffee */
        }

        if (hasHat.load(std::memory_order_relaxed)) {
            hipLevel++;
        }
    }

    void EmployeeEnter()
    {
        hasHat = false;
        hasCoffee = false;
        hipLevel = 0;

        std::thread a(GetThings);
        std::thread b(CheckCoffeeAndHat);

        a.join();
        b.join();

        if (hipLevel == 0) {
            std::cout << "Entry denied" << std::endl;
        } else {
            std::cout << "Entry granted with hip level: " << hipLevel << std::endl;
        }
    }

}
```

They came up with the idea that they don’t really need two security
personnels. What they instead do is that they instruct their employees
to wear a hat and get coffee. So, the security system has to only test for the
coffee, because the employee must already have the hat by then. But this
can procedure can fail, some of the employees can be denied entry. This
is because the operations aren’t sequentially consistent anymore. When
an employee is instructed to `GetThings()`, the employee sees it as there
are 2 tasks they have to complete in *relaxed* manner, that is, perform
whatever seems convenient. The employee has no idea if any other thread
is monitoring its activities. It just cares enough that by the time it
has to exit `GetThings()` it need to have executed both the tasks. So, in
case the employee feels like getting the coffee first and then the hat,
there’s nobody stopping them. While, the security system is under
false impression that whenever a employee has a coffee in their hands
they must also have a hat on their heads. So every once in a while the
security system can encounter an employee that has a coffee in his hands but
not hat yet, but the security system doesn’t waits for the employee to get the
hat and instead immediately runs them through the door, which obviously
denies them the entry. And it’s all due to the misunderstood relaxed
memory ordering.

Company C learns the lessons from both companies A and B, and wants to
get the best of both worlds. So it adopts the lock based memory model.

``` cpp
namespace lock {
    std::atomic<bool> hasHat;
    std::atomic<bool> hasCoffee;
    std::atomic<int> hipLevel;

    void GetThings()
    {
        hasHat.store(true, std::memory_order_relaxed);
        hasCoffee.store(true, std::memory_order_release);
    }

    void CheckCoffeeAndHat()
    {
        while (!hasCoffee.load(std::memory_order_acquire)) {
            /* wait till employee gets a coffee */
        }

        if (hasHat.load(std::memory_order_relaxed)) {
            hipLevel++;
        }
    }

    void EmployeeEnter()
    {
        hasHat = false;
        hasCoffee = false;
        hipLevel = 0;

        std::thread a(GetThings);
        std::thread b(CheckCoffeeAndHat);

        a.join();
        b.join();

        if (hipLevel == 0) {
            std::cout << "Entry denied" << std::endl;
        } else {
            std::cout << "Entry granted with hip level: " << hipLevel << std::endl;
        }
    }
}
```

Here the all the tasks are still *relaxed*, except for the check on
coffee. The test on coffee is set as the synchronization point. Here
`hasCoffee` serves as a token that both the employee and the security
agrees on. The employee is free to do whatever it wishes to do in
whatever order if they agree to perform the store on `hasCoffee` at the
exact point as they’re expected to. It serves as a kind of checkpoint.
Whenever an employee gets a coffee, it means that they have done all the
prior tasks in whatever order that seems fit, nobody cares. So whenever
the security system sees an employee has a coffee in their hands, it is
guaranteed that all the tasks before it have been completed. So, now the
check for the hat can be successfully executed.

Company D took a slightly different approach than company C.

``` cpp
namespace lock2 {
    std::atomic<bool> hasHat;
    std::atomic<bool> hasCoffee;
    std::atomic<int> hipLevel;

    void GetThings()
    {
        std::this_thread::sleep_for(std::chrono::seconds(rand()%3+1));
        hasHat.store(true, std::memory_order_relaxed);
        hasCoffee.store(true, std::memory_order_release);
    }

    void CheckCoffeeAndHat()
    {
        int naps_taken = 0;
        while (!hasCoffee.load(std::memory_order_consume)) {
            /* nap for a while */
            naps_taken++;
            std::this_thread::sleep_for(std::chrono::seconds(rand()%2+1));
        }

        std::cout << "Naps: " << naps_taken << std::endl;

        if (hasHat.load(std::memory_order_relaxed)) {
            hipLevel++;
        }
    }

    void EmployeeEnter()
    {
        hasHat = false;
        hasCoffee = false;
        hipLevel = 0;

        std::thread a(GetThings);
        std::thread b(CheckCoffeeAndHat);

        a.join();
        b.join();

        if (hipLevel == 0) {
            std::cout << "Entry denied" << std::endl;
        } else {
            std::cout << "Entry granted with hip level: " << hipLevel << std::endl;
        }
    }
}
```

It still uses lock based memory model, but instead of asking the
security system to use acquire based loop they use consume based loop. What
this means is that instead of constantly monitoring employees, the
security system can take a nap once in a while and whenever they wake up they
just assume that the employee has a coffee in their hands, if not it
can go back to sleep. This approach works good when you have a somewhat
predictable data on how long does an average employee takes to
`GetThings()`.

### Atomics and Objecitve-C

Coming over to Objective-C, atomicity is simplified. All the properties
are by default atomic. This is good news because if you’re using
multiple threads to update the same property, you will always have a
valid value for that property. But this doesn’t means that the entire
object will be valid.

Let’s consider a employee record example:

``` objc
@interface Employee : NSObject

@property (copy) NSString *firstName;
@property (copy) NSString *lastName;
@property int coffeeConsumed; // in litres

- (NSString *)description;
@end



@implementation Employee
- (id)init
{
    self = [super init];
    if (!self) {
        return nil;
    }

    _firstName = [@"Monty" copy];
    _lastName = [@"Burns" copy];
    _coffeeConsumed = 9235;

    return self;
}



- (void)dealloc

{

    self.firstName = nil;

    self.lastName = nil;

    [super dealloc];

}



- (NSString *)description;

{

    return [NSString stringWithFormat:@"%@ %@: %@ L",

            _firstName,

            _lastName,

            @(_coffeeConsumed)];

}

@end
```

We simply create a employee with some default values. Now suppose we try
to update a single record concurrently

``` objc

void updateRecord()

{
    dispatch_group_t wait = dispatch_group_create();
    dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);

    Employee *emp = [[Employee alloc] init];

    dispatch_group_enter(wait);
    dispatch_async(queue, ^{
        [NSThread sleepForTimeInterval:rand()/(NSTimeInterval)RAND_MAX];
        emp.firstName = @"Homer";
        [NSThread sleepForTimeInterval:rand()/(NSTimeInterval)RAND_MAX];
        emp.lastName = @"Simpson";
        [NSThread sleepForTimeInterval:rand()/(NSTimeInterval)RAND_MAX];
        emp.coffeeConsumed = 2045;
        dispatch_group_leave(wait);
    });

    dispatch_group_enter(wait);
    dispatch_async(queue, ^{
        [NSThread sleepForTimeInterval:rand()/(NSTimeInterval)RAND_MAX];
        emp.firstName = @"Lenny";
        [NSThread sleepForTimeInterval:rand()/(NSTimeInterval)RAND_MAX];
        emp.lastName = @"Leonard";
        [NSThread sleepForTimeInterval:rand()/(NSTimeInterval)RAND_MAX];
        emp.coffeeConsumed = 127;
        dispatch_group_leave(wait);
    });

    dispatch_group_enter(wait);
    dispatch_async(queue, ^{
        [NSThread sleepForTimeInterval:rand()/(NSTimeInterval)RAND_MAX];
        emp.firstName = @"Carl";
        [NSThread sleepForTimeInterval:rand()/(NSTimeInterval)RAND_MAX];
        emp.lastName = @"Carlson";
        [NSThread sleepForTimeInterval:rand()/(NSTimeInterval)RAND_MAX];
        emp.coffeeConsumed = 598;
        dispatch_group_leave(wait);
    });

    dispatch_group_enter(wait);
    dispatch_async(queue, ^{
        [NSThread sleepForTimeInterval:rand()/(NSTimeInterval)RAND_MAX];
        NSLog(@"%@", emp);
        dispatch_group_leave(wait);
    });

    dispatch_group_wait(wait, DISPATCH_TIME_FOREVER);
    [emp release];
}

int main(int argc, char * argv[]) {
    @autoreleasepool {
        for (int i = 0; i < 3; ++i) {
            srand(time(0));
            updateRecord();
        }
        NSLog(@"Done");
    }
    return 0;
}
```

Output

```
2014-11-01 16:11:41.496 a.out[19307:1403] Carl Simpson: 9235 L
2014-11-01 16:11:43.314 a.out[19307:1a03] Homer Burns: 9235 L
2014-11-01 16:11:46.732 a.out[19307:1a03] Lenny Simpson: 2045 L
2014-11-01 16:11:47.558 a.out[19307:507] Done
```

You can see that even though the emp object is absurd every time, but
yet all of its atomic properties have valid values all the time.

As far as I’m aware of nothing is known about atomicity and Swift, but
I’m guessing it would be close to the Objective-C model.

As usual the code for today’s experiment is available at
[github.com/chunkyguy/ConcurrencyExperiments](https://github.com/chunkyguy/ConcurrencyExperiments/tree/master/106_Atomics).

Have fun!
