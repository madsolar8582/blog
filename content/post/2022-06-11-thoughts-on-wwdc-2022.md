---
title: "Thoughts on WWDC 2022"
subtitle: ""
date: 2022-06-11T08:55:00-05:00
tags: ["apple", "wwdc"]
---

WWDC this year was a hybrid event with media and select developers watching the pre-recorded keynote from Apple Park (and getting access to the new Developer Center) while the rest of us watched online. Tours of the Developer Center showed off some really nice conference rooms, workspaces, and a theater. Overall, seems like Apple is putting money down to try to win over developer perception as the current looming legal and regulatory pressure continue to eat away at developer trust in Apple.

## iOS

This year Apple took a page out of Google's playbook and revealed a lot of personal customization options for the iPhone (since the 7th gen iPod Touch is discontinued and isn't supported anymore, now only iPhones run iOS). The lock screen now supports a variety of fonts, photo styles, and widgets (indicative of an always-on display in new models this fall). Later this year, Apple will support widgets that can update with live content (e.g. scores or video). Additionally, the lock screen supports a few different types of notification presentation styles to allow for the content focus to be the user's background photo or widgets. Unfortunately, these customizations aren't coming to the iPad (this year), but perhaps a modified set of capabilities will arrive next year.

Focus mode was not overhauled and is instead getting deeper OS integration. This also means that you can setup different lock screens for different focus modes. Apps can now also get focus mode status and filter out content directly in the app rather than apps being allow/deny listed.

iMessage now supports editing and unsending (on iOS 16+) which is definitely a nice feature to have (looking at you Twitter...). SharePlay is also getting extended and deeper OS integration with the expansion of Shared with You to support SharePlay directly in iMessage and new collaboration features in notes, the iWork apps, and Safari Tab Groups.

If you have a modern CPU, the neural engine now supports capturing text out of videos (an upgrade over just images) and the text OCR system as a whole now supports quick actions (e.g. currency conversion). 

Lastly, Apple is getting similar features to Google GBoard where Siri dictation now supports dictating and keyboard entry combined.

## iPadOS

iPadOS gained some features and lost a key one this year. The loss is that iPads can no longer be home hubs as of iPadOS 16 (now only Apple TV devices). The gains are movements into increasing the "desktopness" of iPadOS by introducing a new multi-tasking windowing mode (Stage Manager), full support for external displays as secondary displays, resolution scaling to increase the pixel density of the iPad itself, and (gasp) virtual memory. This is great for users who want to use their iPads as true PC replacements, but I don't think we're quite there yet as free-form windowing would provide the most power. However, one could argue that at that point, just get a laptop.

On the app front, Weather makes its way to the iPad (finally), but the calculator is still missing.

## macOS

Spotlight got an overhaul to make it more useful by integrating quick actions and shortcuts into the results and Stage Manager made its way to the Mac as well for people overwhelmed by the traditional desktop experience and don't find Spaces and Mission Control helpful. Additionally, the iOS Clock and Weather apps make their way to macOS thanks to Mac Catalyst.

Continuity now supports FaceTime handoff and macOS now natively can use iPhone cameras as webcams (many 3rd party apps were sherlocked).

However, the biggest change is probably the redesign of System Preferences to System Settings (written in SwiftUI). Initial feedback has not been positive and Craig Federighi (Hair Force One) has stated that Apple is still working on fine tuning the details prior to the GA release this fall. I'm not so certain that this new redesign is going to stick as I foresee constant changes throughout macOS 13 (Ventura)'s lifespan given the fact that Apple spent multiple releases changing and then reverting Safari's redesign. For more info on why the changes are disliked, read [part 1](https://lapcatsoftware.com/articles/SystemSettings.html) and [part 2](https://lapcatsoftware.com/articles/SystemSettings2.html) of Jeff Johnson's blog regarding the topic.

## tvOS

tvOS didn't get any attention in the keynote, but we did get confirmation that Matter is arriving later this year and that the Home app on all platforms has been redesigned from the ground up to better support this and improve the user experience. 

## watchOS

While watchOS did get some screen time in the keynote, most of the updates were pretty minor. However, the focus on health continues and the Health app now supports medication reminders and logging as well as notifications about potential harmful interactions. Also, if you sleep with your watch on, watchOS 9 now allows for detailed sleep stage tracking to help you track the quality of your bed rest over time. Lastly, AFib detection got a boost by now being able to track your atrial fibrillation over time to let you know how often you experience arrhythmia.

## Xcode

Apple spent a lot of time on Xcode and its developer toolchain this year.

First up is a smaller Xcode. Now, Xcode is only bundled with the macOS and iOS SDKs and simulators to reduce the overall size of the app (especially noticeable on download and unxip). Additionally, Interface Builder canvas operations are faster.

Compilation and linking got performance boosts as well. The Swift Driver was rewritten for better performance and parallelization. For pure Swift projects, the driver can now emit and link modules eagerly, eliminating some wait time during compilation.

For ld64, the linker itself has improved static and dynamic linking. Content is now copied in parallel, `LINKEDIT` is now built in parallel, hashes are now computed in parallel, UUID computation now uses SHA256 due to hardware acceleration, and exports now use an optimized trie algorithm. Chained fixups are now used to bridge the `DATA` and `LINKEDIT` portions of the macho-o file and on the newer OSes, the kernel is now responsible for the fixups on page-in which reduces dirty memory and launch time (does not apply to `dlopen`). 

Over in Objective-C land, ARC received some compilation improvements by reducing some instructions needed for the inserted calls to `retain`/`release`. Additionally, there is a new option to disable the compiler creating selector stubs with `objc_msgSend` as part of the stub. Rather, the `-objc_small_stubs` linker options tells the linker to generate selector stubs without the message send stub inline (which improves binary size).

Lastly, there are some updates to the debugging tools. The Memory Graph Debugger now is more accurate about what objects are referring to your selected object so that you can get a better picture of incoming references and Instruments gained a new Swift Concurrency template to allow for visualization of the new concurrency system and easier debugging.

The challenge though is will Xcode 14 remain performant and stable throughout its life cycle. Xcode 13 had some serious regressions and performance slow downs (some not even fixed). A key highlight for Xcode 14 is its new smarter and faster autocomplete suggestions and last year Apple touted better syntax highlighting but that broke multiple times so we'll see if it remains working.

## Networking

Apple has introduced new beta features to their OSes this year now that HTTP/3 and QUIC have been finalized.

DNSSEC is now supported on `NSURLSession` and in Network.framework. This comes alongside support for the Discovery of Designated Resolvers (DDR) protocol which allows networks to advertise DNS-Over-HTTPS (DOH) or DNS-Over-TLS (DOT) encrypted DNS resolvers.

Apple is also working with the IETF on a new protocol to improve network throughput by replacing the Explicit Congestion Notification (ECN) protocol with a new implementation called Low Latency, Low Loss, Scalable Throughput (L4S). This new protocol only works with HTTP/3 and QUIC (with a recommendation of IPv6) and Apple is looking for feedback on how well it works. Early data suggests that Round Trip Time (RTT) is significantly reduced and that throughput with the updated queuing is high (although I think this really only impacts ISPs and IXPs as consumer routers aren't incentivized to implement this).

## Security

Apple spent time highlighting two security features this year: Passkeys and Rapid Security Response.

Passkeys are cryptographic key pairs that users can use in lieu of passwords (making them resistent to a whole bunch of attacks). The implementation is based on WebAuthn and requires updated server infrastructure so the roll out might take a while, but the FIDO Alliance is on its way to permanently replacing passwords for good.

On the OS side of things, Apple has updated its OSes to support their new Rapid Security Response implementation which allows Apple to patch the system without having to go through a full update process. This is a tremendous improvement as the recent WebKit security updates have required a full 3GB macOS patch (compared to 100MB on iOS) due to the signed and sealed system volume. This should also allow for faster security updates as the certification process for OS updates can be simplified.

---

Overall, this year was a solid year for developer tools and consumer features. What was interesting though is that Apple did highlight a few features that wouldn't make it in the initial release and some coming in 2023. While I appreciate them being honest with their consumers, I think it highlights that a yearly release cycle isn't appropriate anymore and that they should switch to something that gives them more time or scale back their ambitions so that they can focus on delivering quality and stability.
