---
title: "Thoughts on WWDC 2020"
date: 2020-06-27T12:00:00-05:00
tags: ["apple", "wwdc"]
---

The developer community was interested to see how a remote WWDC would go and I think the general consensus is that this is how the conference should go from now on. By having the session be prerecorded, they are much more polished and the transcripts and sample code are available the same day. This has also allowed for sessions that don't have to be 50 minutes in length, allowing for some quick videos on best practices or updates on smaller frameworks (very helpful). I think the biggest improvement was the keynote/platforms state of the union since the prerecording meant that all of the demos could be optimized and you didn't have to waste time transitioning to different speakers and no time wasted for audience reaction shots (although fewer Craig jokes make me sad). 

The labs this year were WebEx appointments and I think this should be used throughout the year as it allows people to schedule time for get their issues looked at rather than waiting in a line and maybe getting someone to look at your problem (also, no need to worry about missing sessions live since they are already available in video form).

Other than the setup, this year's content was largely focused on the Mac (for once) and there was a lot. I was originally thinking this year would be muted since there were large announcements last year and due to the pandemic, but that didn't stop Apple.

## tvOS

Similarly to last year, there aren't a lot of big changes to tvOS as it is already a mature platform. Instead of getting call notifications, we now get doorbell notifications via HomeKit. I did check to see how many doorbells are compatible with HomeKit and there are 3, but, Ring/Nest/August are not on the list. Due to this, maybe those manufacturers will add HomeKit support or support the new consortium protocol.

## watchOS

Again, a mature platform so there aren't any earth-shattering changes (other than maybe sleep tracking). watchOS gains new workouts support (changing the Activity app to the Fitness app), custom watch face sharing (a long requested feature), and handwashing tracking to make sure you wash your hands for the recommended amount of time. 

## iOS

Springboard got a redesign for iPadOS last year, so now the iPhone's version gets a redesign this year. Now you can add widgets directly to the home screen as well as combine widgets into stacks so that you can have multiple widgets in the same screen space. Apple also added what they are calling the App Library (Android's app drawer) so that you can hide home screen pages and still have the app installed/searchable without being visible normally.

The changes I am most excited for are the changes in Siri and Phone/FaceTime calls no longer taking up the full screen, rather they are now more manageable interactions floating over the current content.

Once change that is coming earlier (in iOS 13.6) is CarKey. CarKey allows you to unlock/lock & start your car with your phone as well as grant guest access to your car. We'll see if this takes off as you can only get it right now on new high-end BMWs and car manufacturers don't add new technology quickly (not to mention car designs take a long time).

One thing that should become popular are App Clips (Android Instant Apps). These are < 10MB little apps that are meant to help users complete actions (e.g. parking) without requiring the full app. NFC powers these interactions and the user can install the full app later if they want. Once we can all go back outside, I suspect that a few kinds of businesses will start providing these as a convenience.

## iPadOS

Surprisingly, iPadOS does not get widgets on the home screen nor the App Library (thus creating the first major deviation in SpringBoard other than multi-tasking). However, iPadOS does get Scribble (Apple's handwriting OCR functionality). With Scribble, users can use the Apple Pencil and write into any native text field and the contents can get transformed into non-handwritten text. This can also be used to transform shapes and copy handwritten text from one area of the system to another.

## macOS

macOS got promoted to the next major revision: 11. This is due to a major visual overhaul (some good, some bad) and support for running on arm64 chips instead of x86_64. A lot of the OS bundled apps are now Catalyst-based or are updated to take advantage of new Catalyst features and Macs running on arm64 (Apple Silicon) can now run iPhone and iPad apps unmodified.

There are some changes that still haven't taken effect yet like the removal of the old shells & the scripting languages as well as the complete removal of kernel extensions. Since the Intel -> arm transition will take about two years according to Apple, I wonder if the next release after 11.0 (Big Sur) will remove them or will it be once no Intel Macs are ever to be released again as it is surprising to see Apple make these technologies work on arm-based Macs when they have been removing as much as possible to get to those Macs as quick as possible.

## Xcode

Xcode didn't get many big features this year: just a UI update to align with macOS, updated Code Completion, and an Organizer that is much more useful. Most of the work is being spent on ensuring the entire toolchain works on arm. 

## Safari

Safari got some enhancements that will make it more usable this year. After killing off their extension API, Safari now supports the extension API used by Firefox and Chrome, but with restrictions in place that will continue to prevent some extensions from coming back to Safari (NoScript, uBlock, etc). However, the start screen is now customizable and tabs have previews if you hover over them, and Safari will now show you what it blocked from tracking you. I still content that they need to release more frequently to get to feature parity with Chromium-based browsers and Firefox.

## Catalyst & SwiftUI

Catalyst got more APIs and refinements to work better (like removing the 77% scaling). Since iOS and iPadOS apps can now run unmodified on arm-based Macs, the recommendation now is to use Catalyst to bring a more native experience for areas of apps that could be modified to be a bit more Mac like, but now it isn't required.

SwiftUI on the other hand is getting a lot of improvements that is positioning it to begin phasing out AppKit and UIKit proper. There are new app life cycle APIs to replace the traditional delegate based approaches and many of the performance issues reported in last year's release are now resolved with new lazily loaded versions of those APIs. Additional concepts regarding lists and navigation are now supported and SwiftUI views can now be shared in Swift Packages and are what power the new widgets due to their lightweighted-ness and the new serialization format (Apple Archive).

## Apple Silicon

Obviously, transitioning from x86_64 to arm64 is an enormous undertaking and Apple has indicated that the performance and power consumption they are expecting will handily beat the Intel chips. To help in this transition both Universal apps and Rosetta are back. Universal apps will now include arm64 slices in addition to x86 & x86_64 slices and Rosetta 2 will attempt to AOT compile the translation upon install but will also support JIT compilation for various apps. Based on the demos, Rosetta is performing very well and will definitely ease the gap time while developers get their apps ported over.

The big question is how long are Intel Macs going to be supported once Apple has converted all of their lines (that they want to) over to their own SoCs. The PPC transition only allowed a single year of OS updates to those Macs before they became obsolete. Apple is currently saying "years", but that technically could be 2.

---

With Swift 5.3 getting performance improvements and now being below Foundation (allowing for low-level daemons to be Swift-based) and SwiftUI getting ready to replace AppKit and UIKit, it should be interesting to see how long the Objective-C runtime will remain current as it is the only way (right now) to get C++ into the mix as Swift only supports bridging into C. What I suspect is that most of the engineering efforts have been spent porting over to arm64, but once that is done, work can be done to rewrite the OS by removing all of the C++ foundations and replacing the C code with Swift.
