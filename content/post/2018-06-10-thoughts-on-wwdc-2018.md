---
title: "Thoughts on WWDC 2018"
subtitle: ""
date: 2018-06-10T11:00:00-05:00
tags: ["apple", "wwdc", "software"]
---

This year's WWDC was supposed to be a sparse in terms of new announcements (if the rumors were to be believed) due to Apple changing up the feature set of their major releases due to the poor reception of iOS 11 and macOS 10.13. In fact, iOS 11 has an atypical distribution share compared to iOS 9 and 10 when they were a year old (low 80s instead of high 90s). Over the weekend, Apple accidentally leaked some of the upcoming features of macOS 10.14, Xcode, and the Mac App Store with the publication of a video preview under Xcode's application identifier. The video showed off a Dark Mode, new applications, and some minor source code editing changes inside of Xcode. The confirmation of Dark Mode for macOS did quite a bit for moral going into the keynote on Monday as macOS hasn't had much attention in the past couple of years. Of course, when the day came, Apple went to bat with lots of content.

## iOS 12
The main focus of this year's iOS release is performance and stability (and this is a good thing). iOS 11 was an ambitious release packed with many features, and as Apple learned, too many features. Since AirPlay 2 and iMessage in the cloud released 51 weeks after their announcement, changing the roadmap to slim the initial release just to contain the core features is smart since minor releases are excellent stepping stones for post-launch features. By focusing on making the OS run smoother and by addressing all of the little things, reducing the "death by one-thousand cuts" experience leads to happier users (and hopefully a larger install base). It was also nice to see Apple trying to improve performance on the oldest devices since this means the more powerful devices will perform even better.

Bringing additional apps to the iPad was inevitable, but I can't help but think they missed out with the weather and calculator apps. If Apple wants to continue pushing the iPad into a computer replacement, it would make sense that all iOS devices have the same set of apps. Regardless, Apple also [sherlocked](https://www.urbandictionary.com/define.php?term=sherlocked) all of the AR measuring tools by creating the new Measure app leveraging ARKit 2.

The new photo sharing stuff mirrors what Google showed off in Google Photos last year, but the difference being that Apple has done all of the machine learning on device, rather than process it in the cloud. Similarly, Siri does most of its learning on device and she got a bit more useful this year. With the ability to create "shortcuts", you can now create [IFTTT](https://en.wikipedia.org/wiki/IFTTT)-like workflows based on system offerings and workflows from 3rd-party applications. While this doesn't make Siri as powerful (and accurate) as the Google Assistant, it is definitely a step in the right direction.

Finally, there are also grouped notifications, improvements to do-not-disturb, new digital health tools to keep you from using your device too much, group FaceTime, and Memoji. These enhancements will be beneficial to users who use their device a lot and the changes to the camera and the addition of Memoji will allow Apple to sell tons of iPhone X's (and the new iPads coming this fall).

## watchOS 5
I was pleasantly surprised to see some worthwhile features added to watchOS as I see this platform essentially being "done" since the hardware is also somewhat static. The walkie talkie feature is neat, but it still isn't radio to radio (I can also see people freaking out when they aren't expecting to receive a message). Other than the fitness announcements, WebKit on the watch was weird. Since you are essentially getting the smallest viewport possible, web content is automatically rendered in reader-mode if possible. It'll be nice to get a quick preview, but I can't image people using this often.

## tvOS 12
Now this platform I don't see changing drastically anytime soon (and I think Apple has run out of ideas). tvOS hasn't caught on for gaming, but it is doing well in terms of 4K HDR content. To compliment this, tvOS is now gaining support for Dolby Atmos audio (it already supports Dolby Vision HDR). This is nice for me personally since I have a Dolby Atmos setup, but I don't think the general consumer has that kind of setup. The other nice additions are the new ISS aerial screensavers as each screensaver Apple has released so far has been gorgeous.

## macOS 10.14
First, Apple ran out of cats and now, out of mountains. macOS 10.14 is the first (in a likely series) of releases with a desert theme: Mojave. Mojave contains the new Dark Mode that affects all system UI elements instead of the current dock and menu bar. Additionally, Finder got some improvements by offering Desktop Stacks (to clean up desktops) and quick actions. However, I think that people who keep messy desktops have their own organization system in place, so stacks will likely make them sad.

The biggest announcement though was a sneak peek at some future refinements of AppKit and UIKit. The Marzipan project has been going on internally at Apple for some time now and instead of it being a UIKit replacement for AppKit, it is more of a translation of UIKit for AppKit. Both UIKit and AppKit are built on top of Apple's core frameworks, but those have diverged since the introduction of UIKit since the needs of both platforms have been quite different. Now, Apple is taking the time to unify the core frameworks so that UIKit apps can run on macOS via a translation process. This has the benefit of removing bugs from both platforms at the same time, but also increase the number of apps on macOS by having developers port over their iPad apps. Ideally, this means that both AppKit and UIKit will stick around, but we saw what happened to Carbon when Cocoa came about. 

## Developer Tools
Xcode improvements this year were all focused on the editing and build experience. The source code editor got the ability to do code folding again, but also gained multi-line editing, the ability to see source control changes in the IDE, and faster project opening. Auto Layout and Interface Builder got performance increases (which hopefully means that IB is usable again) and Instruments got the ability to import custom templates that leverage the new signpost API.

On the testing front, we can now randomize the test order (good for discovering hidden test dependencies) as well as run the tests in parallel on the same destination (complementing parallel destinations).

Lastly, ARKit, CoreML, CreateML, and Network.framework were big focuses API-wise. Apple is banking on developers using ML and AR to create whole new applications to drive new categories on the app store and move off of legacy networking code so that all apps use user-space networking for performance and security reasons.

## Conclusion
This year's conference didn't have any major wow moments, but it was full of solid improvements. Hopefully, this means that be establishing a solid foundation this year, we'll see some really cool new features and possible redesigns next year.
