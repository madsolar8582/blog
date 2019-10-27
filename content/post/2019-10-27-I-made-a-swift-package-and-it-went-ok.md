---
title: "I Made a Swift Package and it went OK"
subtitle: ""
date: 2019-10-27T07:00:00-05:00
tags: ["spm", "swift package"]
---

I wanted to adopt some new features introduced in Xcode 11, one of which is adding a Swift package to an existing project and then distributing it via GitHub. So, I fired up Xcode and went to work. It went OK.

Apple's [documentation](https://developer.apple.com/documentation/xcode/creating_a_swift_package_with_xcode) does a good job of explaining the basic structure and what to configure, however, I found that the SPM documentation on [GitHub](https://github.com/apple/swift-package-manager/blob/master/Documentation/Usage.md) was far more verbose and helpful. 

Defining the package was super easy, but as soon as I started configuring what it was supposed to do, that's where the problems started. I suppose it was partially my fault, but I had assumed that packages were based off of a DSL, like a [Gem](https://guides.rubygems.org/specification-reference/) or [Podspec](https://guides.cocoapods.org/syntax/podspec.html), however, I quickly learned that was not the case. Rather, everything is a Swift function, so if things aren't in the exact right order, the package will not build. This is annoying as DSLs can put things in arbitrary order as long as the items are in the right section, but not with Swift packages. This brings me to my next issue, Xcode didn't help me figure this out at all, rather, I had to use `swift build` in Terminal to get error messages and fix them until the package built. This is very disappointing as Xcode is supposed to have autocomplete and Swift package integration to help in these situations.

---

After about 40 minutes of figuring out things that were not explained and getting things into the right order, I was able to successfully define a package and import it into another project for testing. The automatic module mapping and build configurations were very helpful and the built in semantic versioning upon package inclusion was very slick. Hopefully, newer versions of Xcode get a better understanding of Swift packages so that you don't have to use the command line to troubleshoot packages.
