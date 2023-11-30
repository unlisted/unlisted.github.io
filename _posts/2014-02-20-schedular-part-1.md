---
layout: post
title:  "Schedular Part 1"
date:   2014-02-20 
categories: c++
---

## Purpose
I wanted to create a scheduler that executes tasks at some interval. A task is simply an object that inherits from an interface and implements some function representing some small amount of work. The scheduler should periodically wake up, check for a task, execute if it exists and then goes to sleep. I don't really have a clear business case for this kind of scheduler, it's simply a learning exercise. I'll start out with the simplest case, ignoring some major sources of potential catastrophe, and then develop some better ways of handling things.

Here are the components

* Abstract task interface.
* Some concrete tasks.
* Simple FIFO queue
* Scheduler which executes tasks

### Abstract Task Interface
This is very simply an object from which tasks are meant to derive.
#### ITask.h
{% highlight cpp %}
#ifndef __ITASK_H__
#define __ITASK_H__

namespace scheduler
{
class ITask
{
public:
//virtual ~ITask();

virtual void DoWork() = 0;
};

}
#endif
{% endhighlight %}


#### Task.h
{% highlight cpp %}
#ifndef __TASK_H__
#define __TASK_H__

#include "ITask.h"

namespace scheduler
{

class Task : public ITask
{

public:
    Task();
    ~Task();

    virtual void DoWork();
};

}

#endif
{% endhighlight %}

#### Task.cpp
{% highlight cpp %}
#include <iostream>
#include "Task.h"
using namespace std;

namespace scheduler
{

void Task::DoWork()
{
        cout << "Completing Task." << endl;
}

}
{% endhighlight %}

As you can see the task doesn't do much, it simply display a message to standard out. It's fine for now.

### FIFO
For this first implementation I use a `std::queue`, which is really just an adapter to std::dequeue. A dequeue is just a collection in which you may add to the front or to the back. It's typically implemented as a linked list. Perfect for my needs since my scheduler is only interested in the oldest item in the queue. It will never need random access. The container may change in the future if I wanted to prioritize the tasks. It's interesting to note that the behavior of the scheduler is dependent on the data structure that stores the tasks.

### The Scheduler
Now on to the scheduler. The scheduler's job is a simple one. Go to sleep, wake up, check for a task, execute task, rinse, repeat.. The scheduler has methods to start, stop, add tasks and execute the loop which is handled in a separate thread. In the future we can define a period which represents the time to complete one cycle, .e.g, wake up, execute, go to sleep. With the concept of a period implemented we can then monitor the execution time and take some action if it extends past the period boundary. We can also adjust the sleep with respect to execution time. I'll save all those things for future posts.

#### Scheduler.h
{% highlight cpp %}
#ifndef __SCHEDULER_H__
#define __SCHEDULER_H_

#include <chrono>
#include <condition_variable>
#include <mutex>
#include <queue>
#include <thread>
#include "ITask.h"

namespace scheduler
{

class Scheduler
{
public:
    Scheduler();
    ~Scheduler();

    void AddTask(ITask* task);
    void Start();
    void Stop();
    void ScheduleLoop();

private:
    bool _isRunning;

    std::queue<ITask*> _tasks;
    std::mutex _mtx;
    std::condition_variable _cond;
};

}

#endif
{% endhighlight %}

Note the bare pointer that `AddTask` method takes as a parameter, we'll fix that and a ton of other things later. I'll go over the details after the implementation below.

#### Scheduler.cpp
{% highlight cpp %}
#include <chrono>
#include <iostream>
#include "Scheduler.h"

using namespace std;

namespace scheduler
{


Scheduler::Scheduler():
    _isRunning(false),
    _sleepPeriod(2000)

{
    cout << "Scheduler::Scheduler()" << endl;
}

Scheduler::~Scheduler()
{
}

void Scheduler::AddTask(ITask* task)
{
    if (!_isRunning)
        return;

         cout << "Adding task to queue" << endl;
        _tasks.push(task);
}

void Scheduler::Start()
{
    if (_isRunning)
        return;

    thread worker;

    cout << "Starting worker thread." << endl;

    try
    {
        worker = move(thread(&Scheduler::ScheduleLoop, this));
    } catch (const system_error& ex)
    {
        cout << ex.what() << endl;
    }

    worker.detach();
    _isRunning = true;
} 

void Scheduler::Stop()
{
    if (!_isRunning)
        return;

    // stop thread
    cout << "Stopping worker thread." << endl;
    _isRunning = false;
}

void Scheduler::ScheduleLoop()
{
    cout << "Entering schedule loop." << endl;
    unique_lock lk(_mtx);
    while(_isRunning)
    {
        cout << "Going to sleep for " << _sleepPeriod  << "ms" << endl;
        _cond.wait_for(lk, chrono::milliseconds(_sleepPeriod));

        cout << "Waking up." << endl;
        if(_tasks.empty())
            continue;

        // execute task
        cout << "Executing task." <DoWork();
    }

    cout << "scheduler loop has exited." << endl;
}

}
{% endhighlight %}

First look at `AddTask()` method.

`_tasks.push(task);`

push adds an item to the front of the queue and copies the value stored in task into the newly created item, in this case it copies the address pointed to by task. Later we'll use smart pointers so we don't need to worry about managing the lifetime of the object pointed to. Another thing to consider is concurrent access the queue. Since the queue is a shared resource between two threads, the main thread which pushes (producer) and the SchedulerLoop which pops (consumer), we must come up with a synchronization scheme. There are several options which I'll consider in future posts.

Next we'll take a look at `Start()`

`thread worker;`

This is going to create new thread object that isn't attached to any thread of execution.
{% highlight cpp %}
    try
    {
        worker = thread(&Scheduler::ScheduleLoop, this);
    } catch (const system_error& ex)
    {
        cout << ex.what() << endl;
    }
{% endhighlight %}

Create a new thread of execution, pass `ScheduleLoop` member function and move assign to worker thread object. Thread creation may throw a system exception so we try and catch it. The exception is simply written to the console. Later we can decided what to do with it. Log and exit, most likely.
{% highlight cpp %}
    worker.detach();
    _isRunning = true;
{% endhighlight %}

Notice that we detach the thread. If we don't detach the application will terminate. This happens because the thread dtor will terminate if the thread is still joinable. The worker object is created on the stack, so its dtor is called when `Start()` returns, terminating the application. This avoids one problem, but introduces another. We can no longer synchronize the detached thread. It will run independently until the application exits. We'll address this in the next iteration.

I'm skipping `Stop()` and going straight `ScheduleLoop()`

{% highlight cpp %}
void Scheduler::ScheduleLoop()
{
    cout << "Entering schedule loop." << endl;
    unique_lock lk(_mtx);
    while(_isRunning)
    {
        cout << "Going to sleep for " << _sleepPeriod  << "ms" << endl;
        _cond.wait_for(lk, chrono::milliseconds(_sleepPeriod));

        cout << "Waking up." << endl;
        if(_tasks.empty())
            continue;

        // execute task
        cout << "Executing task." <DoWork();
    }

    cout << "scheduler loop has exited." << endl;
}
{% endhighlight %}

Ok, first the lock. It's a mutex wrapper with some nice properties. I'm really only interested in the guarantee that on destruction its associated mutex will be unlocked barring an exception.

`unique_lock lk(_mtx);`

So the mutex is locked, any other thread attempting to lock the mutex is going to block. That gives us an opportunity for synchronization we can take advantage of later. isRunning is true, next comes the condition variable.

`_cond.wait_for(lk, chrono::milliseconds(_sleepPeriod));`

`wait_for()` takes a lock and a duration as parameters, unlocks the mutex associated with lock and waits for duration, then continues. Our loop is going to continue to try and unlock the mutex associated with the unique_lock at each iteration of the loop. Ok, all that's left to do is get the task and execute it.
{% highlight cpp %}
        ITask* task = _tasks.front();
        _tasks.pop();
        
        task->DoWork();
{% endhighlight %}

We store the task in a pointer to Itask object. We don't care what the task actually is, as long as it inherits from `ITask` and implements `DoWork()`. You probably notice that we call `front()` without checking if the queue is empty. More undefined behavior, add it to the list.

### Output
{% highlight plaintext %}
morgan@vapor:~/projects/development/scheduler$ ./scheduler 
Scheduler::Scheduler()
Starting worker thread.
Adding task to queue
Entering schedule loop.
Going to sleep for 2000ms
Waking up.
Executing task.
Completing Task.
Going to sleep for 2000ms
Waking up.
Going to sleep for 2000ms
Waking up.
Going to sleep for 2000ms
Waking up.
Going to sleep for 2000ms
CWaking up.
Going to sleep for 2000ms
{% endhighlight %}

### Bugs
* _tasks queue, check for empty.

### Concerns
* Concurrent access to queue
* Bare pointers
* Thread synchronization
* Task duration exceeds scheduler sleep time
* Task hangs

### Wishlist
* Logging. Very useful and preferable to cout
* Alternative queues. Change scheduler behavior

### Conclusion
So there you have it. A first working implementation of a periodic scheduler. I plan to address the items above in future posts. It's a work in progress, so please leave suggestions and comments.