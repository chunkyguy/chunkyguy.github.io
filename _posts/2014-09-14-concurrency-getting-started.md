---
layout: post
title:  "Concurrency: Getting started"
date:   2014-09-14 23:28:54 +0530
categories: concurrency
---

The time everybody was speculating for so long has finally arrived. The processors aren’t getting any faster. The processors are hitting the physical thresholds. If we try to run the processors, as they are with current technology, they’re getting over-heated. Moore’s law is breaking down. Quoting Herb Sutter, ‘The free lunch is over‘.

If you want your code to run faster on new generation of hardware, you have to consider concurrency. All modern computing devices have multi-cores in built. More and more languages, toolkits, libraries, frameworks are providing support of concurrency. Let’s talk about concurrency.

Starting from today, I’ll be experimenting with multithreading. I’ll be using the following technologies. First is the C++11 thread library that ships with the C++11 standard. Second is the libdispatch or Grand Central Dispatch (GCD) that is developed by Apple. Although, GCD isn’t a pure threading library. It’s more of a concurrency library that is built over threads. In GCD we normally think in terms of tasks and queues. It’s a level above threads. And, the third is NSOperation. NSOperation is another level up from GCD. NSOperation is pure OOP oriented concurrency API. I’ve used it quite often with Objective-C and I’m sure it works as smoothly with Swift.

Let’s start with our first experiment. Dividing a huge task into smaller chunks and let them run in parallel to each other.

So, first question, what task should we pick. Let’s pick sorting of a dataset. Lets say, we’ve an huge array and we want to sort it. We could do it using the famous Quicksort algorithm as:
``` cpp
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
```

The algorithm is pretty straightforward. Given a list of unsorted data, in each iteration we pick a pivot element and sort that element by moving all smaller elements on the left side and all larger elements on right. By the end of the iteration, our pivot element is in the sorted position with we have two unsorted subarrays.

This algorithm is a prefect pick for concurrency. As in each iteration we get our problem divided into two independent sub problems that we can easily execute in parallel.

Here’s my first approach:
``` cpp
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
```

And here’s my calling code:
``` cpp
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
```

I’ve added some code to profile how long does each of these tasks take to execute the data set. The dataset of 100 elements executes on my machine as:
```
QuickSort: 0.023ms
QuickerSort: 29.12ms
```

As you can see, the concurrent code takes a lot longer to execute than the single threaded one. One guess could be that the creating thread for each subproblem and them the extra load of context switching between the threads could be the reason behind the latency. Or you could also suggest that there are huge flaws in my implementation.

I accept, there could be huge flaws in my implementation, as this is my first take at core multi-threading with C++ thread library. But, still we can’t ignore the fact that we’re spawning a lot of threads with each iteration. The number of threads running concurrent at each iteration are getting added by a factor of 2. This implementation definitely needs to be improved.

This is where GCD shines. First of all, it’s easier to begin with. Keep in mind I’ve equal amount of experience with both the libraries. So, my code should be full of bugs and most probably I’m not using them in the best possible way. But, this is all part of the test and my experiments.

Anyhow, when I implement the same problem using Swift and GCD, like so:

``` swift
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
```

I get the following result:
```
quickSort: 784 ms
quickerSort: 772 ms
```

You obviously tell that the same implementation in Swift is quite slower than in C++, but that’s not the point. We’re not experimenting C++ vs Swift, but single thread vs multiple threads. And using GCD, we see quite a small improvement, even though our test case is so small. This is most probably because we’re not manually handling the thread management ourselves anymore. We simply let the dispatch_queue handle the thread management.

The sunshine of todays experiment is that if we make good use of multi threading, we have a lot of potential for getting a large improvements. And, this is more than enough to motivate me to dive further down the concurrency rabbit hole.

The source code for this experiment is available at the GitHub repository: https://github.com/chunkyguy/ConcurrencyExperiments

