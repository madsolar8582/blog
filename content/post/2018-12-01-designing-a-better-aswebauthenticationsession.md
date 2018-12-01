---
title: "Designing a Better ASWebAuthenticationSession"
subtitle: ""
date: 2018-12-01T10:00:00-06:00
tags: ["aswebauthenticationsession", "sfauthenticationsession"]
---

As I have written [before](https://msolarana.netlify.com/2018/07/01/the-single-sign-on-enigma/), the API provided by Apple to implement SSO via Safari leaves a lot to be desired. One of the main concerns is that starting a session automatically prompts for permission and if the user cancels, it can leave the user in a weird state. On top of that, the permission allowance is not remembered, so alert fatigue becomes real. Therefore, I decided to look at other Apple APIs to see how permission onboarding occurs to find better implementations. I found that the HealthKit process for accessing health records and the API on AVFoundation to determine authorization status combined could be a good solution for the Safari problem.

The HealthKit process is defined in session [706](https://developer.apple.com/videos/play/wwdc2018/706/) of WWDC 2018. In it, there is a flow diagram that goes through an appropriate process of how access to sensitive data should be done:

1. Configure the application. For Safari, this could be a list of endpoints in the app's plist.
2. Call the authorization status API. This lets the app know what is and isn't authorized.
3. If authorization is required, call the request authorization API. This triggers an Apple controlled UI that explains to the user what is being requested and contains the app's explanation as to why the data is being requested. The user can also pre-approve new endpoints if they are added by an application update.
4. Once the request authorization API completes, the app receives what is and isn't authorized and can then decide how to proceed. More importantly, the authorization is remembered and is not required to be asked for again.
5. If authorization is granted, the actual desired API can be called to start the workflow. Otherwise, the app can transition to an alternative workflow.

The API for authorization status and request is defined in the AVFoundation [docs](https://developer.apple.com/documentation/avfoundation/cameras_and_media_capture/requesting_authorization_for_media_capture_on_ios?language=objc). Putting that into practice with the Safari API, you would get something like this:

```obj-c
typedef NS_ENUM(NSUInteger, ASWebAuthenticationSessionDomainAuthorizationStatus) {
  // Never been asked to use before
  ASWebAuthenticationSessionDomainAuthorizationStatusUndefined,
  // Can be used
  ASWebAuthenticationSessionDomainAuthorizationStatusAllowed,
  // Can not be used
  ASWebAuthenticationSessionDomainAuthorizationStatusDisallowed
};

+ (void)requestAuthorizationForEndpointsWithCompletionHandler:(void (^_Nonnull)(NSArray<NSURL *> *authorizedEndpoints))handler;

+ (ASWebAuthenticationSessionDomainAuthorizationStatus)authorizationStatusForEndpoint:(nonnull NSURL *)endpoint;
```

The `authorizationStatusForEndpoint:` method allows an app to check whether or not the endpoint that it wants to launch has been authorized. If authorization is required, the `requestAuthorizationForEndpointsWithCompletionHandler:` can be called and then the app gets a list of endpoints that the user has granted access to. 

Putting all of this together, the user would see something like this:

<center>
{{< figure src="/images/SafariInitialPrompt.png" alt="Safari Initial Prompt" height="600" width="400" >}}
{{< figure src="/images/SafariSecondaryPromptAllow.png" alt="Safari Secondary Prompt" height="600" width="400" >}}
{{< figure src="/images/SafariSecondaryPromptPermission.png" alt="Safari Secondary Prompt with Permission" height="600" width="400" >}}
</center>

---

The text I feel like needs a bit more finessing to explain something relatively technical to an end user, but I feel like this approach is far more friendlier and provides more context than the existing alert that Apple shows today. If you feel like Apple should implement this, log a [radar](https://bugreport.apple.com/) and reference mine: 36296263.
