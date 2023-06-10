---
title: "Thoughts on WWDC 2023"
subtitle: ""
date: 2022-06-10T07:30:00-05:00
tags: ["apple", "wwdc"]
---

As telegraphed, WWDC this year introduced a new product and platform: [Apple Vision Pro](https://www.apple.com/apple-vision-pro/) and [visionOS](https://developer.apple.com/visionos/). Apple calls this "Spatial Computing" instead of AR/VR and they technology behind it (hardware and software) is pretty impressive. I'm still of the mindset that there is no killer set of features that'll make this go mass market, and as a result, will still be very expensive and niche. Funnily enough, the sessions refer to this as xrOS (the original leaked name), visionOS, and "Apple's spatial computing platform" which means that marketing came in _really_ late and changed the name. Myself and other suspect that this is why the SDK is not available until Seed 2 or Seed 3 as all of the symbols and documentation need to be changed.

## iOS

In continuation of Apple realizing that customization is a big hit with users, iOS 17 now allows for customized posters (the contact image, name, and font) when people call you. This is an extension of the lock screen customizations introduced last year and can be saved and shared via contact cards. Additionally, FaceTime allows you to leave a video voice mail and the phone app now lets you see a visual transcription of voicemails as they are made (just like what Google Assistant lets you do on Android via call screening). Overall, this is a pretty big year for phone related changes.

Messages got a small UI redesign to hide the iMessage apps and stickers (which I rarely use) and Apple introduced a new API [Sensitive Content Analysis](https://developer.apple.com/documentation/sensitivecontentanalysis) to help app developers handle potentially unwanted content without having to setup and maintain their own ML models (pretty handy).

Most surprisingly is the introduction of StandyBy, which allows an iPhone in landscape orientation to act as an information hub while docked to a new set of accessories (via [DockKit](https://developer.apple.com/documentation/DockKit)). This is interesting due to the fact that, as of right now, only the 14 Pro and 14 Pro Plus have Always-on-Displays which make this feature useful. Other models can use this feature, but you have to tap the screens in order to see the content. This feature is not mentioned to be supported on the iPad, which is strange as it would make more sense on those devices due to screen real estate. Google, for example, makes heavy use of this (and multi user profiles) on the Pixel tablet with a dock. Perhaps this will come to iPadOS later once Apple reveals new hardware with OLED screens and dock accessories. Otherwise, this seems like a miss or perhaps this is a testbed for a new HomePod device with a screen (similar to the Google Nest Hub).

## iPadOS

iPadOS is almost back to full feature parity with iOS as the lock screen customizations have made their way to iPadOS 17. Due to this, however, is that now widgets on the home screen and lock screen are interactive on both platforms. The Health app also makes its way over and Stage Manager has gotten some needed improvements. Still no calculator...

## macOS

macOS 14 is known as macOS Sonoma and, like iPadOS, is an incremental update. Widgets are making a return (long live Dashboard) by being able to run on the desktop directly rather than being limited to Notification Center. Also, you can use Continuity to have widgets appear from a nearby device without having to have that app installed on the corresponding Mac. Additionally, Apple now has the tvOS screensavers available on macOS and there are new video conferencing features provided by [ScreenCaptureKit](https://developer.apple.com/documentation/screencapturekit) that let apps not need to request the screen sharing permissions.

## tvOS

Surprisingly, Apple didn't forget that tvOS exists this year and brought a feature to tvOS that I've been asking for (albeit in the most Apple way possible). You can now leverage FaceTime on AppleTV by using Continuity Camera via an iPhone or iPad nearby. I suppose this was the only way that it could be done given that Apple does not have USB ports on the AppleTV so that you can connect an external camera (like you can do on iPad), but this just means that you have to purchase the docks/mounts in addition to an expensive iPhone/iPad. However, this is still probably cheaper (and better quality) than some teleconferencing displays.

## watchOS

Apple has basically visually redesigned every system app in watchOS 10 and attempted to make navigating apps and getting to Control Center easier. Still no custom watch face APIs though...

## Xcode

Apple has made Xcode smaller again this year by unbundling the iOS simulator runtime. Additionally, a new highly parallel linker was introduced as well as support for [Mergable libraries](https://developer.apple.com/documentation/xcode/configuring-your-project-to-use-mergeable-libraries) (a way to make dylibs behave like static libs). Additionally, Apple added bookmarks so that you can navigate directly to certain areas of code, added a live rendered documentation view (à la VS Code), and added syntax highlighting to statements that are currently compiled out. Xcode now supports UI previews of UIKit and AppKit more easily and adds support for debugging the new Swift Macro system. Overall, very meaningful improvements to the code editing experience.

## Safari

This year, Safari is gaining features that other browser have had for a while as well as adding support for new features before anyone else. The big change (other than the spatial computing stuff) is that Safari will now allow you to save websites as web applications on macOS similar to what Chrome and Edge do. Safari is also adding support for HEIC images and JPEG XL. The JPEG XL bit is funny because it was [introduced by Google](https://bugs.chromium.org/p/chromium/issues/detail?id=1178058) and supported by various companies (e.g. Adobe) and then Google removed support for it before ever making the feature GA. Now, Apple is taking the lead and convincing the internet that this format is worth implementing and hopefully Mozilla follows suit.

Web Inspector has also gained some new features like typography debugging, accessibility overrides, and symbolic breakpoints.

Private Browsing also gains new protections such as much stricter anti-fingerprinting and anti-tracking including the removal of tracking parameters from URLs. Additionally, extensions get granular permissions (e.g. being able to run in private browsing mode) and Safari now supports profiles like Chrome/Edge and Tab Containers on Firefox.

---

This year's focus was clearly on visionOS so there aren't too many big changes to the other platforms. SwiftUI and Swift itself continue to mature at a steady pace, but Apple still has challenges in the form of government regulations and no indication was given this year of acquiescing to opening up the platforms in any meaningful capacity.
