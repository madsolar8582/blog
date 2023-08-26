---
title: "Privacy Manifest API List"
subtitle: ""
date: 2023-08-26T08:40:00-05:00
tags: ["apple", "privacy"]
---

As communicated at WWDC, Apple has [published](https://developer.apple.com/documentation/bundleresources/privacy_manifest_files/describing_use_of_required_reason_api?language=objc) the list of APIs it now deems sensitive and frameworks/apps must include a justification for using them. The APIs are divided into five categories: file timestamp access, system boot time access, disk space access, active keyboard access, and user defaults.

As with some other recent attempts by Apple to prevent fingerprinting and tracking, legitimate use cases get caught in the crossfire. 

Take for example the file timestamp access APIs (both metadata and stat/fstat) and needing to reconcile data on device and a server. This syncing process would likely need to see which data is newer and attempt to merge if possible. Now, you'd have to justify this use case.

What about the system boot time? Sometimes you may want to have the user reauthenticate after the device reboots or you may want to do a precise calculation based off of time and so you query `mach_absolute_time`. Another justification needed.

Let's say that you want to write a large file to disk or perhaps want to copy a file and make modifications to it. Rather than waste time doing this operation, you'd check the available disk space prior and inform the user if there isn't enough. But now, you guessed it, justification needed.

As far as the active keyboard is concerned, usage of this is less clear to me. Following good programming practices, you annotate your text fields/views with the appropriate [`UIKeyboardType`](https://developer.apple.com/documentation/uikit/uikeyboardtype?language=objc) and the system will provide what you need. I suppose grabbing the current keyboard lets you inspect the user's active language but there are far more relevant APIs for that via [`NSLocale`](https://developer.apple.com/documentation/foundation/nslocale?language=objc).

Lastly, the big one is the justification required for using [`NSUserDefaults`](https://developer.apple.com/documentation/foundation/nsuserdefaults?language=objc). Storing user preferences and caching data is quite common in apps and this API makes it easy to do so. This also allows apps to share data between the host app and its extensions as well as between apps from the same developer so that the user can have a good experience. Now, malicious actors could store a bit of data that is based on some calculation that is the same across apps and now you do have a tracking vector, but even MDMs use this API so what's the big deal?

---

In short, all of these APIs have legitimate use cases that developers now have to justify to non-technical people in app review and that is a recipe for disaster. Instead, I would have expected Apple to provide replacement APIs with privacy protections built in. Since they didn't, I have to assume that they discovered that there is no meaningful way of doing so or it would be too costly/disruptive to developers. Regardless, I'm all for the privacy transparency initiative, but please make it simpler for developers and don't put more decision making power into App Review when they are unqualified to make said decisions.
