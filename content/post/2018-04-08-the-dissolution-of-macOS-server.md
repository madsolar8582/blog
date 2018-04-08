---
title: "The Dissolution of macOS Server"
subtitle: "Hello darkness, my old friend"
date: 2018-04-08T07:00:00-05:00
tags: ["macOS", "server", "apple"]
---

Prior to the release of macOS Lion (née OS X Lion), the server portion of the OS was a separately licensed product and, depending on which license you purchased, you either had or didn't have a client limit for the server services you enabled. Although the server OS was mainly geared towards the Xserve<a href="#note1" id="note1ref"><sup>1</sup></a>, it could run on just about any Mac that could run the normal OS. When the time came for Lion to replace Snow Leopard, things changed.

Rather than continue to sell a separate OS, Apple decided to start leveraging their new Mac App Store to distribute the new Server app. Now, system administrators did have some initial concerns over the new distribution method since every time you needed to update the "server", you would have to manually relaunch the app for the update to complete. This is still a major headache today since if you manage headless servers, you need to use ARD to restart your services whereas the old way didn't require any user interaction at all due to the fact that the server services were part of the OS and restarted as soon as the server rebooted. However, the price was much better: you could get unlimited clients for $20 instead of $999. Another one of the benefits was that now the server features weren't necessarily tied to macOS updates and could get patched at other intervals.

For a while, you could still install the old server administration apps to configure a few things on a Mac running the new Server app, but they were primarily supposed to be used to manage Snow Leopard and older servers. Eventually, those apps stopped working and you had to use the new app to do everything (including [Xsan](https://en.wikipedia.org/wiki/Xsan) management). As macOS continued to evolve, so did the Server app, but then a new trend emerged.

As Apple released newer versions of macOS, it began absorbing server functionality (Caching, Time Machine, & File Sharing) and offered those services in the base OS while the more advanced features of the Server app started being hidden by default. Surprisingly, Apple even moved Xcode server directly into Xcode itself. Some of the changes made sense:

1. By moving file sharing into the base OS, users were able to setup file shares more simply.
2. By moving the Caching service, users were able to designate multiple Macs as mini Apple CDNs.
3. By moving the Time Machine service, users were able to designate any Mac as a backup hub.
4. By hiding more advanced services, new users wouldn't be overwhelmed and wouldn't screw up their network by enabling a service they didn't understand.

Alas, Apple's motives were different. Starting with macOS Server 5.6 (3/29/18), Apple announced the deprecation of almost all of their server services:

* FTP
* Documents
* DHCP
* DNS
* VPN
* Firewall
* Mail
* Calendar
* Wiki
* Websites
* Contacts
* Net Boot/Net Install
* Messaging
* Radius
* AirPort Management

With these services deprecated and scheduled for removal in the Fall of 2018 (aka macOS 10.14 launch), the only services that will be officially supported are Open Directory (LDAP), Profile Manager (MDM), & Xsan. According to their notice [page](https://support.apple.com/en-us/HT208312), one of the primary reasons for the removal of these features is availability:

> Customers can get these same services directly from open-source providers. This way, macOS Server customers can install the most secure and up-to-date services as soon as they’re available. Because these open-source services are the same services that currently ship with macOS Server, customers will be able to run them with the same data on the same Mac computers.

Since there is no longer a dedicated Mac team at Apple, it appears that there is no resourcing available to continue maintaining these services. By slimming down the Server app, the complexity of the software goes way down and that means they could speed up development of the existing features. This also means that Apple's enterprise presence takes a major hit. Without the built-in functionality provided by the Server app, there is little benefit of an organization to deploy a Mac server installation and just use Linux or Windows. Unfortunately, I don't think that this bothers Apple since they killed off the Xserve way back in 2010 and haven't embraced virtualizing or containerizing macOS.

I guess we'll see in June of this year what plans Apple has in store for the future of the Server app, but the migration to the open-source versions of these services will be a pain since most don't have a GUI. My guess is that organizations will swap over to some other commercial implementation since enterprises love support contracts. Regardless, I'm sad to see the Server aspects of macOS get gutted so unceremoniously since Apple could have made a move into the business space with their simple but powerful Server offerings (especially the small/medium market). 

<a id="note1" href="#note1ref"><sup>1</sup></a> RIP Xserve
