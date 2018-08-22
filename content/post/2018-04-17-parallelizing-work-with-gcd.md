---
title: "Parallelizing Work with GCD"
subtitle: "Create a Parallel Universe"
date: 2018-04-17T17:00:00-05:00
tags: ["concurrency", "parallelism", "gcd"]
---

In order to increase performance of your application or framework, it behooves you to parallelize your work if possible. By batching together groups of similar work, you can take advantage of multi-core systems to increase the responsiveness and throughput of your code. Good candidates for parallelization are network downloads, resource fetching and loading, and computationally heavy calculations. To accomplish this, [GCD](https://developer.apple.com/documentation/dispatch?language=objc) provides a few built-in options and you can also use some of its primitives to create more sophisticated synchronization tools.

## Countdown Latch
By leveraging `dispatch_semaphore_t`, a Countdown Latch can be created. This type of synchronization mechanism provides a way to block a calling thread for a certain amount of time that decrements with each task it performs. In order for the latch to eventually end, a maximum execution time needs to be defined so that the calling thread is not blocked eternally. Also, since the calling thread is blocked, the method containing the latch should be dispatched to another queue so that the main thread is not blocked.

Example:
```obj-c
static const uint64_t MAX_EXECUTION_TIME = 10; // Max time is defined in seconds
dispatch_semaphore_t semaphore = dispatch_semaphore_create(0);
NSDate *start = [NSDate date];

for (id task in tasks) {
  [someObject doWork:task withCompletionHandler:^{
    // Optionally process an individual result
    dispatch_semaphore_signal(semaphore);
  }];
  for (NSUInteger i = 0; i < tasks.count; ++i) {
    uint64_t elapsed = (uint64_t)[[NSDate date] timeIntervalSinceDate:start];
    uint64_t left = (MAX(MAX_EXECUTION_TIME, elapsed) - elapsed) * NSEC_PER_SEC;
    // If on a Mac, you can use dispatch_walltime to get a more fine-grained clock
    dispatch_semaphore_wait(semaphore, dispatch_time(DISPATCH_TIME_NOW, (int64_t)left));
  }
}

dispatch_async(dispatch_get_main_queue(), ^{
  // Process all results
});
```

## dispatch_group
Creating a Countdown Latch requires you to write some additional code, whereas `dispatch_group_t` provides just about the same functionality with less code. Using a group comes in two varieties, blocking and non-blocking, and can even be simplified further by skipping the `enter` and `leave` by dispatching to a work queue.

Blocking Example:
```obj-c
dispatch_group_t group = dispatch_group_create();

for (id task in tasks) {
  dispatch_group_enter(group);
  [someObject doWork:task withCompletionHandler:^{
    // Optionally process an individual result
    dispatch_group_leave(group);
  }];
}
    
dispatch_group_wait(group, DISPATCH_TIME_FOREVER); // Can be configured with dispatch_time
// Process all results
```

Non-Blocking Example:
```obj-c
dispatch_group_t group = dispatch_group_create();

for (id task in tasks) {
  dispatch_group_enter(group);
  [someObject doWork:task withCompletionHandler:^{
    // Optionally process an individual result
    dispatch_group_leave(group);
  }];
}
    
dispatch_group_notify(group, dispatch_get_main_queue(), ^{
  // Process all results
});
```

Dispatch Example:
```obj-c
dispatch_group_t group = dispatch_group_create();

for (id task in tasks) {
  dispatch_group_async(group, dispatch_get_global_queue(QOS_CLASS_UTILITY, 0), ^{
    [someObject doWork:task withCompletionHandler:^{
      // Optionally process an individual result
    }];
  });
}
    
dispatch_group_notify(group, dispatch_get_main_queue(), ^{
  // Process all results
});
```

## dispatch_apply
Finally, if you have a lot of work that you want to do that doesn't have the ability to benefit from [NSFastEnumeration](https://developer.apple.com/documentation/foundation/nsfastenumeration?language=objc) with [NSEnumerationConcurrent](https://developer.apple.com/documentation/foundation/nsenumerationoptions/nsenumerationconcurrent?language=objc) specified, you can create a parallel [for-loop](https://en.wikipedia.org/wiki/For_loop) with `dispatch_apply`. GCD only provides a synchronous version, but it can be easily made asynchronous by dispatching to another queue. Also, if you leverage `DISPATCH_APPLY_AUTO`, the system will attempt to use worker threads that match the configuration of the current thread as closely as possible.

Example:
```obj-c
dispatch_queue_t queue = dispatch_get_global_queue(QOS_CLASS_UTILITY, 0);
dispatch_apply((size_t)tasks.count, queue, ^(size_t index) {
  [someObject doWork:tasks[index] withCompletionHandler:^{
    // Optionally process an individual result
  }];
});
    
// Process all results
```

Auto Example:
```obj-c
dispatch_apply((size_t)tasks.count, DISPATCH_APPLY_AUTO, ^(size_t index) {
  [someObject doWork:tasks[index] withCompletionHandler:^{
    // Optionally process an individual result
  }];
});

// Process all results    
```

---

By using GCD, you can create parallelizable code anywhere in your application or framework. There are other options (e.g. [OpenMP](http://www.openmp.org/)) that add even more advanced parallelization features, but for 99% of use cases, GCD will suffice.
