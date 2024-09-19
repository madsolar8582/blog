---
title: "Apple Enterprise API Continued"
subtitle: ""
date: 2024-09-19T07:35:00-05:00
tags: ["apple", "enterprise"]
---

After posting on the forums indicating the issue with the API keys, the original DTS response was to log a Feedback with a suggestion for allowing these keys to work with xcodebuild. However, a few days later, I got a follow up post from DTS indicating that this was indeed a bug and the Feedback should be a bug report. I logged FB15055171 and within a few days I was told to regenerate the keys.

These new keys did finally work via xcodebuild, but with a twist: they could only be used when performing the `exportArchive` operation and nothing else. This is different from when an account is signed in as the development profile can be updated during the `build` or `archive` operations (in addition to potentially updating devices in the portal). According to DTS, this is intended, so I gave up on using the API and will continue using an account signed into Xcode even though Apple no longer recommends this approach.

I'm sincerely grateful for the DTS engagement I received while trying to resolve this issue as they were quite responsive and provided detailed answers/comments to my questions. However, I'm still disappointed in how this launched as this bug was easy to trigger, so I'm not sure this was tested thoroughly and the documentation is lacking regarding all of these limitations.

---

The entire reason I went down this path was due to FB14060904 where the keychain was not working correctly and Xcode was unable to communicate with the portal. Now that Xcode 16 is GA, the security content was [published](https://support.apple.com/en-us/121239) and two items stand out: CVE-2024-44162 & CVE-2024-40862. One or both of these vulnerabilities, to me at least, indicate why Xcode 16 changed how it handles its own credentials which caused the issue with my keychain. To ultimately resolve the issue, I had to attempt the keychain reset [workflow](https://support.apple.com/guide/keychain-access/if-you-need-to-update-your-keychain-password-kyca2429/mac) and that didn't work. Therefore, I had to manually delete all of my keychain content in `~/Library/Keychains` and reboot. After that, Xcode was able to store credentials in the keychain.
