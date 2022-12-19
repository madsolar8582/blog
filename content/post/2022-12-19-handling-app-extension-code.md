---
title: "Handling App Extension Code"
subtitle: ""
date: 2022-12-19T08:40:00-05:00
tags: ["app extension", "preprocessor"]
---

Over the past couple of years, Apple has been slowly increasing the number and kinds of App Extensions one can associate with their applications. As such, reusing code between a main application and an extension is somewhat desireable.

At the most basic level, adding a logic branch to a path is crude, but effective:
```obj-c
if ([NSBundle.mainBundle.bundlePath hasSuffix:@".appex"]) {
  // Do app extension stuff
}
else {
  // Do main app stuff
}
```
Unfortunately, this does not handle the case where the main application branch references APIs that are unavailable in app extensions. Apple only provides a way of handling the case where you tell a developer not to call an API (and solves the linkage problem): `NS_EXTENSION_UNAVAILABLE_IOS("...")` However, Apple does not offer a target conditional for compilation which is what most situations need as the macro cannot be applied to classes, only properties and methods.

Luckily, you can create your own target conditional. I stumbled upon [an article](https://davedelong.com/blog/2019/04/09/conditional-compilation-part-3/) by [Dave DeLong](https://mastodon.social/@davedelong) which goes into depth as to how the compiler uses the `APPLICATION_EXTENSION_API_ONLY` flag and how it can be reappropriated.

To summarize the article briefly, you can use the evaluation semantics of the preprocessor to create your own flag to control compilation in your source code:
```
_APP_EXTENSION_YES = 1
_APP_EXTENSION_NO = 0
APP_EXTENSION_GCC = BUILDING_FOR_APP_EXTENSION=$(_APP_EXTENSION_$(APPLICATION_EXTENSION_API_ONLY))
GCC_PREPROCESSOR_DEFINITIONS = $(inherited) $(APP_EXTENSION_GCC)
```

Then, in code, you can do conditional compilation:
```obj-c
#if BUILDING_FOR_APP_EXTENSION
  // Do app extension stuff
#else
  // Do main app stuff
#endif
```
---

Hopefully this helps handling code reuse between applications and extensions. I am a bit disappointed though that Apple _still_ hasn't provided an official way of marking conditional compilation given the number (and now ubiquity) of app extensions.
