---
title: "Xcode 16 Beta Developer Portal Issue"
subtitle: ""
date: 2024-07-09T07:45:00-05:00
tags: ["apple", "xcode", "beta"]
---

There is a quite annoying regression that was introduced in Xcode 16 β2 that was not present in β1 nor Xcode 15.4 where the account that you are signed into's credentials are not accessible on the command line via `xcodebuild`.

Typically, in the keychain, you'd see `Xcode-Token` and `Xcode-AlternateDSID` indicating that Xcode has an account signed in and it uses those credentials to talk to the developer portal when you pass `-allowProvisioningUpdates` to your build. Starting with Xcode 16 β2 (and not fixed in β3), those values are not present even if Xcode's UI says that the account is signed in. If you quit and relaunch Xcode, the UI indicates that the session is expired and wants you to sign in again. Interestingly, if you do command line builds, a whole bunch of "Unknown Accounts" are added to the account list with zero account information. You have to remove each one as they cause issues for Xcode trying to determine what account to use for code signing.

From the command line, you see these errors:

```bash
xcodebuild[47235:33073528]  DVTDeveloperAccountManager: Failed to load credentials for <UUID>: Error Domain=DVTDeveloperAccountCredentialsError Code=0 "Invalid credentials in keychain for <UUID>, missing Xcode-Username" UserInfo={NSLocalizedDescription=Invalid credentials in keychain for <UUID>, missing Xcode-Username}
xcodebuild[47235:33073528]  DVTDeveloperAccountManager: Failed to load credentials for <email>: Error Domain=DVTDeveloperAccountCredentialsError Code=0 "Invalid credentials in keychain for <email>, missing Xcode-Token" UserInfo={NSLocalizedDescription=Invalid credentials in keychain for <email>, missing Xcode-Token}
...
error: The operation couldn’t be completed. Unable to log in with account ''. The login details for account '' were rejected.
```

---

I haven't found a workaround for this. Logged as FB14060904.
