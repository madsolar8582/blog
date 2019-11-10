---
title: "The Curious Case of Transporter"
subtitle: ""
date: 2019-11-10T07:00:00-06:00
tags: ["transporter"]
---

During this year's WWDC, Apple announced that a new App Store Connect API would be made available to developers. Buried in this announcement was another announcement that Application Loader would be removed in Xcode 11 and replaced by a new application. Xcode 11.0 was made available on 9/20/19, yet, this new application was nowhere to be seen. Fast-forward to October 15th, and the new application was revealed: [Transporter](https://apps.apple.com/us/app/transporter/id1450874784?mt=12). Once I started downloading it, I knew something was up as the download was quite large. Turns out, it is just a reskinned Application Loader, and that is sad.

Application Loader was loathed since it didn't support changes in Apple's authentication, e.g. 2FA, and because it used Java underneath to perform all of its operations. The reason for this is that uploading to App Store Connect allows you to leverage two proprietary protocols, [Signiant](https://www.signiant.com/) & [Aspera](https://asperasoft.com/), as well as the open protocol [WebDAV](http://www.webdav.org/) which aren't natively supported on Apple's platforms and that Apple's services are largely powered by Java (code reuse). However, with the announcement of a new app and new APIs, the developer community thought that Apple would take the opportunity to make this new application entirely native. Unfortunately, they didn't. While the application layer itself did get rewritten to be more modern and support Apple's infrastructure better, they reused the Java implementation underneath. I supposed this was due to timing constraints as Apple needed to deliver a minimum viable product relatively quickly.

---

Hopefully, in the future, Apple will release a new version of Transporter that is entirely native so that the download size is reduced and so that the application performs faster.
