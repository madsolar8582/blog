---
title: "Thoughts on WWDC 2021"
date: 2021-06-14T08:00:00-05:00
tags: ["apple", "wwdc"]
---

Continuing the with the very high production value, this year's virtual WWDC was very good. This year included collaborative lounges (in Slack) where lots of Apple engineers could answer whatever question you might have. If WWDC continues to be virtual, I think this is a good way of reconstructing the lab experience so long as the questions and answers get archived publicly.

## watchOS

The watch got a very small update this year with support for keys and IDs being brought to the Wallet app and a fully updated Mindfulness app. Still no sign of a full departure from being reliant on the phone, but perhaps this year is just a catch up year due to COVID with next year being a big year.

## tvOS

Unsurprisingly, tvOS didn't get much screen time. The updates were minor alignment things: SharePlay, Spacial Audio, & HomeKit support for Matter. Since the platform is essentially done, I don;t expect much to change next year either.

## iOS

iOS got a moderate feature bump this year with the FaceTime improvements, redesigned notifications, and Siri being more on device. With SharePlay via FaceTime and text OCR in images, I can see those being the most used features outside of the new focus system. What's weird though is that these features seem to be influenced by the pandemic, but are coming out when we are, hopefully, putting COVID behind us. For example, Microsoft switched gears on a lot of their products and pushed out helpful features during the pandemic. This makes me wonder how agile Apple is to adjusting feature road maps and if they considered backporting tent pole features to older OSes due to the apparent need.

## iPad OS

This year's iPad OS update was essentially adding the features from iOS 14 that were cut due to time last year. However, the multitasking UI improvements (the "..." menu and Shelves) in additional to being able to control multitasking from the keyboard should make iPad app usage way more productive. Widgets on the home screen should also be a value add as the additional space on the bigger home screen allows widgets to expose more functionality.

## macOS

Luckily, no really big overhauls were announced this year which should mean that things are going to stabilize. Universal control should be a big help in multi-device workflows and having Macs become AirPlay receivers helps replace the missing Target Display Mode.

## Swift

This was a big year for welcome improvements. Now being able to have easy localized as well as attributed strings (especially in SwiftUI) continues to bring convenience up to par with other languages. The additional changes to `Date` to allow for Swift native formatting instead of bridging into `NSDateFormatter` is very cool. The biggest change though is in concurrency (and the one that is driving a lot of conversation online). async/await and Actors allow Apple to move away from GCD and threads in a traditional context and go towards a much more thought out concurrency model with stricter runtime guarantees that make for safer code. However, with the target requirement being the new OSes, a lot of developers have to have multiple concurrency models in place and have requested that Apple backport the feature. Apple is declining to do so since it requires kernel changes. Google and Microsoft have gotten away from this problem by allowing you to bundle the explicit runtime into your application (which Apple does not want to do).

Additionally, DocC was introduced this year and I think it is great. By having a standardized documentation system that also exports to Xcode compatible archives as well as a web app hopefully means that Apple themselves will begin improving their documentation situation. For me though, it still doesn't support Objective-C, so I'm still waiting to take advantage of it.

## Xcode

Minor UI refreshes were included this year with the focus on source control feature and Xcode Cloud. Xcode Cloud should be interesting once it goes GA sometime next year, but having no prices listed dampens some of the enthusiasm. For small developers, this should be a boon. However, for large enterprises (as well as developers who distribute outside of the App Store), I don't see this being useful due to the inherit limitations of the product.

## Safari

A very controversial year for Safari due to the UI redesign. Many people were asking for a "Safari Classic" mode, but for macOS I forsee an exodus to Edge or Firefox for most power users. On iOS, the goal appears to have been reachability and it seems to work ok on the iPhone, but on iPad with multitasking, the UI doesn't seem quite right. Hopefully Apple will adjust quickly through the beta period before final release.

## iCloud

This was a surprise this year with the announcement of iCloud Relay. This is going to be an interesting thing to debug when Apple, Cloudflare, or Akamai experience issues as most users will not understand that their traffic going to ingress or egress proxies is impacted. From a privacy perspective, this is great as a lot of VPNs that say they are private aren't really and they don't also anonymize the exit.

---

Overall this was a pretty solid year and should hopefully allow Apple to catch up on stabilization due to pandemic impact. I will say that the thing that I thought was the best announcement was the new `UIKeyboardLayoutGuide` as you no longer have to use notifications plus view coordinate math to "avoid" the keyboard.

Theoretically, next year will be a big feature year and I'm looking forward to what Apple can do to move the iPad closer to a more professional/power user computing device.
