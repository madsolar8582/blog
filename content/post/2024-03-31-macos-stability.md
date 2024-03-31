---
title: "macOS Stability"
subtitle: ""
date: 2024-03-31T17:00:00-05:00
tags: ["macos"]
---

The recent macOS 14.4 [kerfuffle](https://arstechnica.com/gadgets/2024/03/usb-hubs-printers-java-and-more-seemingly-broken-by-macos-14-4-update/) has worryingly brought back to light the [Core Rot](https://macperformanceguide.com/autoTopic.html?dglyTP=Apple+Core+Rot) theory where users of macOS highlight problems in macOS that should not exist and/or experience degrading issues that lead to a [death by 1000 cuts](https://en.wikipedia.org/wiki/Creeping_normality).

In this case, it would appear from the outside that QA missed common use cases or was not performed at all as standard uses of macOS broke. Some of these breakages seem to be related to the [security patches](https://support.apple.com/en-us/HT214084) which are not made available for public testing in the betas (for valid security reasons). The problem with this is that for normal end users with auto update turned on, they get a broken update and corporate I.T. departments have to update their MDM after clearing a beta release for consumption to delay the patch until Apple fixes the issue with an understanding that they only get 90 days to delay the update as that is the maximum Apple allows. In the Microsoft world, I.T. departments have more control over the updates, but Apple's update strategy is not as flexible. Rather than each update being its own package to install, and thus being able to be easily rolled back, the [Signed System Volume](https://support.apple.com/guide/mac-help/what-is-a-signed-system-volume-mchl0f9af76f/mac) does not allow for this.

You can, of course, try to roll back via Time Machine, but users have [experienced mixed success](https://eclecticlight.co/2023/07/15/why-cant-you-just-roll-back-from-a-bad-macos-update/) with this as macOS updates also update firmware which may not be compatible with older system volumes. Therefore, since macOS releases don't have an "oops, revert" strategy, one would expect that greater care would be placed on these updates.

While it is not feasible given time constraints to test every possible permutation of every program, especially with regression testing something as complex as an operating system, you should still have decent enough test coverage to ensure that the 90% case is unaffected (yes, 10% of millions of users is still a large number). However, we know that while Mac hardware is selling well, [not a lot of attention](https://arstechnica.com/gadgets/2016/12/report-apples-mac-team-is-getting-a-lot-less-attention-than-before/) is put on the software. Apple's main focus is the iPhone, followed now by the Vision Pro since it is a new platform, then the iPad, then the Mac, then Services, then everything else (wearables/home/tv). This is demonstrable by the fact that major apps and features launch first on the iPhone and then make it to the other platforms.

Competition is supposed to help address this, but with Apple Silicon, you can only run macOS and Apple is further locking down how the platform can be used. We can also look at the yearly release cycle and how promotions and compensations are tied to launching new features instead of fixing bugs, improving performance, and being proactive at preventing issues instead of being visible in "fire fighting" mode.

Many have been asking Apple to slow down, either by abandoning the yearly release cycle and/or by backing off on features and focusing on fixing issues and fully patching them backwards to older releases. This is likely never to happen with Apple's current leadership but one can hope.

---

I fully believe that once Microsoft's deal with Qualcomm expires, running Windows on Apple Silicon will not only be the best Windows experience, but a lot of macOS users will make the jump as Microsoft still seems to be more invested in desktop computing than Apple.
