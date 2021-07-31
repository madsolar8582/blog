---
title: "Using Multi-Platform Projects"
date: 2021-07-31T18:00:00-05:00
tags: ["xcode"]
---

If you create a reusable library for Apple platforms, you know that managing it can be a bit cumbersome. In Xcode, you have multiple targets and test targets (iOS/Catalyst, macOS, watchOS, & tvOS) and then you need to keep all of the Xcode configuration consistent between them. 

One way of handling this is to create a few [xcconfig](https://nshipster.com/xcconfig/) files, but this has its drawbacks too. You can create a base config file for each target to inherit from, then create individual ones for the specific platforms. The problem is that we've continued to duplicate more config for each target and these files do not come with Xcode's built in help for making sure that they are correct. Luckily, Apple has provided a solution for this problem in Xcode 13.

New in Xcode 13 is the Multi-Platform Framework option. You can create a new target using this template or modify an existing target to become one. Essentially, this allows a framework target to be used on any platform without needing to define platform specific targets. This works by setting the _Supported Platforms_ to `Any` and _Allow Multi-Platform Builds_ to `Yes` and now you can build to whatever platform you want. Xcode even includes the ability to manage what files are included/compiled per platform as well as what frameworks are linked without having to create `xcconfig` files. Note: older versions of Xcode will not understand this, so you need to set your Xcode project file compatibility level to Xcode 13 or newer.

---

Multi-Platform support in Xcode makes maintaining libraries much easier now that there is a way of managing all of the platforms you support from a single entry point. Combine this with SPM and it becomes stupidly simple to maintain and release a reusable library across all of Apple's platforms.
