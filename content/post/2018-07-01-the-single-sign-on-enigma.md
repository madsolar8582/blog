---
title: "The Single Sign-On Enigma"
subtitle: ""
date: 2018-07-01T08:00:00-05:00
tags: ["SSO"]
---

Let's say that you are building an application that needs some way of authenticating a user. Let's also say that you don't want to have your own user database and worry about keeping user information secure. This sounds like a perfect opportunity for [identity federation](https://en.wikipedia.org/wiki/Federated_identity). Identity federation allows you to offload the authentication of a user to the selected system and then the user can authorize your application to access their information. To accomplish this on iOS, you'll want to leverage Safari.

## Pre iOS 11

Prior to iOS 11, Apple recommends using [SFSafariViewController](https://developer.apple.com/documentation/safariservices/sfsafariviewcontroller?language=objc) (available in iOS 9+) to facilitate authentication with an accompanying [delegate](https://developer.apple.com/documentation/safariservices/sfsafariviewcontrollerdelegate?language=objc). As the name suggests, it is a view controller, so the API involves creating the instance and then presenting it. To handle the callback from the [identity provider](https://en.wikipedia.org/wiki/Identity_provider), you need to register a URL scheme in your app's Info.plist and then [handle it](https://developer.apple.com/documentation/uikit/core_app/allowing_apps_and_websites_to_link_to_your_content/communicating_with_other_apps_using_custom_urls?language=objc). In iOS 10 and 11, Apple added some additional enhancements such as the ability to tint the bars and controls in the view controller to match the displayed content, changing the style of the close button, and change the available activities the user has the option of selecting when the view controller is displayed.

```obj-c
- (void)launchSafariViewController
{
  SFSafariViewControllerConfiguration *safariConfig = [[SFSafariViewControllerConfiguration alloc] init];
  safariConfig.entersReaderIfAvailable = NO;
    
  SFSafariViewController *controller = [[SFSafariViewController alloc] initWithURL:[NSURL URLWithString:@"https://www.example.com/"] configuration:safariConfig];
  controller.delegate = self;
  controller.dismissButtonStyle = SFSafariViewControllerDismissButtonStyleCancel;
    
  [self presentViewController:controller animated:YES completion:nil];
}

- (NSArray<UIActivityType> *)safariViewController:(SFSafariViewController *)controller excludedActivityTypesForURL:(NSURL *)URL title:(nullable NSString *)title
{
  return @[
            UIActivityTypePostToFacebook,
            UIActivityTypePostToTwitter,
            UIActivityTypePostToWeibo,
            UIActivityTypeMessage,
            UIActivityTypeMail,
            UIActivityTypePrint,
            UIActivityTypeCopyToPasteboard,
            UIActivityTypeAssignToContact,
            UIActivityTypeSaveToCameraRoll,
            UIActivityTypeAddToReadingList,
            UIActivityTypePostToFlickr,
            UIActivityTypePostToVimeo,
            UIActivityTypePostToTencentWeibo,
            UIActivityTypeAirDrop,
            UIActivityTypeOpenInIBooks,
            UIActivityTypeMarkupAsPDF
        ];
}

- (void)safariViewControllerDidFinish:(SFSafariViewController *)controller
{
  // Check authentication status/handle cancellation
}

- (void)safariViewController:(SFSafariViewController *)controller didCompleteInitialLoad:(BOOL)didLoadSuccessfully
{
  // Check if launch succeeded
}
```

The code above results in the user authenticating in Safari while remaining in your application:
<center>
{{< figure src="/images/safariviewcontrollerexample.png" alt="Safari View Controller Example" height="600" width="400" >}}
</center>

## iOS 11

In iOS 11, Apple decided to make some changes regarding data storage between Safari and apps due to advertisers abusing APIs and tracking users without consent. This has greatly impacted more enterprise focused applications that implement SSO (we'll get to that later). The result of that decision is that Safari data is no longer shared with applications without user consent. This means that SFSafariViewController now only functions as a way to render web content in your app using Safari. To replace the authentication workflow capability, Apple introduced [SFAuthenticationSession](https://developer.apple.com/documentation/safariservices/sfauthenticationsession?language=objc) (after industry uproar in ß3 of iOS 11). The major difference between the old and new APIs is that the authentication session is no longer a view controller, rather, an opaque object that handles the remote Safari view. However, this process has the benefit of simplifying how much code you need to write and allows you to bypass Info.plist URL white-listing if you want by specifying the callback scheme directly (although you must keep a reference to the session object).

```obj-c
- (void)launchSafariAuthenticationSession
{
  SFAuthenticationSession *session = [[SFAuthenticationSession alloc] initWithURL:[NSURL URLWithString:@"https://www.example.com/"] callbackURLScheme:@"ssocallback://" completionHandler:^(NSURL *_Nullable callbackURL, NSError *_Nullable error) {
    // Process callback
  }];
    
  if (self.safariAuthSession) {
    [self.safariAuthSession cancel]; // Cancel existing session
  }
    
  self.safariAuthSession = session;
  [session start]; // Start process for end user
}
```

But... now a user has to acknowledge a prompt prior to interacting with Safari:
<center>
{{< figure src="/images/safariauthenticationprompt.png" alt="Safari Authentication Prompt" height="600" width="400" >}}
</center>

## iOS 12

This year, Apple has created the new [Authentication Services](https://developer.apple.com/documentation/authenticationservices?language=objc) framework and thus deprecated SFAuthenticationSession in favor of [ASWebAuthenticationSession](https://developer.apple.com/documentation/authenticationservices/aswebauthenticationsession?language=objc). There isn't really an API nor a functional difference between the two as it is essentially a renamed SFAuthenticationSession. The advantage though is that password managers (in addition to iOS with [iCloud Keychain](https://support.apple.com/en-us/ht204085)) can now populate credentials.

```obj-c
- (void)launchAuthenticationSession
{
  ASWebAuthenticationSession *session = [[ASWebAuthenticationSession alloc] initWithURL:[NSURL URLWithString:@"https://www.example.com/"] callbackURLScheme:@"ssocallback://" completionHandler:^(NSURL *_Nullable callbackURL, NSError *_Nullable error) {
    // Process callback
  }];
    
  if (self.authSession) {
    [self.authSession cancel];
  }
    
  self.authSession = session;
  [session start];
}
```

## The Issues

As mentioned above, the changes starting in iOS 11 to separate Safari and app data and require user consent prior to invoking Safari for authentication impact institutions more than consumer apps.

### Intelligent Tracking Prevention

In addition to the data separation, Apple introduced [Intelligent Tracking Prevention](https://webkit.org/blog/7675/intelligent-tracking-prevention/) (ITP) in Safari 11 (enhanced with [v1.1](https://webkit.org/blog/8142/intelligent-tracking-prevention-1-1/) in Safari 11.1). ITP uses machine learning (read: a non-industry standard mechanism) to determine what sites have the ability to track you with data they drop on sites and then partition them in such a way that they no longer function. For enterprise authentication solutions leveraging a [security token service](https://en.wikipedia.org/wiki/Security_token_service) (STS) for identity federation, this becomes a problem since the STS is supposed to track users (i.e. validate authentication state). Apple stated that to avoid the partitioning, apps can request that the user interact with the authentication domain directly every 24 hours (annoying end users) to reset this behavior. For an STS, usually the interaction is done via [CORS](https://en.wikipedia.org/wiki/Cross-origin_resource_sharing) requests, so that doesn't help, and now with Safari 12, ITP [v2.0](https://webkit.org/blog/8311/intelligent-tracking-prevention-2-0/) no longer has the 24 hour window.

### User Consent Prompt

As shown above, when attempting to use Safari for authentication purposes, the user must allow the application access to Safari. Now, in the consumer world, this makes sense since you are using an arbitrary application and possibly authenticating via a social network which may have a lot of personal data. However, in the enterprise space, it does not make sense since the institution has decided on behalf of all of their users that the application is allowed and that it should be granted all permissions automatically all the time.

The other issue with the prompt is that it is shown every time and that the application is not allowed to provide additional context. For example, when an application requests access to Face ID, there is a privacy string that the application can provide to provide a reason and context as to why that capability is being requested (not to mention that that prompt only occurs once). The prompt as it is now is too vague and can cause issues for non-technical users who may not understand that the application is requesting access to the underlying Safari data store to access the authentication cookie rather than to "share information about you" and since it is shown every time, users will get alert fatigue (not to mention get upset that there is a login confirmation after they already chose to log in each time).

## The Existing Solutions

Many members of various companies and authentication technologies have been providing Apple feedback on the user experience and technical limitations of the system that they have in place. Apple hasn't moved much, but there are various strategies for applications to implement.

### The Apple Solution - Storage Access

Apple has created a new API called [Storage Access](https://webkit.org/blog/8124/introducing-storage-access-api/) to facilitate cross origin authentication via some data (e.g. cookies). There are a few issues with this though:

1. Currently, Safari is the only browser to support this API (i.e. it is not industry standard). This means that vendors need to implement support for a non-standard feature in their products with possibly a very low ROI due to the market share of Macs and iOS devices. The other issue is that browser vendors and the W3C are already moving forward with a [different approach](https://fidoalliance.org/fido-alliance-and-w3c-achieve-major-standards-milestone-in-global-effort-towards-simpler-stronger-authentication-on-the-web/) to web authentication that Apple is not participating in (at this time).
2. It still has a user prompt associated with it, albeit, it is one time, but it is still disruptive to the authentication flow and does still does not allow the application/site to provide additional context.

### The Hacky Solution - WKWebView

Rather than use Safari for authentication (thus avoiding ITP and access prompts), you can use [WKWebView](https://developer.apple.com/documentation/webkit/wkwebview?language=objc) with your own custom UI. Once the user authenticates, you can [grab the cookie](https://developer.apple.com/documentation/webkit/wkhttpcookiestore/2882005-getallcookies?language=objc) and store it in a [shared cookie store](https://developer.apple.com/documentation/foundation/nshttpcookiestorage/1411361-sharedcookiestorageforgroupconta?language=objc) between your applications. As you may have guessed by the heading, this isn't great for a couple of reasons:

1. Since WKWebView has full access to all data on the page, malicious actors could steal information that they aren't supposed to have. Therefore, some vendors (and rightly so) have [disabled](https://developers.googleblog.com/2016/08/modernizing-oauth-interactions-in-native-apps.html) authentication in embedded contexts.
2. Since Safari is not being used and the cookie is stored via an app group, interacting with other applications outside of your developer account is not possible since the SSO context is only within your applications. Now, this may not be a huge issue for institutions that are only leveraging their own applications, but it does impact those who are using apps from other developers.

## The Proposals and Requests

There are other solutions to this usability problem that aren't possible yet as they require new API from Apple. These are a few of the ones that I have suggested:

### Access Prompt Enhanced

To improve the end user experience, it would go a long way to adjust the access prompt by:

1. Allowing applications to provide a custom message.
2. Make the prompt appear once per endpoint (and have a Settings section to make adjustments).

By doing these simple things, most of the issues with end users can be resolved with training and makes future authentications painless (although it does not resolve issues with ITP).

### SFSafariViewController and App Groups

Similar to the WKWebView suggestion above, if a developer can initialize a regular SFSafariViewController with an app group, it'll allow users to authentication in Safari (without the access prompt) while being able to share data between applications from the same developer. Now, this doesn't solve the integration problem with 3<sup>rd</sup> parties nor ITP, but it does get apps in a better place.

### MDM

An MDM can be used in a variety of ways to mitigate the various limitations imposed on the applications. By creating an MDM payload, bypasses/white-listing can be implemented in such a way that ITP will leave certain domains alone and access prompts can be skipped for those domains as well. For institution owned devices, this is a big win since they can now control the full experience and make sure that the applications they give their users will work. However, the trend towards [BYOD](https://en.wikipedia.org/wiki/Bring_your_own_device) may mean that the I.T. staff need to create two profiles (full and lite) so that the personal devices get a payload that only manages the appropriate settings.

### Password Manager Control

This is of particular interest for healthcare applications as there are various laws/regulations, policies, and [STIG](https://iase.disa.mil/stigs/app-security/app-security/Pages/index.aspx)s that prohibit the use of password autofill. With the requirement of ASWebAuthenticationSession, it becomes impossible to prevent a user from using a password manager to enter credentials. Therefore, it is necessary to add the ability to disable password manger extensions and iCloud Keychain for some apps. Luckily, Apple already provides an API to accomplish this via [UIApplicationDelegate](https://developer.apple.com/documentation/uikit/uiapplicationdelegate/1623122-application?language=objc) for 3<sup>rd</sup> party keyboards. All it needs is a new extension point identifier for password managers.

## Closing Thoughts

If looking to provide apps that want to provide a SSO experience on iOS, you may want to hold off a bit while Apple and the industry figures out the best approach to doing so. Prior to iOS 11, it used to be as simple as the process is on Android (via [Chrome](https://github.com/GoogleChrome/custom-tabs-client)) by using Safari, but that is no more. In order to move the process forward, [logging radars](https://bugreport.apple.com/) appears to be the only thing possible until Apple acquiesces to developers to provide them a better experience.
