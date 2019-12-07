---
title: "UI Testing ASWebAuthenticationSession"
subtitle: ""
date: 2019-12-07T07:00:00-06:00
tags: ["xcuitest", "aswebauthenticationsession", "automation"]
---

[UI testing](https://developer.apple.com/library/archive/documentation/DeveloperTools/Conceptual/testing_with_xcode/chapters/09-ui_testing.html) provides additional opportunities to validate the correctness of your application. One challenge though is that sometimes, certain workflows interrupt the test execution, for example, requesting camera permission. To solve this, Apple provides the [interruption handling API](https://developer.apple.com/documentation/xctest/xctestcase/handling_ui_interruptions?language=objc). However, there is a small problem when [ASWebAuthenticationSession](https://developer.apple.com/documentation/authenticationservices/aswebauthenticationsession?language=objc) is in play.

For some reason, when the ASWebAuthenticationSession prompt occurs, the interruption handler needs some help realizing that the prompt is blocking the application. To do this, you need to add a superfluous tap to the application. The result looks something like this:

```obj-c
[self addUIInterruptionMonitorWithDescription:@"Acknowledge Safari Permission Prompt" handler:^BOOL(XCUIElement *_Nonnull interruptingElement) {
  XCUIElement *continueButton = interruptingElement.buttons[@"Continue"];
  [continueButton tap];
  return YES;
}];

[self.currentApplication tap];
```

However (again), I wasn't able to get the interruption handler to work reliably. So, you need to use an alternative: SpringBoard. By adding an application reference to the SpringBoard process, you can get the actual element directly when the prompt is shown to the user:

```obj-c
XCUIApplication *springboard = [[XCUIApplication alloc] initWithBundleIdentifier:@"com.apple.springboard"];
XCUIElement *ssoAlert = springboard.alerts.firstMatch;
(void)[ssoAlert waitForExistenceWithTimeout:10];
[ssoAlert.buttons[@"Continue"] tap];
```

Now, the code above takes the easy way out and grabs the first alert, but, the query can be optimized ot grab the element that matches the title of the alert (based off of your application name) or the message contents (based off of the URL being requested). Unfortunately, this approach hasn't made it to Apple's official documentation, but, this is the recommended approach by Apple (I learned this from a Feedback ticket).

---

UI testing is usually pretty straightforward, but, sometimes you run into APIs that don't work the way you think they should and you need to implement something outside of the box in order for your tests to work. Also, I learned that the UI recorder outputs invalid code when recording ASWebAuthenticationSession interactions due to a bug in UTF-8 escape sequences. Apple says that they will fix this in a future Xcode release.
