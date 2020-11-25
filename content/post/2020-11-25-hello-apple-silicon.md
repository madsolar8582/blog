---
title: "Hello Apple Silicon"
subtitle: ""
date: 2020-11-25T09:00:00-05:00
tags: ["arm", "macos", "big sur", "apple silicon"]
---

After 12 years, it was finally time to replace my original 2008 unibody MacBook Pro. With the iPad Pro taking over the role of mobile computing device, I opted for a Mac Mini with the M1 chip. First impressions are that the hardware is really fast, the I/O is a bit sparse, and the Big Sur is buggy.

One of the first things I tested was Xcode unarchiving and I was able to use `xip` to unpack Xcode 12.2 in under 4 minutes compared to about 25 minutes on the 2016 MacBook Pro I use for work. App launches are also pretty quick and the simulator boots near instantaneously. However, Homebrew needs some work (as well as arm compatible packages), many Ruby dependencies with native code need uplifts, and quite a few other apps needs native arm binaries. This means that to turn this into a full development machine will take another few months, but progress is being made steadily. Otherwise, I am quite happy with it.

Now as far as Big Sur goes, it was only released so that Apple could sell the M1 Macs (i.e. it isn't ready). When I got the Mini, it shipped with 11.0 and needed to be updated to 11.0.1 (the GA release). It took me 3 attempts to get the update to install successfully. Luckily, the machine did not brick like others have reported requiring revivification via Apple Configurator. Shortly thereafter, Apple released a revised 11.0.1 (20B50) and that release is unavailable to me nor does anyone know what was fixed in it. Currently I'm experiencing the "normal" macOS issues with external monitors and WindowServer using too much CPU. Allegedly 11.1 (note: new release scheme) is fixing WindowServer, but no one has stated that monitors losing settings/positioning is fixed yet. I've also had UI elements stutter and iCloud misbehave. 

One of the things that did work reasonably well was using iOS apps natively. Of the apps that I own, only 5 are offered via the Mac App Store so I've had to offload the ipas directly from my devices for other apps that I want to use. It is interesting that Apple allows this instead of resigning the approved apps with something so that unapproved apps cannot be used.

---

Overall, the M1 powered Macs are in a good spot hardware wise but suffer from macOS stability issues. I think if less time had been spent on the UI changes and instead had been spent on the core OS, things would be in a better spot.
