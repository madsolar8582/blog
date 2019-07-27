---
title: "Distributing and Consuming Dependencies"
subtitle: ""
date: 2019-07-27T07:00:00-05:00
tags: ["cocoapods", "carthage", "spm"]
---

One of the benefits of a healthy platform is the availability of high quality open source projects that compliment it. However, without an official method of distribution, the community was left to solve this problem. Two major methods emerged: CocoaPods & Carthage.

## CocoaPods

[CocoaPods](https://cocoapods.org/) is a Ruby gem that allows you to create a CocoaPod. Each pod defines integratable targets via source files and/or binary files, resources, and other dependencies in a YAML based podspec. The pod can then be published to the master spec repository, or, remain in some publicly accessible source control system (e.g. GitHub) and then referenced by others.

As of this writing, this is the most common method of dependency distribution and consumption. However, there are some side effects to this approach. The primary downside is that CocoaPods's integration method requires it to rewrite Xcode project and workspace files and reconfigure aspects of the build phase to link and compile all of the necessary components. This can go wrong when Apple decides to make changes to the build system or Xcode which can cause the project to no longer build. This also introduces the possibility for subtile bugs to be introduced due to misconfigured builds or wrong assumptions about how items should be integrated.

## Carthage

[Carthage](https://github.com/Carthage/Carthage) is a Swift project that builds frameworks for binary distribution. Projects create a Cartfile which includes all of the desired dependencies and then Carthage builds each one of the requested frameworks. Once the dependencies are built, you just add them to Xcode.

The primary difference here is that Carthage does not modify and Xcode files or settings, which may be less convenient, but more stable. The Carthage developers also like to highlight that their approach is decentralized as there is no master repository.

## Swift Package Manager

With the rise of Swift, Apple itself decided that they should have an official way of distributing Swift code so they created [Swift Package Manager](https://swift.org/package-manager/) to replace both community solutions. Although the name implies Swift, Swift packages may contain C, C++, and Objective-C code with or without Swift code, so the utility of the community solutions is preserved. A Swift package is similar to a podspec where the targets and dependencies are defined (with optional platform and Swift language requirements), but, as of this writing, there is no method for defining resource bundles. Another limitation is that Swift packages work with macOS and linux out of the box, but Xcode 11 is required to enable iOS, watchOS, and tvOS integration.

---

Limitations aside, the native integration in Xcode increases the power of Swift packages as well as maintains sane build system settings. Since Apple is now evangelizing Swift packages, I forsee Swift packages rising in popularity over the next few years. However, I think this growth will be slowed if Apple does not move *swiftly* in stabilizing the functionality of the package manager so that it does not necessarily require Xcode and to allow for resource bundles.
