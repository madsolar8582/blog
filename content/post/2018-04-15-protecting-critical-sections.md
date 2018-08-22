---
title: "Protecting Critical Sections"
subtitle: "Knock knock! Race Condition. Who's there?"
date: 2018-04-15T10:00:00-05:00
tags: ["concurrency", "threading", "parallelism", "gcd"]
---

When working on an application or framework, you eventually run into the problem of needing to do some work in a controlled environment. In order to guarantee the safety of the work being done, you want to lock down that particular path so that it can execute without state changing while it is running. This is what's called a [critical section](https://en.wikipedia.org/wiki/Critical_section). Luckily, the Apple development platform provides an abundance of options for implementing parallelizable code. However, each one has a different cognitive load when using them, so it may be in your best interest to use different implementations depending on the type of concurrency you are trying to achieve.

## POSIX Mutex
[POSIX](https://en.wikipedia.org/wiki/POSIX) conformant systems provide the pthreads library and it contains a mutex ([lock](https://en.wikipedia.org/wiki/Lock_(computer_science))) implementation: `pthread_mutex_t`. The API creates a blocking lock (that can optionally be configured as a recursive lock) and must be initialized and destroyed using specific functions. Since the functions take in a reference to the mutex, you can't pass in the mutex's property address, so you need to pass in the iVar directly. Therefore, most instances of these mutexes are defined in some global scope, rather than a property/iVar of an Objective-C class. However, that doesn't mean that you can't store the mutex as a property<a href="https://stackoverflow.com/questions/9086736/why-would-you-use-an-ivar" id="note1ref"><sup>1</sup></a>, you'll just need to dereference the backing iVar (which introduces `NULL` dereference issues if you aren't careful: `NULL` behaves differently than `nil`).

Another variant provided by the pthreads library is `pthread_rwlock_t`. This lock is a [readers/writer lock](https://en.wikipedia.org/wiki/Readers%E2%80%93writer_lock) and it allows for multiple threads to obtain the value of the critical section, but only allow one thread at a time to change the value of the critical section. Using this lock rather than the standard `pthread_mutex_t` for the case of code that can safely allow multiple read-only accessors, but only one write accessor is a performance boost.

Example:
```obj-c
// Non-Recursive
pthread_mutex_t mutex;
pthread_mutex_init(&mutex, NULL);
pthread_mutex_lock(&mutex);
[someObject doWork];
pthread_mutex_unlock(&mutex);
pthread_mutex_destroy(&mutex);

// Recursive
pthread_mutexattr_t attr;
pthread_mutexattr_init(&attr);
pthread_mutexattr_settype(&attr, PTHREAD_MUTEX_RECURSIVE);
pthread_mutex_t mutex;
pthread_mutex_init(&mutex, &attr);
pthread_mutex_lock(&mutex);
[someObject doWork];
pthread_mutex_unlock(&mutex);
pthread_mutex_destroy(&mutex);
pthread_mutexattr_destroy(&attr);
```

The source for Apple's implementation can be found [here](https://opensource.apple.com/source/libpthread/).

## @synchronized
Objective-C provides a language feature for synchronization in the form of the `@synchronized` directive. The synchronize directive can be applied to any Objective-C object, so you can either synchronize `self` or any property/associated object of the instance/class. Under the hood, the compiler turns the code into the following:
```obj-c
// Source
@synchronized(self) {
  [someObject doWork];
}

// Compiler replacement where object is the argument passed into the directive
@try {
  objc_sync_enter(object);
  [someObject doWork];
}
@finally {
  objc_sync_exit(object);
}
```

The sync functions in turn allocate and manage a `pthread_mutex_t` on your behalf and provides some error handling to release the lock in the event that an exception is thrown during execution. However, the exception handling and the [locking implementation itself](https://opensource.apple.com/source/objc4/objc4-723/runtime/objc-sync.mm.auto.html) with its object tables and [reentrancy](https://en.wikipedia.org/wiki/Reentrancy_(computing)) introduce overhead that can be eliminated by using an explicit lock if you don't need those features. But, by providing this as a language construct, you get easy to implement synchronization. As with everything in computer science, there are trade-offs to everything and if you can afford to pay the cost by making it up somewhere else, then you're ok.

The source for Apple's implementation can be found [here](https://opensource.apple.com/source/objc4/).

## os_unfair_lock
Prior to the release of iOS 10 & macOS 10.12, Apple provided atomic functionality via [kernel API](https://developer.apple.com/documentation/kernel/osatomic.h?language=objc) (particularly `OSSpinLock`). However, [spinlocks](https://en.wikipedia.org/wiki/Spinlock) were never available on iOS due to the nature of spinning [wasting CPU resources](https://en.wikipedia.org/wiki/Starvation_(computer_science)) and possibly causing a [deadlock](https://en.wikipedia.org/wiki/Deadlock). Now, that API is deprecated and a [new API](https://developer.apple.com/documentation/os/synchronization?language=objc) is available: `os_unfair_lock` and it is usable on all Apple platforms.

As the name suggests, it is [unfair](https://en.wikipedia.org/wiki/Unbounded_nondeterminism#Fairness). This means that if there are multiple threads trying to acquire this lock, there is no guarantee that resources will be allocated in a manner that allows all threads equal priority, allowing a single thread to hog the lock. This is a trade-off to allow the lock to perform better in cases where fairness is not needed and keep its overhead small.

Example:
```obj-c
os_unfair_lock unfairLock = OS_UNFAIR_LOCK_INIT;
os_unfair_lock_lock(&unfairLock);
[someObject doWork];
os_unfair_lock_unlock(&unfairLock);
```

The source for Apple's implementation can be found [here](https://opensource.apple.com/source/libplatform/).

## NSLock
If you feel more comfortable with an Objective-C object rather than dealing with C APIs, Apple provides a wrapper around the POSIX locks via [NSLock](https://developer.apple.com/documentation/foundation/nslock?language=objc) and its siblings [NSRecursiveLock](https://developer.apple.com/documentation/foundation/nsrecursivelock?language=objc), [NSDistributedLock](https://developer.apple.com/documentation/foundation/nsdistributedlock?language=objc), & [NSConditionLock](https://developer.apple.com/documentation/foundation/nsconditionlock?language=objc). Since these are wrappers, using the locks is as simple as calling `lock` and `unlock` (except NSDistributedLock & NSConditionLock since they have implementation specific APIs) as the underlying complexity has been abstracted away.

Example:
```obj-c
// Non-Recursive
NSLock *lock = [[NSLock alloc] init];
[lock lock];
[someObject doWork];
[lock unlock];

// Recursive
NSRecursiveLock *lock = [[NSRecursiveLock alloc] init];
[lock lock];
[someObject doWork];
[lock unlock];
```

## Grand Central Dispatch
Rather than using a lock, you can also use [queues](https://en.wikipedia.org/wiki/Priority_queue) (serial or concurrent). Submitting work to a [GCD](https://developer.apple.com/documentation/dispatch?language=objc) queue can be done synchronously or asynchronously and there are [barrier](https://en.wikipedia.org/wiki/Barrier_(computer_science)) variants that will wait for existing submissions to complete before executing themselves. The system provides default queues to perform work on: 

* `QOS_CLASS_USER_INTERACTIVE` aka the main (UI) queue
* `QOS_CLASS_USER_INITIATED` (née `DISPATCH_QUEUE_PRIORITY_HIGH`)
* `QOS_CLASS_DEFAULT` (née `DISPATCH_QUEUE_PRIORITY_DEFAULT`)
* `QOS_CLASS_UTILITY` (née `DISPATCH_QUEUE_PRIORITY_LOW`)
* `QOS_CLASS_BACKGROUND` (née `DISPATCH_QUEUE_PRIORITY_BACKGROUND`)

However, it is possible to create your own queues similar to how you can create your own threads. One thing to be aware of is that queues will deadlock if you enter them recursively, so you need to be careful with your dispatches (luckily, GCD takes care of [priority inversion](https://en.wikipedia.org/wiki/Priority_inversion) for you).

Example:
```obj-c
// Use DISPATCH_QUEUE_SERIAL for a serial queue
dispatch_queue_attr_t attr = dispatch_queue_attr_make_with_qos_class(DISPATCH_QUEUE_CONCURRENT, QOS_CLASS_UTILITY, 0);
dispatch_queue_t queue = dispatch_queue_create("com.example.concurrent-queue", attr);
// Block calling thread until execution completes
dispatch_sync(queue, ^{
  [someObject doWork];      
});
// Don't block calling thread and return immediately
dispatch_async(queue, ^{
  [someObject doWork];        
});
// Block the calling thread and wait until previously submitted blocks run    
dispatch_barrier_sync(queue, ^{
  [someObject doWork];        
});
// Don't block the calling thread and wait until previously submitted blocks run    
dispatch_barrier_async(queue, ^{
  [someObject doWork];        
});
```

The source for Apple's implementation can be found [here](https://opensource.apple.com/source/libdispatch/).

## NSOperationQueue
Like NSLock, Apple provides a wrapper around GCD in the form of [NSOperationQueue](https://developer.apple.com/documentation/foundation/nsoperationqueue?language=objc). Instead of taking blocks, NSOperationQueue takes subclasses of [NSOperation](https://developer.apple.com/documentation/foundation/nsoperation?language=objc) such as [NSBlockOperation](https://developer.apple.com/documentation/foundation/nsblockoperation?language=objc) and [NSInvocationOperation](https://developer.apple.com/documentation/foundation/nsinvocationoperation?language=objc) (you can create your own subclasses too). By leveraging concrete operations, you can also add dependencies so that certain operations do not execute unless their dependencies have completed and it also allows you to cancel operations that are in the process of executing or haven't started yet (you can also suspend the operation queue to pause all executions). Unlike GCD, the only global operation queue is the main queue, but like GCD, you can create your own operation queues.

Example:
```obj-c
NSOperationQueue *operationQueue = [[NSOperationQueue alloc] init];
operationQueue.name = @"com.example.concurrent-operation-queue";
operationQueue.maxConcurrentOperationCount = 10; // Default is 1, aka serial
operationQueue.qualityOfService = NSQualityOfServiceUtility;
// Create an anonymous block operation 
[operationQueue addOperationWithBlock:^{
  [someObject doWork];         
}];
// Add an instance of a concrete NSOperation
[operationQueue addOperation:someOperation];
```

## Semaphores
Both the pthread library and GCD also provide a [semaphore](https://en.wikipedia.org/wiki/Semaphore_(programming)) implementation as a form of synchronization (although the pthread implementation is deprecated). Semaphores are similar to locks in that a calling thread requests (locks) the shared resource and then signals (unlocks) when it is done using it. The semaphore behaves like a blocking lock and blocks the calling thread until it is available. Therefore, it is important to call into semaphore protected code with threads that share the same priority since they do not benefit from [priority inheritance](https://en.wikipedia.org/wiki/Priority_inheritance).

POSIX Example:
```obj-c
sem_t semaphore = sem_init(&semaphore, 0, 1);
sem_wait(&semaphore);
[someObject doWork];
sem_post(&semaphore);
sem_destroy(&semaphore);
```

GCD Example:
```obj-c
dispatch_semaphore_t semaphore = dispatch_semaphore_create(1);
// The time can be configured to be anything via the dispatch_time function
dispatch_semaphore_wait(semaphore, DISPATCH_TIME_FOREVER);
[someObject doWork];
dispatch_semaphore_signal(semaphore);
```

## Atomics
In the event that you want to go lockless, the [C11](http://en.cppreference.com/w/c/atomic) and [C++11](http://en.cppreference.com/w/cpp/atomic) standards introduced built-in atomics. These types and functions provide a concurrent implementation that does not use a lock and [guarantee progress](http://en.cppreference.com/w/cpp/language/memory_model#Threads_and_data_races) by eliminating [data races](https://en.wikipedia.org/wiki/Race_condition). Unfortunately, this only applies to the provided types and Objective-C classes are not included. However, this does mean that you can have these atomic types as properties in your class and not need to implement and additional concurrency logic since you just implement the getter and setter by calling the appropriate atomic function or provide convenience methods that will get and set the property without exposing the property publicly.

Example:
```obj-c
atomic_int aInt;
atomic_store(&aInt, 10); // Set initial value to 10
atomic_fetch_add(&aInt, 2); // Add 2
atomic_fetch_sub(&aInt, 1); // Subtract 1
```

## Performance
By modifying this [gist](https://gist.github.com/steipete/36350a8a60693d440954b95ea6cbbafc) by [Peter Steinberger](https://twitter.com/steipete) for Objective-C, this is how some of these options perform (average):

* Concurrent GCD Queue: 0.109s
* Dispatch Semaphore: 0.052s
* NSLock: 0.077s
* NSRecursiveLock: 0.104s
* PThread Mutex: 0.061s
* Recursive PThread Mutex: 0.088s
* PThread Semaphore: 2.100s
* Serial GCD Queue: 0.083s
* Spinlock: 0.046s
* Synchronize: 0.170s
* Unfair Lock: 0.050s

Metrics were collected on a 2016 15" MacBook Pro with a 2.6GHz Intel® Core™ [i7-6700HQ](https://ark.intel.com/products/88967/Intel-Core-i7-6700HQ-Processor-6M-Cache-up-to-3_50-GHz) CPU and 16GB of RAM running macOS 10.13.4 (17E199) with Xcode 9.4ß1 (9Q1004a) compiled with optimization level 3 and monolithic LTO. [Source](https://gist.github.com/madsolar8582/4537a61f5dedd9a8688cd343d41d5392)

## Troubleshooting Problems
Whenever concurrency is involved, bugs are bound to happen and chasing down these [heisenbugs](https://en.wikipedia.org/wiki/Heisenbug) are difficult since any number of factors like CPU speed, number of threads, running applications, etc. can influence the outcome of a concurrent operation. To help hunt these down, Xcode provides the [thread sanitizer](https://developer.apple.com/documentation/code_diagnostics/thread_sanitizer). TSan looks for errors in concurrent code by instrumenting the source code during compilation to check memory access during program execution. Therefore, it is important to actually exercise the code you want to debug since TSan can't check unexecuted code.

Another sanity checking tool is the [Dispatch](https://developer.apple.com/library/content/documentation/DeveloperTools/Conceptual/InstrumentsUserGuide/Instrument-Dispatch.html#//apple_ref/doc/uid/TP40004652-CH51-SW1) template available in Instruments. It allows you to observe how GCD is being used and identify potential implementation problems in your code. Along the same vein, the [Time Profiler](https://developer.apple.com/library/content/documentation/DeveloperTools/Conceptual/InstrumentsUserGuide/Instrument-TimeProfiler.html#//apple_ref/doc/uid/TP40004652-CH73-SW1) template can be used to observe how long threads are waiting and how long they take to execute.

---

Concurrency is hard and there is overhead no matter what option you choose. Therefore, you should pick the most performant option that you feel comfortable leveraging and makes sense for the critical section you're trying to protect. By reducing the cognitive load required to read and write your mutlithreaded code, you'll save yourself a lot of headaches later when trying to debug it (the code needs to work, not be the cleverest).

In the future, this may be a non-issue. Both [Swift](https://swift.org/) and [Rust](https://www.rust-lang.org/en-US/) are trying to create an environment where the programmer does not need to think too deeply into concurrent implementations. As of the time of this writing, Rust has labeled their approach [Fearless Concurrency](https://doc.rust-lang.org/book/second-edition/ch16-00-concurrency.html) and have provided a language guarantee that their [memory model](https://doc.rust-lang.org/book/second-edition/ch04-00-understanding-ownership.html) combined with their concurrency concepts prevent the programmer from [footgun-ing](https://en.wiktionary.org/wiki/footgun) themselves. Whereas Swift has not implemented their full [ownership model](https://github.com/apple/swift/blob/22530b922f4fcd79b31583ec834fadd90bff07bb/docs/OwnershipManifesto.md) yet due to not having a stable [ABI](https://en.wikipedia.org/wiki/Application_binary_interface) and the Swift maintainers have not approved the concurrency [proposal](https://github.com/apple/swift/blob/22530b922f4fcd79b31583ec834fadd90bff07bb/docs/proposals/Concurrency.rst).
