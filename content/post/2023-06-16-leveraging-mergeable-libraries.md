---
title: "Leveraging Mergeable Libraries"
subtitle: ""
date: 2023-06-16T19:00:00-05:00
tags: ["xcode", "linker"]
---

When building your application with binary dependencies, you are either statically or dynamically linking them to your app. When statically linking, the symbols used are copied directly into the binary executable and discards the unused symbols. This increases the compile time of the app and the size of the binary executable, although some additional space savings (overall) can be had by using Link Time Optimization (LTO) and by using the `-Os` optimization level. On the other hand, dynamic libraries are copied wholesale into the app package (note: not the binary) and then the dynamic linker and the kernel map the referenced symbols into the app's memory space upon launch. This has the effect of slowing time the time it takes for the app to become responsive (i.e. usable by the end user) and, depending on the size of the unused symbols, can increase the overall size of the application by a lot.

As you can see, each strategy has its pros and cons and ultimately, those tradeoffs are manageable depending on how many dependencies you have of each type and how large they are. To provide a happy medium, Xcode 15 and the new linker introduces [Mergable Libraries](https://developer.apple.com/documentation/xcode/configuring-your-project-to-use-mergeable-libraries). What this allows you to do is leverage dynamic libraries as if they were static libraries. By doing so, you can reduce the overall size of your application while also improving the launch time by deferring the work to compile time when you were previously unable to.

In order for this to work, framework owners must set `MERGED_BINARY_TYPE` to automatic (suitable for most cases) or manual if you want finer grained control within your project. With that set, the linker will build the target as mergeable if it is able (unsupported architectures, cyclical dependencies, & resource bundle hooks may require additional configuration and workarounds). Also, debug builds do not use dyld. 

Assuming that you meet the criteria and your dependencies are marked as mergeable, you should be good to go.

---

One thing to keep in mind is that while source and line information is preserved in backtraces, the path will be different.
