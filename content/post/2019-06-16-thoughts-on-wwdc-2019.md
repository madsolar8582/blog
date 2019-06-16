---
title: "Thoughts on WWDC 2019"
subtitle: ""
date: 2019-06-16T08:00:00-05:00
tags: ["apple", "wwdc"]
---

Oh boy, this year sure was chock full of stuff. At WWDC, Apple revealed their plans for a unified development platform, elevated their newer platforms to first class status, and revealed some long overdue hardware. Putting all of that together, there was a lot to unpack and even more to learn with the announcement of SwiftUI, Catalyst, Combine, and a slew of other new frameworks (most being Swift only).

## tvOS

tvOS isn't useless anymore with the announcement that tvOS 13 will add support for Xbox One and PS4 game controllers. This should be a big boon for Apple Arcade as well as non-participating developers who want to release a game on Apple TV but don't want to use the Siri remote as a controller nor force people to buy a MFi certified controller. Additionally, tvOS getting multi-user support is nice for content recommendations, but I'm not so sure if I like the homescreen now being overlaid on top of auto-playing content.

## watchOS

As hinted at with App Store Connect backend updates, watchOS can now have dedicated apps. As a result, the dreaded WatchKit framework is now dead. It should be interesting to see how many apps come back to the watch now that there is a dedicated app store and better development tools. On the heath front, I am also interested to see how the Noise app plays out as I think a lot of people don't realize how noisy their environment is.

## iOS

So, iPadOS is a thing now (although it is purely for marketing). The SpringBoard redesign on iPad sure does make it more useful, although the first beta is extremely buggy. Dark mode is definitely a welcome feature and the new visual design language is helping to bring back contrast and context to the UI (less flat). Apple is still focusing on performance with faster app launches as well as smaller downloads for initial installs and updates. The new APIs for scenes (multitasking) will take some getting used to, but I think overall this is a good release that sets them up for even more desktop-like features in the future.

## macOS

The only feature I am super excited for is Sidecar. Being able to mirror or extend your display to an iPad and then have the iPad be able to interact with the screen elements is fantastic. On the other hand, there are some things afoot. In 10.15, the system folder is now a separate APFS volume that is read-only (thus moving user data to its own volume). This is a security architecture decision to help stabilize macOS, but there is a problem: embedded scripting languages. Since macOS installed Ruby, Python, and Perl in the system directory, those directories still have to be writable since that is how gems and packages are installed for use. So, Apple announced that those scripting languages will be removed from macOS in a future release. This will make it interesting for developers as applications sometimes depend on these being installed for the applications to work, but also ensure that there is a consistent version to depend on. This will also impact development setups as Homebrew is written in Ruby and RVM uses Homebrew to install packages to install Ruby (chicken/egg problem). 

Kernel extensions are also on the way out. With the introduction of DriverKit, Apple says that future versions of macOS will not longer support kernel extensions, so developers are forced to move their extensions into userspace. I wonder if this will allow Nvidia back onto the Mac (Apple is currently refusing to sign drivers for macOS 10.14). This also presents a problem for Docker as it relies on VirtualBox to deploy the images.

Another change coming is the move to zsh as the default shell. This continues Apple's approach of removing software (see macOS Server) that has newer versions (i.e. Bash), but cannot be used as the license isn't Apache nor MIT. 

Overall, I see these changes as a push for more security as 10.15 requires notarization, apps (sandboxed or not) can opt into the hardened runtime, and now the system is slimming down. However, this does bring up the fact that if the system is slimmed down enough so that it isn't that different from the iOS stack, ARM based Macs could function with just a little more work from Apple.

## Catalyst

Now that Catalyst (née Marzipan) is available to developers, Twitter has been full of happy developers porting their apps over with it taking about 1 to 3 days total (also, Twitter themselves are bringing the iOS client over to the Mac). I think the biggest pain point right now are subscriptions not being able to be shared between iOS and Catalyst apps. There is also room for improvement on certain AppKit-like functionality, but those appear to be resolvable using public APIs, just in an odd way (easily fixable by Apple).

## SwiftUI

SwiftUI (née Amber) was built to replace WatchKit, but then Apple decided to bring it to all of Apple's platforms. It provides a declarative way of designing and implementing UI. The live preview is also huge as you can put in sample data and multiple configurations, to see how the application will look. The fun part, though, is that the UI composition can be done via drag and drop, meaning that in the future, we could get an Apple IDE on iPadOS. A unified UI tool and implementation means that there should not be problems like AppKit and UIKit in the future which is great for developers.

## Combine

Combine is one of those things I am still trying to process as it is very foreign to me. Apple says that this is a way of unifying asynchronous programming paradigms (callbacks, delegates, etc) and I can see it in their sample code, but I don't see what is wrong with GCD. To me, GCD is the simplest way of implementing concurrency and block-based callbacks are easy to implement. I guess the benefit with Combine is the built-in data transformations and error handling. Since this is being used by Apple on a lot of Foundation classes, this does appear to be the way Apple wants code to work, so it'll become mandatory in the future.

## Xcode

Xcode got some good updates this year with the editor being the primary focus. Being able to take control of how the editor functions and what it is presenting at all times is good, but the mini-map is super helpful (although it is a bit weird to see killed plugins being brought back as first-class features). I'm just happy that the simulator is very performant now that it is written in Metal and uses way less CPU (makes using multiple instances for testing way better).

---

This year was a great WWDC as Apple has clearly communicated the future of their platforms and given developers the tools that they need to be successful on those platforms. The only remaining question is when will the Mac switch to ARM occur and how long will they keep the x86_64 runtime around?
