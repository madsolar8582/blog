---
title: "Crashing New Relic"
subtitle: ""
date: 2018-09-16T09:00:00-05:00
tags: ["crash", "new relic"]
---

There are many analytics and crash reporting services available for mobile platforms like [Firebase](https://firebase.google.com/), [Microsoft App Center](https://azure.microsoft.com/en-us/services/app-center/), [New Relic](https://newrelic.com/), [Apteligent](https://www.apteligent.com/), [Instabug](https://instabug.com/), [Bugsnag](https://www.bugsnag.com/), & [Bugsee](https://www.bugsee.com/). Although Apple provides crash reporting through Xcode, many crashes go unnoticed as crashes are only uploaded if the device has developer analytics turned on and they are backing up their device, so developers choose to use these services so that they always get performance and error data to make their applications better. The apps that I work on use New Relic and I recently stumbled upon a scenario that causes the New Relic instrumentation to crash.

## The Setup

New Relic uses [method swizzling](https://nshipster.com/method-swizzling/) when instrumentation is enabled (default) to track the actions that occur in the application such that when it crashes, it can correlate user actions with the stack trace. However, New Relic does method swizzling a bit differently than the standard approach: [static C functions](https://blog.newrelic.com/engineering/right-way-to-swizzle/). This approach has the benefit of preventing the swizzled method from changing the selector name. This is important in the case of instrumentation as you don't want the instrumentation to appear at all.

Conventional:

```obj-c
+ (void)load
{
  static dispatch_once_t onceToken;
  dispatch_once(&onceToken, ^{
    Method m1 = class_getInstanceMethod([self class], @selector(viewWillDisappear:));
    Method m2 = class_getInstanceMethod([self class], @selector(example_viewWillDisappear:));

    BOOL didAdd = class_addMethod([self class], @selector(viewWillDisappear:), method_getImplementation(m2), method_getTypeEncoding(m2));
    if (didAdd) {
      class_replaceMethod([self class], @selector(example_viewWillDisappear:), method_getImplementation(m1), method_getTypeEncoding(m1));
    }
    else {
      method_exchangeImplementations(m1, m2);
    }     
  });
}

- (void)example_viewWillDisappear:(BOOL)animated
{
  // Do something different
  [self example_viewWillDisappear:animated];
}
```

New Relic:

```obj-c
static IMP __original_Method_Imp;

+ (void)load
{
  static dispatch_once_t onceToken;
  dispatch_once(&onceToken, ^{
    Method m = class_getInstanceMethod([self class], @selector(viewWillDisappear:));
    __original_Method_Imp = method_setImplementation(m, (IMP)__example_viewWillDisappear);
  });
}

void __example_viewWillDisappear(id self, SEL _cmd, BOOL animated)
{
  // Do something different
  ((void(*)(id,SEL,BOOL))__original_Method_Imp)(self,_cmd,animated);
}
```

## The Problem
The situation that I found was that if you are calling some view lifecycle methods with an irregular call chain (e.g. skipping the super class), it can cause problems. Here is an example of the child view controller's implementation of [viewWillDisappear](https://developer.apple.com/documentation/uikit/uiviewcontroller/1621485-viewwilldisappear?language=objc):

```obj-c
#ifndef __clang_analyzer__
- (void)viewWillDisappear:(BOOL)animated
{   
  Class class = [UIViewController class];
  if (class) {
    IMP imp = class_getMethodImplementation(class, _cmd);
    if (imp) {
      void (* callableIMP)(id, SEL, BOOL) = (__typeof(callableIMP)) imp;
      callableIMP(self, _cmd, animated);
    }
  }
}
#endif
```

With the code above, the child class skips the parent class's implementation of `viewWillDisappear` and directly calls the parent's superclass (in this case, `UIViewController`). When this code gets executed with New Relic instrumentation enabled it causes an infinite recursion and crashes the application as the stack overflows. To make matters worse, because the New Relic crash handler uses the instrumentation to add metadata to the crash, sometimes the crash never gets reported.

---

This issue was reported to New Relic and they declined to address it. To be fair to them, this is an edge case that technically isn't following Apple's API guidelines to the letter. However, I still think that some sort of infinite recursion detection would do their SDK some good as a crash reporter (and its utilities) should not crash themselves.
