---
title: "Pass the Remote"
subtitle: ""
date: 2019-04-28T07:00:00-05:00
tags: ["xpc", "remote view controllers"]
---

If you have a single application, typically you only need to get data from your server. However, if you have multiple applications that interact with one another, you need to be able to communicate between them. In terms of communicating data, Apple has you covered in quite a few ways: NSUserDefaults, App Groups, & Keychain Sharing.

## NSUserDefaults
[NSUserDefaults](https://developer.apple.com/documentation/foundation/nsuserdefaults/1409957-initwithsuitename?language=objc) is a great place to store small bits of information like preferences. This allows you to have a consistent user experience across all of your applications on the device.

## App Groups
An [application group container](https://developer.apple.com/documentation/foundation/nsfilemanager/1412643-containerurlforsecurityapplicati?language=objc) is a location on the file system that is shared between applications. This allows you to store any kind of file in it that can be used by your applications. This could be used for a shared database, document storage, or a shared cache.

## Keychain Sharing
[Keychain Sharing](https://developer.apple.com/documentation/security/keychain_services/keychain_items/sharing_access_to_keychain_items_among_a_collection_of_apps?language=objc) allows you to have a keychain that is accessible from all of your applications. This allows you to securely store secrets like API keys, passwords, or other credentials which could allow you to implement SSO or some form of MFA.

But what if you want your apps to be able to communicate state? Well, that is where iOS is somewhat lacking.

## Darwin Notification Center
Using a special type of [CFNotificationCenter](https://developer.apple.com/documentation/corefoundation/cfnotificationcenter?language=objc), the [darwin notification center](https://developer.apple.com/documentation/corefoundation/1542572-cfnotificationcentergetdarwinnot?language=objc) can [send](https://developer.apple.com/documentation/corefoundation/1542592-cfnotificationcenterpostnotifica?language=objc) notifications to multiple applications. However, there are some limitations:

1. Applications must register for these notifications.
2. There is no way of preventing other applications from not being able to subscribe to the notifications.
3. There is no way to authenticate the notification (i.e. ensure that it came from your application).
4. Apps that are in the background will not be woken up to respond to the notification unless they have background modes.
5. No object nor userInfo parameters are sent in the notification (i.e. there is no metadata).

With these limitations in mind, it becomes quite challenging to send context with the notification, so the consuming application must do additional work when it receives a notification. This essentially makes this functionality worthless as Apple does not allow for distributed notification centers on iOS which would allow you to send metadata.

## XPC
The true method of secure interprocess communication is Apple's [XPC](https://developer.apple.com/documentation/foundation/xpc?language=objc) framework. However, as you may have guessed, is not available on iOS. If it were available, you'd be able to send data to applications securely and be able to respond. But that isn't the only benefit of XPC as XPC allows for remote view controllers.

Remote view controllers are exactly what they sound like, a view controller that is remote (i.e. another application's/framework's) view controller hosted in your application. This is how [WKWebView](https://developer.apple.com/documentation/webkit/wkwebview?language=objc), [SFSafariViewController](https://developer.apple.com/documentation/safariservices/sfsafariviewcontroller?language=objc), and others are implemented by Apple in order to make sure those views have process isolation. If you yourself could implement something like this, you could embed your other apps' workflows within each other without having to launch them via URLs. Imagine if you could share authentication states and events between applications via XPC. That sure would be swell.

---

With iOS 13 just around the corner and it being rumored to focus on "pro" workflows, it would be a pleasant addition to the iOS SDK to allow for fully functional cross-app notifications and remote view controllers. Otherwise, iOS applications would still not have a reliable way of sharing state based information to take action on or be embeddable within applications. 
