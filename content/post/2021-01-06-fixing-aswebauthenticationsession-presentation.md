---
title: "Fixing ASWebAuthenticationSession Presentation"
subtitle: ""
date: 2021-01-06T07:00:00-06:00
tags: ["aswebauthenticationsession"]
---

[ASWebAuthenticationSession](https://developer.apple.com/documentation/authenticationservices/aswebauthenticationsession?language=objc) uses a technology limited for Apple use called remote view controllers: 

```
<SFAuthenticationViewController 0x7f9d21082200>, state: appeared, view: <SFSafariView 0x7f9d21a04da0>, presented with: <_UIFullscreenPresentationController 0x7f9d21a04b10>
  | <SFBrowserRemoteViewController 0x7f9d21075c00>, state: appeared, view: <_UISizeTrackingView 0x7f9d1f5658e0>
```

Additionally, you do not get access to the view controller created when you start an authentication session, rather, you only get the opaque session object to retain until authentication is complete. This leads to an interesting problem where the controller is being managed exclusively by Apple and it attempts to adapt its presentation to the application's content and it chooses something that does not fit your needs. An example of this is on iPad and you have some controller presented as a form sheet and then ASWebAuthenticationSession presents Safari as a form sheet as well (instead of full screen) which is suboptimal for the user as the view is quite small.

To correct this, we can use the Objective-C runtime to override the [`modalPresentationStyle`](https://developer.apple.com/documentation/uikit/uiviewcontroller/1621355-modalpresentationstyle?language=objc) to the desired style. A simple way of doing this is to create a category on ASWebAuthenticationSession and then swap out the methods on [`initialize`](https://developer.apple.com/documentation/objectivec/nsobject/1418639-initialize?language=objc):

```obj-c
@import ObjectiveC.runtime;

@interface UIViewController (Override)

- (UIModalPresentationStyle)fix_overrideModalPresentationStyle;

@end

@implementation UIViewController (Override)

- (UIModalPresentationStyle)fix_overrideModalPresentationStyle
{
  return UIModalPresentationFullScreen;
}

@end

@implementation ASWebAuthenticationSession (Override)

+ (void)initialize
{
  /**
   * Done in +(void)initialize so that we do not disrupt application launch performance.
   * Starting with iOS 13, ASWebAuthenticationSession presents incorrectly even though it is starting on a UIModalPresentationFullScreen view controller (with that controller's parent being UIModalPresentationStyleFormSheet on iPad).
   * Under the covers, ASWebAuthenticationSession presents a SFAuthenticationViewController (subclass of SFSafariViewController) and it leverages a SFAuthenticationViewControllerPresentationDelegate to figure out where it should present.
   * This is where the defect lies (due to UIModalPresentationAutomatic), so if we override the modalPresentationStyle method, we can enforce the presentation style we want at all times (i.e. ignoring what the delegate wants the view controller to do).
   * We construct the class name inconspicuously to avoid private API detection and then swizzle the method leveraging our override method that always returns UIModalPresentationFullScreen.
  */
  static dispatch_once_t onceToken;
  dispatch_once(&onceToken, ^{
    NSString *inconspicuousString = [@[[NSStringFromClass([SFSafariViewController class]) substringWithRange:NSMakeRange(0, 2)], [NSStringFromClass([ASWebAuthenticationSession class]) substringWithRange:NSMakeRange(5, 14)], [NSStringFromClass([UIViewController class]) substringWithRange:NSMakeRange(2, 14)]] componentsJoinedByString:@""];
    Class theClass = NSClassFromString(inconspicuousString);
    SEL originalSelector = @selector(modalPresentationStyle);
    SEL replacementSelector = @selector(fix_overrideModalPresentationStyle);
    Method originalMethod = class_getInstanceMethod(theClass, originalSelector);
    Method replacementMethod = class_getInstanceMethod([UIViewController class], replacementSelector);
        
    BOOL didAdd = class_addMethod(theClass, originalSelector, method_getImplementation(replacementMethod), method_getTypeEncoding(replacementMethod));
    if (didAdd) {
      class_replaceMethod(theClass, replacementSelector, method_getImplementation(originalMethod), method_getTypeEncoding(originalMethod));
    }
    else {
      method_exchangeImplementations(originalMethod, replacementMethod);
    }
  });
}

@end
```

And voilà, we now have a modicum of control over the view controller used for authentication and we get a consistent user experience.

---

I logged this as FB6446524, but Apple chose not to resolve the issue as they stated it was working as intended.
