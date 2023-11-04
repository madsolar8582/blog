---
title: "Flexible Computing"
subtitle: ""
date: 2023-11-01T08:40:00-05:00
tags: ["macos", "windows"]
---

For many individuals, computers are tools used to accomplish tasks. This could be development work, art generation, media consumption, or simply a portal to the web. Apple's approach more recently with the underlying unification of its platforms is to enable these tasks and workflows, but in ways Apple wants you to do so. 

Take, for example, iPadOS. It has a limited multi-tasking and windowing system as well as a limited file system. However, it has the same hardware as a MacBook running macOS, so you would expect it to be able to do the same work. Nope.

Another example is on iOS. If you go to your Wi-Fi network and look for information regarding what band you are connected to and what authentication mechanism is in use, it doesn't tell you. Android will.

Complexity and information density has also been slowing going away in macOS. This is mainly due to Apple unifying the experience between all of their platforms so that it is easier for users to move between them and pick up how to use them intuitively. For power users and developers, there are still ways to access more technical information by knowing key commands, where to press <kbd>Control</kbd> or <kbd>Option</kbd>, or [`defaults write`](https://macos-defaults.com/) preferences.

However, if we take a look at Windows, we can see Apple is not being as flexible with how it could enable and empower its users.

Do you want to run a new app in a controlled environment to test it out prior to using it on your main system? Apple does not provide a way to do so, but Microsoft provides [Windows Sandbox](https://learn.microsoft.com/en-us/windows/security/application-security/application-isolation/windows-sandbox/windows-sandbox-overview) for this exact purpose.

Do you want to enforce hardened security on network access and applications? Apple does allow for macOS apps to opt-in to the Sandbox and Hardened Runtime, but Microsoft goes a step further and provides [Microsoft Defender Application Guard](https://learn.microsoft.com/en-us/windows/security/application-security/application-isolation/microsoft-defender-application-guard/md-app-guard-overview) which virtualizes and isolates individual applications from the host operating system for an extra layer of security.

Do you want to work on a macOS cloud desktop remotely from any machine? Apple does not provide general purpose cloud desktops of macOS, but Microsoft offers [Dev Box](https://azure.microsoft.com/en-us/products/dev-box/)es which allow you to develop anywhere in the world using just a browser.

Do you want to use a preconfigured VM to test something? Apple does not provide VM images of macOS, but Microsoft [does](https://developer.microsoft.com/en-us/windows/downloads/virtual-machines/) for free and supports all the major formats (VMWare, Hyper-V, VirtualBox, & Parallels). Also, while we're on the topic, Xcode does not allow you to bundle up fullying configured development environments, but Visual Studio Code (a Microsoft product) does via [Dev Containers](https://code.visualstudio.com/docs/devcontainers/containers).

Do you want to use a Linux distro for development and/or testing that is integrated with the host operating system? Apple does not provide a built-in method of doing so, but Microsoft does via [Windows Subsystem for Linux](https://learn.microsoft.com/en-us/windows/wsl/about). This even allows you to run GUI Linux applications.

Do you want to install development tools, libraries, or apps via the command line? Apple does not provide a built-in package manager (though [Homebrew](https://brew.sh/) exists), but Microsoft does via [Windows Package Manager](https://learn.microsoft.com/en-us/windows/package-manager/) (winget). You can even get a [GUI interface](https://www.marticliment.com/wingetui/) for it.

Do you want an easy way to setup your development environment? Apple allows you to download Xcode from the App Store, but only focuses on Apple development. Microsoft, on the other hand, provides [Dev Home](https://learn.microsoft.com/en-us/windows/dev-home/) which is an extensible platform that allows you to setup a machine for all kinds of development easily using a single tool.

What about enabling power user features and exposing core internals of the system? Well, as mentioned earlier, if you know what commands to execute or install additional debug tools accompanying Xcode, you can get some of this on macOS. However, Microsoft publishes [PowerToys](https://learn.microsoft.com/en-us/windows/powertoys/) and [Sysinternals](https://learn.microsoft.com/en-us/sysinternals/) which makes it much easier to customize and interact with Windows. You can even use a 3<sup>rd</sup> party app called [ViVeTool](https://github.com/thebookisclosed/ViVe) to enable feature flags in Windows to test new features.

Now, that's not to say everything is perfect in Microsoft's ecosystem as apps like [ShutUp10](https://www.oo-software.com/en/shutup10), [Privatezilla](https://www.builtbybel.com/apps/privatezilla), & [DoNotSpy10](https://pxc-coding.com/donotspy11/) exist to disable some invasive and unwanted features/capabilities of Windows. However, from a flexibility and ease-of-enablement perspective, Microsoft is doing much more than Apple.

An alternative to some of the above is to use VMs. However, Apple restricts virtualization of macOS to macOS hardware unless you have a special agreement with them and I suspect once macOS drops x86_64, a lot of ESX setups will become useless. Apple also restricts VMs by only allowing a maximum of two virtualized instances to be running at a given time. The macOS VMs running on Apple Silicon also don't support signing in with an Apple ID, which restricts what you can do with them. For completeness, there are even more "[gaps](https://eclecticlight.co/2023/09/14/current-limitations-on-macos-virtual-machines-running-on-apple-silicon-macs/)".

Given the overall market share of Apple desktops, Apple probably doesn't see this as a problem, but organizations attempting to deploy and manage large fleets of Mac devices are likely to pass given the capabilities of the Windows ecosystem (including much more powerful MDM features). Let's hope Apple can reinvigorate macOS, but knowing them, they want money based on Apple hardware and cloud services and some of these things do not align with that goal.

---

I will note, macOS on Apple Silicon can run iOS/iPadOS apps and Windows can do the same via [Windows Subsystem for Android](https://learn.microsoft.com/en-us/windows/android/wsa/) with the caveat that Google Play Services are not supported.
