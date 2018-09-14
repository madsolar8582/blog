---
title: "Anti-Debugging"
subtitle: ""
date: 2018-09-14T18:00:00-05:00
tags: ["ptrace", "debugger", "mach", "sysctl"]
---

When developing apps, the debugger is your friend. When your app is running in production, the debugger is your enemy. To protect applications, sometimes developers put in code that only runs in non-debug mode that prevents debugger attachment or ends program execution once one is attached. This is a game of cat and mouse for those persons who like working around these limitations.

## Using ptrace

One of the simplest solutions is to use [ptrace](https://en.wikipedia.org/wiki/Ptrace). ptrace provides the ability to trace programs (hence the name). However, it also provides the flag `PT_DENY_ATTACH` to disable the functionality of ptrace. By doing so, you disable the ability for the debugger to debug. The challenge with this approach is that the ptrace API isn't public, so you have to be careful with how it is implemented as you could get rejected from the App Store. Also, to make things more difficult for reverse engineers, you can use `NS_INLINE` to tell the compiler to get rid of the function body. Now, this isn't a unbreakable solution, but it adds additional difficulty to the bypass process.

So, to use ptrace, you have to define the function pointer as ptrace will be loaded in manually and then redefine the deny flag value. Once you get a handle to ptrace, you can call exit to end the application's execution:
```obj-c
#import <dlfcn.h>
#import <sys/types.h>
#import <stdio.h>

typedef int (*ptrace_ptr_t)(int _request, pid_t _pid, caddr_t _addr, int _data);
#if !defined(PTRACE_DENY_ATTACH)
#define PTRACE_DENY_ATTACH 31
#endif
NS_INLINE void sanityCheck(void);

void sanityCheck(void)
{
    ptrace_ptr_t ptrace_ptr = (ptrace_ptr_t)dlsym(RTLD_SELF, "ptrace");
    ptrace_ptr(PTRACE_DENY_ATTACH, 0, 0, 0);
    exit(EXIT_FAILURE);
}
```

## Using sysctl

Another way of checking for the presence of the debugger is to use [sysctl](https://en.wikipedia.org/wiki/Sysctl). sysctl functions almost like the ptrace API, but uses the kernel to find out the information instead. To use sysctl, you create a MIB ([Management Information Base](https://en.wikipedia.org/wiki/Management_information_base)) to function as the query and then a `kinfo_proc` variable to hold the response. Once you have the response, you can check the flag contains `P_TRACED` and then exit:
```obj-c
#import <sys/types.h>
#import <stdio.h>
#include <sys/sysctl.h>
NS_INLINE void sanityCheck(void);

void sanityCheck(void)
{
    int retVal;
    int mib[4];
    struct kinfo_proc info;
    size_t size;
    info.kp_proc.p_flag = 0;
    
    mib[0] = CTL_KERN;
    mib[1] = KERN_PROC;
    mib[2] = KERN_PROC_PID;
    mib[3] = getpid();
    
    size = sizeof(info);
    __unused retVal = sysctl(mib, sizeof(mib) / sizeof(*mib), &info, &size, NULL, 0);
    
    if ((info.kp_proc.p_flag & P_TRACED) != 0) {
        exit(EXIT_FAILURE);
    }
}
```

## Using the Mach Subsystem

The last way (that I am aware of) using the oft forgotten [mach subsystem](https://en.wikipedia.org/wiki/Mach_(kernel)). The mach API is another way of interacting with the kernel to get information about your process. By using this approach, you can also prevent debuggers not using ptrace from attaching. To accomplish this, you need to define a struct similar to a MIB and then check if anything is configured with `THREAD_STATE_NONE`. Once you find it, exit:
```obj-c
#import <sys/types.h>
#import <stdio.h>
#include <mach/mach.h>
NS_INLINE void sanityCheck(void);

void sanityCheck(void)
{
    struct ios_exception_info
    {
        exception_mask_t masks[EXC_TYPES_COUNT];
        mach_port_t ports[EXC_TYPES_COUNT];
        exception_behavior_t behaviors[EXC_TYPES_COUNT];
        thread_state_flavor_t flavors[EXC_TYPES_COUNT];
        mach_msg_type_number_t count;
    };
    
    struct ios_exception_info *info = malloc(sizeof(struct ios_exception_info));
    __unused kern_return_t retVal = task_get_exception_ports(mach_task_self(), EXC_MASK_ALL, info->masks, &info->count, info->ports, info->behaviors, info->flavors);
    
    for (uint32_t i = 0; i < info->count; i++) {
        if (info->ports[i] != 0 || info->flavors[i] == THREAD_STATE_NONE) {
            exit(EXIT_FAILURE);
        }
    }
}
```

---

With these approaches, you can prevent debuggers from attaching to your applications. Now, most of the time, these functions are called when the application starts, but debuggers can be attached at any time. To combat this, you can run these checks on a timer in addition to startup. However, most reverse engineers know what to look for in the binary to see if these kinds of checks are implemented. By using inline functions, it makes it more difficult to patch, but they can be removed. Some developers have taken to using macros or writing assembly to implement the checks, but tools like [Frida](https://www.frida.re/) and [Hopper](https://www.hopperapp.com/) can help the reverse engineer defeat these protections quite quickly. So, if you choose to implement these techniques, they protect you from script kiddies, but not the determined adversary.
