---
title: "Breaking Code Signing via SSH"
subtitle: ""
date: 2019-01-20T09:00:00-06:00
tags: ["code signing", "xcode"]
---

Ok, so the title is a bit misleading. Code signing wasn't broken by SSH itself, it was the keychain interactions that occur when using SSH, but some context is needed. When you sign into the developer portal in Xcode, Xcode stores two items in the keychain that allows developer tools to access the developer portal on your behalf: `Xcode-Token` & `Xcode-AlternateDSID`. Prior to Xcode 9.3, these items were stored in the normal login keychain. However, since Xcode 9.3, these items have been stored in the local items keychain. The local items keychain isn't a "real" keychain as it isn't stored as a .keychain in ~/Library/Keychains. Rather, it is an SQL database with items in it that are managed exclusively by the system. This new format is also what allows it to be shared across multiple machines via iCloud.

These two items, when stored in the local items keychain, have a keychain access group set on them so that only Apple's approved applications can access them. This differs from the previous format where the user could modify which items could access those items. Now, when you start an SSH session, the user's keychain isn't unlocked. To unlock it, you must use the `security-unlock` command and pass it the keychain password and a reference to the keychain that needs to be unlocked. This is where the bug manifests. Since the local items keychain isn't a keychain that security-unlock can understand, you can't trigger the unlock. So, when you are using `xcodebuild` with the `-allowProvisioningUpdates` flag set, Xcode knows that you are signed in, but cannot access the credentials. This leads to a failed build with the error indicating that your developer portal session is expired.

Previously, Apple had an issue similar to this (introduced in Xcode 9.3) where Xcode would forget that you were logged in. To solve this, you needed to disable the new keychain service: `defaults write com.apple.dt.Xcode DVTDeveloperAccountUseKeychainService -bool NO`. Apple later fixed this issue in Xcode 9.3.1. So, I tried this workaround hoping for some success, but alas, it no longer worked. What did work though was to use Xcode 9.2 to recreate the credentials as it never used the new keychain service. This too stopped working once I upgraded to macOS 10.14 as any Xcode version less that 10 triggers a probable bug in Mojave as you cannot open the preferences of Xcode. The error in Console is as follows: _[NSApplication orderFrontPreferencesPanel] was invoked but there is no custom implementation and no Settings.bundle in the app bundle. Nothing will be displayed._

---

After logging a radar for this (46521387), Apple was able to provide a workaround that is based off of their previous workaround. It appears that they changed the preference key from `DVTDeveloperAccountUseKeychainService` to `DVTDeveloperAccountUseKeychainService_2`. Setting that to NO forces Xcode to store the credentials in the login keychain and that it is able to be unlocked when using an SSH session. Hopefully, this will be resolved in a future Xcode release.
