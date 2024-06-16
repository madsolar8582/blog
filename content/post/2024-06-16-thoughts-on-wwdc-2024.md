---
title: "Thoughts on WWDC 2024"
subtitle: ""
date: 2024-06-16T14:30:00-05:00
tags: ["apple", "wwdc"]
---

As expected, the big reveal was Apple Intelligence ([AI](https://machinelearning.apple.com/research/introducing-apple-foundation-models)). This allows users to access ML capabilities on device as well as determine if what was requested needs to go off device for server-side processing. This allows Apple to plug and play different AI models (which OpenAI being the first) while also ensuring privacy and security using a complex implementation of privacy and security techniques dubbed Private Cloud Compute ([PCC](https://security.apple.com/blog/private-cloud-compute/)).

As a result, the [AppIntents framework](https://developer.apple.com/documentation/appintents/appintent?changes=latest_minor) got yet another overhaul to allow for developers to advertise how to integrate into the newly upgraded Siri for Apple Intelligence capabilities.

Other than that and the Swift 6 uplift, there aren't many major changes (besides the new floating tab bar in iPadOS), which is a nice breath of fresh air given the typical rapid pace of deprecations, changes, and new features. Rather, it appears Apple was primarily focused on Apple Intelligence and continued work on visionOS to make it more feature pair with the rest of Apple's operating systems.

That's not to say that we didn't get anything. We can now do nested virtualization on macOS Sequoia including being able to sign into iCloud, Xcode 16 has a coding copilot on macOS Sequoia and with 16GB of RAM (which makes it even more distasteful that Apple sells "pro" machines starting with 8GB of RAM), window tiling on macOS Sequoia, and iPhone screen mirroring.

Consumer feature wise, I don't think I'll end up using most of the announced AI features, but I do think Dark Mode app icons was a good idea. However, the biggest problem is that iPadOS 18 does nothing to help solve the core problem of having too much hardware and too simple software. At this point in time, I think that I need to move on from iPadOS as I'm not a creative professional and I want my computing device to do more.

---

Like last year, many tent-pole features are not arriving for the late Summer initial release. However, a larger than normal amount are [being deferred](https://9to5mac.com/2024/06/12/here-are-all-the-new-features-that-wont-arrive-until-ios-181-or-later/) to subsequent releases. Part of this, I think, was due to the AI shift in addition to needing to throw a lot of resources at visionOS for its launch. That aside though, a yearly release cycle combined with the ever growing number of platforms is really putting a strain on Apple resources. To me, Apple needs to slow down and target 18-22 months worth of development time so that features can bake and bugs can be fixed. 

Also, I think this is the last year for Intel-based Macs. All the AI features require Apple Silicon. I suspect that once Apple reveals the M4 Max and Ultras, we'll see a change in Xcode Cloud too which, to me, will indicate that Apple has completed the transition.
