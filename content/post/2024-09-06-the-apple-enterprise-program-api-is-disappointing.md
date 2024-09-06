---
title: "The Apple Enterprise Program API is Disappointing"
subtitle: ""
date: 2024-09-06T07:55:00-05:00
tags: ["apple", "enterprise"]
---

At WWDC 2024, Apple announced that _finally_ Enterprise Account members would get an API to interact with the developer portal akin to the App Store Connect API. Last week, Apple made the API available and published the [official documentation](https://developer.apple.com/documentation/enterpriseprogramapi) for it. 

Since this isn't App Store Connect, not every feature that API [provides](https://developer.apple.com/documentation/appstoreconnectapi) is available, but the core pieces are there. At the most basic level, in order to use either API, a key is needed and the process to generate this key is the same.

For command line builds, `xcodebuild` has the option of interacting with the developer portal to update provisioning profiles and/or register new devices via the `-allowProvisioningUpdates` and `-allowProvisioningDeviceRegistration` flags respectively. By default Xcode uses the credentials of the account(s) stored in the keychain. However, this is no longer the recommended approach by Apple. Instead, Apple wants you to use the new API keys.

To do this, you need to pass three additional flags: `-authenticationKeyPath`, `-authenticationKeyID`, and `-authenticationKeyIssuerID`. The values of these are the key downloaded from the portal and the key identifier and the issuer id (both shown in the portal).

This is what led to a surprising discovery: keys for Enterprise accounts don't work with xcodebuild. Every time I tried using it, I would get the following error:

```
Invalid authentication key credential specified (CryptoKit.CryptoKitASN1Error.invalidPEMDocument)
```

Given the message indicates a potentially malformed key, I asked DTS if there was an issue with the Enterprise key generator. They responded that no, the generator is working fine and that the use case I was trying to do is not supported. That response left me baffled.

Nowhere in the official documentation nor in the announcement email does it state that this workflow is not supported, the man page for xcodebuild does not state that this workflow is not supported, and the error message from is probably technically correct, but does not disclose the actual problem (so there is room for improvement).

Giving Apple the benefit of the doubt, getting an MVP out before the launch of the new OSes makes sense. However, knowing that there is another use of the API, it should be communicated that this feature is not yet available and is coming sometime in the future. 

This is simply disappointing from a product perspective because I can't use it even though I was promised a usable API, but also from the lack of care put into the documentation and developer experience.

---

Of course, DTS asked me to log a Feedback requesting this feature. I'm pretty sure that since the introduction of the App Store Connect API in 2018 all Enterprise customers have been asking for this, so this isn't new nor a surprise...
