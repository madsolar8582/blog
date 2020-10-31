---
title: "Single App Mode"
subtitle: ""
date: 2020-10-31T10:00:00-05:00
tags: ["mdm", "authentication"]
---

When your app is running in an enterprise environment, you may come across deployments that lock devices into [Single App Mode](https://support.apple.com/guide/mdm/single-app-mode-payload-settings-mdm80a981/web). When this is enabled, the device cannot leave the app set in the payload and this can potentially impact your app's functionality. To detect this scenario, apps need to call [UIAccessibilityIsGuidedAccessEnabled](https://developer.apple.com/documentation/uikit/1615173-uiaccessibilityisguidedaccessena?language=objc) (Guided Access is the non-MDM term used for Single App Mode as users can lock devices to a single app for accessibility purposes). Once you know this restriction is in place, you can avoid attempting to launch other apps or other workflows (e.g. [MFMailComposeViewController](https://developer.apple.com/documentation/messageui/mfmailcomposeviewcontroller?language=objc)) until the restriction is lifted. However, this poses a problem for authentication.

If you are using [ASWebAuthenticationSession](https://developer.apple.com/documentation/authenticationservices/aswebauthenticationsession?language=objc) for federated login or are logging in via an Identity Service Provider that uses the [SSO Extension](https://developer.apple.com/videos/play/tech-talks/301/) APIs, the MDM will deny the other process from running as those processes do not match the app's bundle identifier allowed in the MDM payload. Therefore, your users cannot authenticate and hence cannot use your application. I logged this as FB8807837 and Apple states that this is the way they expect iOS to behave and consider allowing this kind of authentication to function an enhancement request.

---

I sincerely hope that this is addressed soon as it does not make sense for the Apple Enterprise team and the Authentication Services team to not have considered this a major blocker for enterprise deployments.
