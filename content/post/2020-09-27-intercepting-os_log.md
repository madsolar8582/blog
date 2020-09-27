---
title: "Intercepting os_log"
subtitle: ""
date: 2020-09-27T11:00:00-05:00
tags: ["logging"]
---

Apple has been adjusting the [OSLog](https://developer.apple.com/documentation/oslog?language=objc) APIs with each major release. However, this year promised a big leap in functionality as iOS apps would get access to the log store just like macOS. Unfortunately, Apple pulled the API in the GM builds of iOS 14 and Xcode 12 after the entire summer went by with the API never working. That being said, it did lead to a little exercise in reverse engineering to see if I could get the log messages and forward them to another logging API.

The OSLog APIs are C functions with multiple layers of macros used to implement them. This leads to reverse engineering being a bit more difficult since we cannot use the Objective-C runtime to swizzle method calls and the macros themselves are hard to parse as they're layered and use `__builtin` functions so you never see the implementation until it is in assembly. Regardless, we can at least see some of the implementation.

Apple suggests creating a dedicated logger to allow you to filter log messages (or use the default logger `OS_LOG_DEFAULT`):

```obj-c
os_log_t logger = os_log_create("com.example.myapp", "com.example.subsystem");
```

Once created, you can log messages to it using the provided macros:

```obj-c
os_log_debug(logger, "My log message %{public}@", someString); // OS_LOG_TYPE_DEBUG
os_log_info(logger, "My log message %{private}@", someString); // OS_LOG_TYPE_INFO
os_log(logger, "My log message %{private}@", someString); // OS_LOG_TYPE_DEFAULT
os_log_error(logger, "My error log message %{public}@", someString); // OS_LOG_TYPE_ERROR
os_log_fault(logger, "My error log message %{public}@", someString); // OS_LOG_TYPE_FAULT
```

By calling those macros, eventually you end up at the implementations:

```obj-c
void _os_log_impl(void *dso, os_log_t log, os_log_type_t type, const char *format, uint8_t *buf, uint32_t size);

void _os_log_debug_impl(void *dso, os_log_t log, os_log_type_t type, const char *format, uint8_t *buf, uint32_t size);

void _os_log_error_impl(void *dso, os_log_t log, os_log_type_t type, const char *format, uint8_t *buf, uint32_t size);

void _os_log_fault_impl(void *dso, os_log_t log, os_log_type_t type, const char *format, uint8_t *buf, uint32_t size);
```

Most of the parameters of the implementation are quite simple:

* log - The logger
* type - The type of log message
* format - The log message format string
* buf - The arguments to the format string
* size - The size of the argument buffer

Where things get difficult is the dso parameter. It is opaque, so that is where the internal implementation details are stored. So, what do we do? Well, first we need to intercept the implementation functions so that we can poke around some more.

Using [fishhook](https://github.com/facebook/fishhook), we get an API to rebind symbols and that allows us to create our own implementation and swap out the original (like method swizzling):

```obj-c
@import Foundation;
@import os.log;
#import "fishhook.h"

// No fault impl as I don't log fault messages
void my_os_log_debug_impl(void *dso, os_log_t log, os_log_type_t type, const char *format, uint8_t *buf, uint32_t size);
void my_os_log_error_impl(void *dso, os_log_t log, os_log_type_t type, const char *format, uint8_t *buf, uint32_t size);
void my_os_log_impl(void *dso, os_log_t log, os_log_type_t type, const char *format, uint8_t *buf, uint32_t size);

@interface SomceClass : NSObject

@end

@implementation SomeClass

+ (void)load
{
  // This replaces the function calls, but you can store off the original after custom handling
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        const char *function1 = "_os_log_debug_impl";
        const char *function2 = "_os_log_error_impl";
        const char *function3 = "_os_log_impl";
        struct rebinding binding1 = {function1, my_os_log_debug_impl};
        struct rebinding bindings1[] = {binding1};
        rebind_symbols(bindings1, 1);
        struct rebinding binding2 = {function2, ,my_os_log_error_impl};
        struct rebinding bindings2[] = {binding2};
        rebind_symbols(bindings2, 1);
        struct rebinding binding3 = {function3, my_os_log_impl};
        struct rebinding bindings3[] = {binding3};
        rebind_symbols(bindings3, 1);
    });
}

void my_os_log_debug_impl(void *dso, os_log_t log, os_log_type_t type, const char *format, uint8_t *buf, uint32_t size)
{
  // do logging
}

void my_os_log_error_impl(void *dso, os_log_t log, os_log_type_t type, const char *format, uint8_t *buf, uint32_t size)
{
  // do logging
}

void my_os_log_impl(void *dso, os_log_t log, os_log_type_t type, const char *format, uint8_t *buf, uint32_t size)
{
  // do logging
}

@end
```

With the replacement methods, you get access to the log message, but there are two problems: 1) the dso pointer has all the trace information (file, function, line, etc.) and 2) the format string needs its arguments put in it. However, this is where Apple's implementation stops being "friendly" as the buffer of arguments was created with `__builtin_os_log_format` and the size parameter is created with `__builtin_os_log_format_buffer_size`. This means that the methodology to figure out how to decode the buffer isn't readily available to us and without proper functions to pull information out of the dso pointer, it remains opaque.

Alas, this is where the experiment ends. Although it was fun using fishhook, its use would mean an automatic rejection from the App Store as the techniques it uses are not allowed. Additionally, any more probing into the OSLog internals would trigger private API rejection. Even if that weren't the case, Apple would swing the ban hammer at you for trying to reverse engineer OSLog due to potentially getting access to strings declared as `private` (i.e. defeating the privacy measures of the log system) since that could expose information that you should not have access to. It is unclear as to whether or not iOS implements the same protection, but macOS 10.15 used to allow this until 10.15.3 where an [entitlement](https://saagarjha.com/blog/2019/09/29/making-os-log-public-on-macos-catalina/) was added as an additional defense.

Now, you might be wondering why you don't just create some wrapper API that logs to more than one system (including OSLog)? Well, the answer to that comes from Apple as well. Due to their use of multilayered macros, the tracing information would always be the call site location of the wrapper API and all strings would be public. If you don't care about this, then sure, go ahead. However, you lose a lot of the optimizations and features the OSLog implementation gets you, so I chose not to do this.

If you aren't dissuaded by any of this, you may have more luck if you have experience with the kernel as it appears a lot of the implementation [is there](https://opensource.apple.com/source/xnu/xnu-6153.141.1/libkern/os/log.c.auto.html).

---

Once day Apple should reintroduce the API to get access to the log store since you can get formatted log messages from it. Honestly, that has been the biggest ask since OSLog was introduced as it is somewhat limited without programmatic access for reporting purposes (you need to use a sysdiagnose to get logs right now).

Hopefully we do not need to wait for much longer...
