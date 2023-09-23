---
title: "Good Idea, Flawed Execution"
subtitle: ""
date: 2023-09-23T10:25:00-05:00
tags: ["apple", "privacy", "safari"]
---

Prior to iOS 11, developers could leverage [`SFSafariViewController`](https://developer.apple.com/documentation/safariservices/sfsafariviewcontroller?language=objc) to interact with Safari in app view a remote view controller for authentication. However, because this is a view controller, it can be hidden from the user. As a result, malicious actors abused this API to track end users without their consent. Apple's strategy to combat this was to limit the data sharing capabilities of this API to only the app (i.e. creating a sandbox). Of course, this meant that SSO no longer worked. After the industry reached out, Apple introduced [`ASWebAuthenticationSession`](https://developer.apple.com/documentation/authenticationservices/aswebauthenticationsession?language=objc) (née [`SFAuthenticationSession`](https://developer.apple.com/documentation/safariservices/sfauthenticationsession?language=objc)) that does share data across apps, but with a twist.

The new API prompts the end user to allow for the data sharing and is hosted outside of the app's view hierarchy in order to prevent abuse. The prompt has the following attributes:

Title: "Example App" Wants to Use "example.com" to Sign In<br>
Text: This allows the app and website to share information about you.<br>
Option 1: Cancel<br>
Option 2: Continue<br>

App developers do not have any way of providing context to this alert and the description provided by Apple makes the action seem malicious rather than a standard action. This especially is weird when you need to open Safari to complete logout and the user thinks they are signing in again based on Apple's wording.

Regardless, the UX of this is not great especially since the user is interrupted and can prevent normal session flow from proceeding by canceling the operation.

Starting in beta seed 4 of iOS 17, Apple adjusted this prompt to be significantly different. The prompt now has the following attributes:

Title: Do you want "Example App" to also sign in to "example.com" in Safari?<br>
Text: This allows "Example App" and "example.com" in Safari to share information about you such as your account. "Example App" will work without this.<br>
Option 1: Cancel<br>
Option 2: Sign In to "Example App" & Safari<br>
Option 3: Only Sign In to "Example App"<br>

This new alert allows for an instance of `ASWebAuthenticationSession` to be configured by iOS with an app sandbox rather than getting the system data store. This option is not available to developers programmatically, only the ability to use the system and an ephemeral data store. However, the phrasing Apple provides (again with no app developer input) is still problematic. 

First, this alert is far more technical than the original and will confuse end users and second, the implication that the app will work without sharing data with Safari is disingenuous. For SSO purposes, the session data is not available if the user selects the 3rd option and may end up logging a support ticket because it didn't sign them in as they'd expect. Additionally, they can footgun themselves during reauthentication by selecting a different option. This means that the session data is not present, or, could be referencing a different session. Both can lead to errors in an IDP (and potentially weird edge cases in session management within the app) and then the user logs a support ticket.

Since we don't want end users to cause themselves issues, I logged FB12809622 in order to suggest Apple allowing developers to handle this more gracefully (e.g. MDM or additional app provided context). I did not receive a response in the affirmative, but as of the GA release of iOS 17, it looks like Apple reverted the behavior for now so I suspect they are reworking this new implementation.

It's not that I don't understand what they're wanting to do, rather, their initial approach did not take everything in the identity ecosystem into account. Hopefully the next iteration is much better.

---

Speaking of authentication oddities, if an application is locked down in Single App Mode, using the remote Safari APIs does not work as iOS prevents the remote view controllers from starting with a cryptic error. I logged FB8807837 and Apple's response was this is working as intended. That's really unfortunate since the fallback is to create an embedded web view and many IDPs block that since it is insecure. In my opinion, remote view controllers (especially Safari) should be allowed in Single App Mode since the user is not leaving the locked down app.
