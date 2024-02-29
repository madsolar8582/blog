---
title: "New ASWebAuthenticationSession Features"
subtitle: ""
date: 2024-02-29T09:40:00-05:00
tags: ["aswebauthenticationsession"]
---

New in iOS 17.4 is the ability to set headers on an instance of [`ASWebAuthenticationSession`](https://developer.apple.com/documentation/authenticationservices/aswebauthenticationsession?language=objc).

Like `NSURLSession`, simply add to the new [`additionalHeaderFields`](https://developer.apple.com/documentation/authenticationservices/aswebauthenticationsession/4316246-additionalheaderfields?language=objc) property.

However, this is not the only change. There is now a new initializer, [`initWithURL:callback:completionHandler:`](https://developer.apple.com/documentation/authenticationservices/aswebauthenticationsession/4316247-initwithurl?language=objc), which leverages the new [`ASWebAuthenticationSessionCallback`](https://developer.apple.com/documentation/authenticationservices/aswebauthenticationsessioncallback?language=objc) class. This allows developers to more strictly control callback behavior by differentiating custom schemes and HTTP URL schemes.
