---
title: "Broken Builds with Universal Xcode"
subtitle: ""
date: 2020-08-02T08:00:00-05:00
tags: ["xcode"]
---

When building a project in Xcode, two settings specify what architectures are used: `ARCHS_STANDARD` and `VALID_ARCHS`. By using the intersection between the two, Xcode produces binaries that include those architectures. However, this is now different with Xcode 12. In Xcode 12, `VALID_ARCHS` is "discouraged" and superseded by `EXCLUDED_ARCHS`. So what happens when you had custom values set for architectures when you attempt to build arm64 projects on x86_64 machines in Xcode 12? It fails to build.

I discovered this and logged feedback to Apple and Apple came to the conclusion that by having custom values for valid architectures that does not include x86_64 (since the build system maps arm64 to x86_64 for simulators automatically) was fine in Xcode 11 but that is no longer the case. Therefore, prior to using Xcode 12 with custom settings, you need to revert to the defaults so that Xcode 12 can correctly build projects since it now is universal (i.e. supports running arm64 code directly on Macs). 

---

In the event you forget about this and run into build issues later, `VALID_ARCHS` will show up in the User Defined settings section of your projects' build settings and you can remove it there. I still think a different and more descriptive error message would be better other than "target does not have compatible architectures" as most people won't understand what that means without a little bit of digging.
