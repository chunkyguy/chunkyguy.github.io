---
layout: post
title:  "Concurrency: Thread-safe Singletons"
date:   2014-10-13 00:02:00 +0530
categories: concurrency
---

Love them or hate them, you just can not ignore
[singletons](http://127.0.0.1/rants/?tag=singleton). Let’s try
experimenting with singletons and concurrency.

Singletons are objects that you wish to have only a single instance all
over your program lifecycle. Usually, you want to create singleton
objects lazily. Here’s a typical way you can create a singleton in a
single threaded application.

```  cpp
class Library {
public:
    static Library *SharedInstance()
    {
        if (!shared_) {
            shared_ = new Library();
        }
        return shared_;
    }

    static void Dealloc()
    {
        if (shared_) {
            delete shared_;
        }
    }
    
    Library() :
    books_({
        {"Ernest Hemingway", "Old man and the sea"},
        {"Aldous Huxley", "Brave new world"},
        {"Aldous Huxley", "The Genius and the Goddess"},
        {"Aldous Huxley", "Antic Hay"},
        {"Salman Rushdie", "Midnight's children"},
        {"Fyodor Dostovesky", "Crime and punishment"},
        {"Boris Pasternak", "Doctor Zhivago"},
        {"Bernard Shaw", "Arms and the Man"},
        {"Bernard Shaw", "Man and Superman"}
    })
    {
        std::cout << "Library " << this << std::endl;
    }

    ~Library()
    {
        std::cout << "~Library " << this << std::endl;
    }

    std::vector<std::string> GetBooks(const std::string &author) const
    {
        typedef std::multimap<std::string, std::string>::const_iterator BookIt;

        std::pair<BookIt, BookIt> range =  books_.equal_range(author);
        std::vector<std::string> bookList;

        for (BookIt it = range.first; it != range.second; ++it) {
            bookList.push_back(it->second);
        }

        return bookList;
    }

    std::size_t GetSize() const
    {
        return books_.size();
    }

private:
    std::multimap<std::string, std::string> books_;
    static Library *shared_;
};

Library *Library::shared_ = nullptr;

void PrintBooks(const std::string &author)
{
    std::cout << "Searching for author: " << author << std::endl;
    std::vector<std::string> list = Library::SharedInstance()->GetBooks(author);

    std::copy(list.begin(), list.end(), std::ostream_iterator<std::string>(std::cout, "\n"));
}

void PrintSize()
{
    std::cout << "Library Size: " << Library::SharedInstance()->GetSize() << " books" << std::endl;

}
```

When invoked with:

``` cpp
void SerialImpl()
{
    std::string searchAuthor("Aldous Huxley");
    PrintBooks(searchAuthor);
    PrintSize();
}

void ThreadsafeSingleton_main()
{
    SerialImpl();
    Library::Dealloc();
}
```

We get

```
Searching for author: Aldous Huxley
Library 0x100a6bfe0
Brave new world
The Genius and the Goddess
Antic Hay
Library Size: 9 books
~Library 0x100a6bfe0
main() exit
```

We see that our singleton actually gets created lazily when actually
required and also gets deallocated when the program finally quits,
thanks to the Dealloc() function.

Next, lets try calling it from multiple threads.

``` cpp
void ConcurrentImpl()
{
    std::string searchAuthor("Aldous Huxley");
    std::thread bookSearchThread(PrintBooks, std::ref(searchAuthor));
    std::thread libSizeThread(PrintSize);

    bookSearchThread.join();
    libSizeThread.join();
}
```

This might actually work or not depending on how the threads are
switching on your machine. To see what is problem with this code, let’s
deliberately sleep the thread while creating the singleton instance.

``` cpp
    static Library *SharedInstance()
    {
        if (!shared_) {
            std::this_thread::sleep_for(std::chrono::seconds(1));
            shared_ = new Library();
        }
        return shared_;
    }
```

Now, if you run it you might see two instances of Library objects
getting created.

``` 
Searching for author: Aldous Huxley
Library Size: Library 0x100b84fe0
Brave new world
The Genius and the Goddess
Antic Hay
Library 0x100ba4fe0

9 books

~Library 0x100ba4fe0

main() exit
```

There are two problems here. First of all, we’ve two instances of the
Library object and second, because our Dealloc routine depends on the
guarantee that at any given moment we have a single instance of the
Library object, so the other Library object never gets released. This is
a classic example of a memory leak.

We can solve this issue with two techniques. First is using the C++11’s
static initialization.

Let’s modify our Library class and instead of creating the singleton at
heap, we create a static instance of a Library object.

``` cpp
static Library &SharedInstance()
{
    static Library lib;
    return lib;
}
```

And modifying our calls

``` cpp

void PrintBooks(const std::string &author)
{
    std::cout << "Searching for author: " << author << std::endl;
    std::vector<std::string> list = Library::SharedInstance().GetBooks(author);
    std::copy(list.begin(), list.end(), std::ostream_iterator<std::string>(std::cout, "\n"));
}

void PrintSize()
{
    std::cout << "Library Size: " << Library::SharedInstance().GetSize() << " books" << std::endl;
}
```

If we now run our code, we get

``` 
Searching for author: Aldous Huxley
Library 0x10006d3c8
Brave new world
The Genius and the Goddess
Antic Hay
Library Size: 9 books
main() exit
~Library 0x10006d3c8
```

This works because with C++11 we’re now guaranteed to have the static
variables created thread-safely. Another benefit with this approach is
that we don’t have to implement any Library::Dealloc() function as
static data gets deallocated automatically by the runtime. If this
satisfies your requirements, this is best way out.

But, this won’t be the best fit for some cases. Say, you want to have
Library singleton object only in heap memory. Also, if you’re using
Objective-C this won’t work as you’re not allowed to allocate memory of
custom objects on the stack. This brings us to our second method: Using
C++ thread library.

Let’s switch back to our original implementation, with Library object in
heap memory. The first thing that we can try is using mutex.

``` cpp
    static Library *SharedInstance()
    {
        std::lock_guard<std::mutex> lock(mutex_);

        if (!shared_) {
            std::this_thread::sleep_for(std::chrono::seconds(1));
            shared_ = new Library();
        }

        return shared_;
    }
```

And this works as expected:

```
Searching for author: Aldous Huxley
Library Size: Library 0x100b92fe0
Brave new world
The Genius and the Goddess
Antic Hay

9 books

~Library 0x100b92fe0

main() exit
```

The only problem with this code is that Library::SharedInstance() is
going to be called from plenty of places, and this code unnecessary lock
and unlock the mutex at each call. What we actually need it the mutex
only for cases when the single instance has not been created, all other
times we can just simple pass the pointer back. This brings us to the
infamous [double checking lock
design](http://en.wikipedia.org/wiki/Double-checked_locking).

``` cpp
    static Library *SharedInstance()
    {
        if (!shared_) {
            std::lock_guard<std::mutex> lock(mutex_);
            if (!shared_) {
                std::this_thread::sleep_for(std::chrono::seconds(1));
                shared_ = new Library();
            }
        }

        return shared_;
    }
```

This code looks great right? It applies the mutex only when creating the
shared instance, every other time it just returns the pointer. But this
has a serious bug. This code would work almost all the time, except when
you’ve forgotten about it. It’s so hard to debug that you probably can’t
even simulate the issue in the debug environment and unfortunately you
won’t ever see this bug again for probably a million clock cycles. This
sounds like a programmer’s nightmare right?

Suppose you’re running this code with two thread ‘bookSearchThread’ and
‘libSizeThread’. bookSearchThread comes in and sees the shared\_ is
NULL. It would acquire the lock and start allocating and initializing
the instance. bookSearchThread is half-way done, it has allocated a
space in the heap, so the shared\_ isn’t NULL anymore, but
bookSearchThread is not done initializing the instance yet, just the
allocation. Something like in Objective-C as:

``` objc
Library *shared = [[Library alloc] init];
```

Suddenly, the thread switching happens and libSizeThread comes in. It
sees the shared\_ isn’t NULL. It says OK, good somebody must have
already created this instance, lets use it straightaway. This could
result in undefined behavior, as libSizeThread is now using an object
that isn’t actually fully initialized.

This is such a great problem that almost every thread library provides a
way to make this work. In C++11, we can use the std::call\_once

``` cpp
class Library {

private:
    static std::once_flag onceFlag_;

    /* other data members */
public:
    static Library *SharedInstance()
    {
        std::call_once(onceFlag_, []{
            std::this_thread::sleep_for(std::chrono::seconds(1));
            shared_ = new Library();
        });

        return shared_;
    }

    /* other member functions */
}
```

Even GCD provides something similar:

``` objc
+ (instancetype)sharedInstance
{
    static dispatch_once_t onceFlag;
    static id shared_;

    dispatch_once(&once, ^{
        shared_ = [[[self class] alloc] init];
    });

    return shared_;
}
```

These helper objects from standard libraries are provided just to
guarantee that the object allocation and initialization is complete
before setting the flag and every other time, it just returns
immediately. So, no more race conditions for us.

So there it is, singletons the thread-safe way.

The code for todays experiment is available at
[github.com/chunkyguy/ConcurrencyExperiments](https://github.com/chunkyguy/ConcurrencyExperiments/blob/master/103_SharedData/ThreadsafeSingleton.cpp).
Check it out and have fun experimenting on your own.
