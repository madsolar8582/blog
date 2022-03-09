---
title: "Resource Bundle Code Signing"
subtitle: ""
date: 2022-03-08T18:00:00-06:00
tags: ["Xcode", "CocoaPods"]
---

With Xcode 13.3, Apple introduced a new requirement for iOS applications to code sign any and all resource bundles. This has, apparently, been a requirement for Mac Catalyst for quite some time and has now made its way to the iOS platform. This is somewhat problematic for teams that consume resource bundles as part of a library/framework as the distributor cannot specify the code signing configuration on behalf of the consuming application. Luckily for CocoaPods, there is a workaround. Using a post install hook, you can update the code signing configuration of every resource bundle:

```ruby
post_install do | i |
  i.pods_project.targets.each do | t |
    if t.respond_to?(:product_type) && t.product_type == 'com.apple.product-type.bundle'
      t.build_configurations.each do | c |
        c.build_settings['DEVELOPMENT_TEAM'] = '<TEAM_ID>'
      end
    end
  end
end
```
For those that have more advanced code signing requirements, you can substitute or add additional fields (e.g. `CODE_SIGN_IDENTITY`). Lastly, if you have your own resource bundles in the app's Xcode project, you can set the code signing config directly in Xcode.

---

It is unfortunate that this was introduced in a minor Xcode release as this breaks a lot of setups. It is stranger still that resource bundles have to be code signed when they don't contain any executable code.
