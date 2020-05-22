---
title: "Go Away Music App"
subtitle: ""
date: 2020-05-22T17:00:00-05:00
tags: ["apple"]
---

If you have any external device connected to your Mac that has audio controls, you have undoubtedly run into the scenario where you push the button(s) and the Music app decides to open itself (iTunes for macOS < 10.15). Obviously, you don't want this app open as maybe you are using a control on a device for teleconferencing or maybe you are trying to control Spotify or a browser, but alas, Apple really really wants you to use the music app.

Besides the fact that Apple provides no way to control what apps respond to external media controls, the listener for external media controls is always active in the form of the Apple Remote Control Daemon (com.apple.rcd). Now you could use `launchctl` to unload the daemon, but that is now impossible with [System Integrity Protection](https://en.wikipedia.org/wiki/System_Integrity_Protection). The next logical step would be to remove the execution bit of the app binary with `chmod`, but that too is impossible with SIP enabled.

So now what? Well, you create your own launch agent to constantly run an kill the Music app process when it tries to start. Luckily, other people got fed up with this and created an app to do this in a more user friendly way: [noTunes](https://github.com/tombonez/noTunes). This app can be set to launch upon login and can be disabled via the menu bar when you do actually want to run the Music app. This app has been a lifesaver and I cannot recommend it enough.

---

Once Apple stops trying to make their desktop operating system like a Fisher Price OS and allow users to control key aspects, apps like this won't be necessary. However, I'm not holding my breath as power user/developer features and controls have been disappearing for quite some time.
