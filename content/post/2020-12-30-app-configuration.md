---
title: "App Configuration"
subtitle: ""
date: 2020-12-30T09:00:00-05:00
tags: ["NSUserDefaults", "Settings Bundle", "MDM"]
---

When creating an app, you will most likely need to store user preferences or other configuration data to customize or control the behavior of the application. While this can be done server-side, local configuration is faster. Apple provides [NSUserDefaults](https://developer.apple.com/documentation/foundation/nsuserdefaults?language=objc) to store small configuration items between the app and its extensions. These items must be PList serializable (NSData, NSString, NSNumber, NSDate, NSArray, & NSDictionary), but helper methods exist for serializing and deserializing primitives (as well as NSURL).

Apple's recommendation is to have preferences and configuration in the app, but what if you need things before the app is launched? A Settings bundle can be added to an application that allows users to configure aspects of the application in the Settings app. These configuration values are mapped to NSUserDefaults and offer the user free text fields, sliders, toggles, and selectable lists. Values can be grouped and you can even offer hierarchical configuration views. Once your application launches, you read in the values from NSUserDefaults and you have your user configured preferences and configuration. One thing to note though is that some items in the Settings bundle need default values and those values need to be registered with NSUserDefaults via [`registerDefaults:`](https://developer.apple.com/documentation/foundation/nsuserdefaults/1417065-registerdefaults?language=objc) each time the application launches or else they won't work correctly.

Taking this a bit further, you can use iCloud to sync these items across multiple devices by using NSUserDefault's related class [NSUbiquitousKeyValueStore](https://developer.apple.com/documentation/foundation/nsubiquitouskeyvaluestore?language=objc). There are a few more restrictions placed on usage of this class (size limits, max number of items, etc), but you get it for free rather than having to leverage your own infrastructure. To keep the two data sources in sync, you can add KVO to NSUserDefaults and then update iCloud and then you can call [`synchronize`](https://developer.apple.com/documentation/foundation/nsubiquitouskeyvaluestore/1415989-synchronize?language=objc) on the iCloud backed data store and reconcile the two and then push the updates accordingly.

Lastly, if you are in an enterprise environment and know that the application is managed by an MDM, you have access to an additional option where administrators can provide the configuration you need and/or set preference values so that users can change them. If an app is MDM managed, an additional dictionary is available in NSUserDefaults called `com.apple.configuration.managed` (this is nil if not MDM managed). You can then pull values out of it set by the MDM where the keys and their respective value types are defined by you:

```obj-c
NSDictionary *managedPrefs = [NSUserDefaults.standardUserDefaults dictionaryForKey:@"com.apple.configuration.managed"];
```

---

Preferences and configuration are pretty easy to implement and you can even get iCloud based sync and MDM support. You probably still can use a server implementation for something like feature flags or capabilities, but if you are managed by an MDM, you can probably just use NSUserDefaults.
