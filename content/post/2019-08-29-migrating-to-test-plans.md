---
title: "Migrating to Test Plans"
subtitle: ""
date: 2019-08-29T18:00:00-05:00
tags: ["schemes", "test plans"]
---

Comprehensive tests are an important part of delivering a quality product. However, the amount of configuration and duplication of work grows with the number of devices you need to test your UI in, get screenshots for, and then you need to multiply that by all of the languages you support. Prior to Xcode 11, you had to use [Schemes](https://help.apple.com/xcode/mac/10.2/#/dev0bee46f46) and [Xcode build configuration](https://help.apple.com/xcode/mac/10.2/#/dev745c5c974) files to manage your project and run all configurations. Now, there are Test Plans.

## Schemes

Schemes allow you to specify what targets are built (and in what order), run any pre or post build scripts, configure diagnostics, and override any launch arguments (and the environment) for testing. The limit of schemes though is that you can only build one at a time. Luckily, they are very easy to duplicate (Xcode provides this option), so you can create schemes with each specific config that you need and then loop through them in a script and store off each result. This isn't ideal as you can end up with many schemes and then you need to remember to modify each one if you need to make a change.

## Xcode Build Configuration Files

The `xcconfig` files allow you to specify configuration values that you normally manage in Xcode project configuration menus (including user defined values). This allows you to provide specific overrides when required rather than create more schemes or to provide overrides to specific schemes. This is extremely convenient for CI when you may need to mock endpoints or provide different data for testing.

## Test Plans

Test Plans are a new concept in Xcode 11 that takes some of the responsibilities of schemes and config files and combines them into a singular file. To start using them, you can go to Manage Schemes and then click Convert to Test Plans. This allows you to duplicate any existing schemes into test plans or create new ones from scratch. Once you have a test plan, you can modify the shared configuration and then create sub-configurations to override specific values. This allows you to have one test plan that encompasses all of your configurations (selectable in Xcode by Option clicking the test diamond or by passing the test plan into `xcodebuild`). When it is built, Xcode builds all of the variants and then combines the results into a new Xcode result bundle. To take advantage of this, it is recommended to run the tests in parallel or in parallel on multiple different devices. 

---

Test plans reduce the hassle of managing many different configurations across multiple schemes and/or config files when testing multiple configurations on multiple devices. This is a big improvement for developer productivity as Xcode's test plan editor has nice menus and can point out invalid config and then the files themselves are easily diff-able as they are stored as JSON. Hopefully, you find them easy to use and valuable in your projects.
