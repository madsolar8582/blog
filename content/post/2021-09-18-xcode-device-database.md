---
title: "Xcode Device Database"
date: 2021-09-18T12:00:00-05:00
tags: ["xcode"]
---

When new device models are announced, one of the things that applications that identify what device they are running on need to do is update their model to device name mapping. When referring to device model, we are not talking about the [model](https://developer.apple.com/documentation/uikit/uidevice/1620044-model?language=objc) property on `UIDevice`, rather the underlying device model Apple assigns to each of its devices.

To get the model, you use the kernel via [uname](https://developer.apple.com/library/archive/documentation/System/Conceptual/ManPages_iPhoneOS/man3/uname.3.html):

```obj-c
#import <sys/utsname.h>

struct utsname sysInfo;
if (uname(&sysInfo) == 0) {
  NSString *model = @(sysInfo.machine);
  // Lookup model
}
else {
  // Handle error
}

```

The lookup needs a mapping of model identifiers to a more well understood value and that is where Xcode itself comes into play. Located inside of Xcode lies a SQLite database with information about all of the devices Xcode supports: `/Applications/Xcode.app/Contents/Developer/Platforms/iPhoneOS.platform/usr/standalone/device_traits.db`. Using a tool like [DB Browser for SQLite](https://sqlitebrowser.org/) lets you view the schema for the tables as well as the tables themselves. The `Devices` table has the information that we seek via the `ProductType` and `ProductDescription` columns. For example, the `iPhone13,3` identifier maps to the iPhone 12 Pro.

Now that you have your data source, you can create a method that takes in the device model string and return a string with the friendly name either by having the mappings hardcoded in the application or perhaps pulled from a server.
