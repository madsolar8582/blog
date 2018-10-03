---
title: "Architectures Galore"
subtitle: ""
date: 2018-10-02T19:00:00-05:00
tags: ["architectures", "bitcode", "arm64e", "arm64_32"]
---

While some developers might be pining for the days when everything was written in [assembly](https://en.wikipedia.org/wiki/Assembly_language), I am not one of those people. CPU [instruction sets](https://en.wikipedia.org/wiki/Instruction_set_architecture) are very complicated these days and there are a lot of revisions based on what [chipset](https://en.wikipedia.org/wiki/Chipset) you might be running on, which is why [compilers](https://en.wikipedia.org/wiki/Compiler) are awesome. If we take a look at what [architectures](https://en.wikipedia.org/wiki/Computer_architecture) the Apple platforms are using (and have supported), you can see why writing everything in assembly is folly:

### Mac

- i386
- x86_64

### iOS

- armv6
- armv7
- armv7s
- arm64

### tvOS

- arm64

### watchOS

- armv7k

That's a lot of architectures. Say for example, you created a reusable framework that supported each of those platforms. Prior to iOS 12 and watchOS 5, when packaging the framework for release, you got a reprieve as 10.13+ is essentially [64-bit](https://en.wikipedia.org/wiki/64-bit_computing) only, iOS 11+ is 64-bit only (but you need to include simulator architectures for the Mac), tvOS is 64-bit only (but you need to include simulator architectures for the Mac), and watchOS is [32-bit](https://en.wikipedia.org/wiki/32-bit) (but you need to include simulator architectures for the Mac). Thus, your [fat binaries](https://en.wikipedia.org/wiki/Fat_binary) would be a bit bigger during distribution to other developers, but the App Store submission process would remove the unneeded architectures. Apple saw this problem and introduced a solution: [bitcode](https://lowlevelbits.org/bitcode-demystified/). Bitcode creates a different format of [mach-o](https://en.wikipedia.org/wiki/Mach-O) binaries that Apple then uses to create optimized versions of the end product (the application) once submitted to the App Store. This leads to smaller apps for end users as the application (and optionally its resources) are specifically for their kind of device. Coincidentally, Apple made bitcode required for tvOS and watchOS applications, so some developers opted to enable bitcode for their iOS apps as well.

The amount of work required to support something like bitcode for a compiler is a lot and it paid off with the Series 4 Apple Watch. The new Series 4 uses an arm64_32 CPU (64-bit instructions with 32-bit pointers). Rather than have apps run in a compatibility mode until developers updated their applications for the new architecture, bitcode allowed Apple to create the new versions of these applications for the new device without needing anything from the apps' developers. This is a big win as some developers never added arm64 support for their applications and those applications cannot run on iOS 11+. This lead to users seeing prompts from iOS letting them know that they were going to lose access to that application, ut watchOS users never have to go through that process (yay). Going back to the previous paragraph, now you must add arm64_32 to the list of architectures for your watchOS target.

Ok, let's talk about iOS. The new iPhones (and presumably the new iPad Pros coming later this month) now have an arm64e CPU. Those CPUs support the ARMv8.3 specification which includes [Pointer Authentication Code](https://lwn.net/Articles/718888/). PAC is a security enhancement to compliment [Data Execution Prevention](https://en.wikipedia.org/wiki/Executable_space_protection) (DEP) and [Address Space Layout Randomization](https://en.wikipedia.org/wiki/Address_space_layout_randomization) (ASLR) that provides integrity to pointers to help prevent malicious code execution by refusing to execute an instruction if the integrity check fails. Right now, iOS apps are running in a pseudo compatibility mode on the new hardware as apps need to be recompiled for arm64e to get the full benefit of the new CPU features. Now, it may be possible for bitcode enabled apps to be recompiled by Apple, but I suspect that that may not be the case. Xcode 10.1 ß2 introduced arm64e as an architecture target, but it is considered optional as App Store Connect will not allow submissions with this architecture (it is stripped from the binary). My guess is that as of March 2019, apps will need to include arm64e as a required architecture (another architecture to add to the list for your universal binary) and _maybe_ Apple will start requiring bitcode for iOS apps.

---

With all of the new architectures being added with the latest OS releases (and the ever more likely ARM powered Mac), leveraging bitcode seems like a good idea to stay on top of all of the changes Apple is making with their hardware. We'll have to see if there will be any other changes in Xcode that might make this easier for developers as writing scripts to create fat binaries using [lipo](https://ss64.com/osx/lipo.html) and having to manage the bcsymbolmaps and [dSYM](https://lldb.llvm.org/symbols.html)s is quite cumbersome.
