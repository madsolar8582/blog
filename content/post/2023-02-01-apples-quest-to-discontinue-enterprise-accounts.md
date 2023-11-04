---
title: "Apple's Quest to Discontinue Enterprise Accounts"
subtitle: ""
date: 2023-02-01T17:30:00-05:00
tags: ["apple", "developer program", "abm", "asm", "enterprise"]
---

At $company, it was time for our yearly renewal of the [Apple Developer Enterprise Program](https://developer.apple.com/programs/enterprise/). Instead of just handing them money, this year we were forced to get approval to allow us to renew our membership. This process included filling out forms with lots of questions about the number of developers we have, the security around our internal applications, the number of devices we have, and the kind of apps we produce in-house. After that, we had to submit photos of identification documents to prove that we were real. Luckily, we were approved, but this whole process really shows that Apple wants to gut the program.

You may be asking why they would want to do this and the reason, as usual, comes down to control. In the past, there have been a few high profile breaches of the agreement ([Facebook](https://arstechnica.com/gadgets/2019/01/facebook-and-google-offered-gift-cards-for-root-level-access-to-ios-users-data/) & [Google](https://arstechnica.com/gadgets/2019/01/apple-shuts-down-googles-internal-ios-apps-just-like-facebook/)), which led to Apple starting to change how the program worked. Prior to 2022-ish, most of these changes have been small (licensing) and additional scrutiny on who could apply. However, the recent pushes now leverage additional messaging about Apple's "better" replacements: [Apple Business Manager](https://support.apple.com/guide/apple-business-manager/welcome/web), [Apple School Manager](https://support.apple.com/guide/apple-school-manager/welcome/web), and [Unlisted App Distribution](https://developer.apple.com/support/unlisted-app-distribution/).

These programs, however, all require the App Review process (which the Enterprise program does not). Ergo, a lot of internal applications/tools would simply not meet the criteria and getting VPN connections/test credentials for an App Reviewer might be impossible due to corporate rules. Apple's "solution" to this is [TestFlight](https://developer.apple.com/testflight/), but that has problems (including App Review being required for initial builds) as well. The first being that apps cannot live in TestFlight for eternity, i.e. they must eventually ship to the App Store. This does not work for internal apps/tools. The second being that, to the best of my knowledge, you cannot have multiple active testing tracks and that is also limited to bundle identifier so you can't have dev, staging, preprod, etc. Only production (but pre-release). The third being that you are limited to 100 internal testers, so larger organizations can't adopt this. Simply, tools like App Center or via [Manifest](https://www.hexnode.com/blogs/how-to-distribute-apps-via-manifest-url/) are far superior as they support more use cases with less friction.

---

It is interesting to see Apple push this now as regulators all over the world are pushing for the App Store to become less controlling. The Enterprise Program, while still beholden to Apple to get entitlements, allows companies to develop and distribute apps without oversight (i.e. what governments are pushing for) and serves as an example of how the ecosystem _could_ function. Since the writing is on the wall, I really wonder what can convince Apple to lay off and let developers do what they need to do for their business and customers.