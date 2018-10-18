---
title: "Missing Delegates"
subtitle: ""
date: 2018-10-18T18:00:00-05:00
tags: ["uiwebview", "wkwebview"]
---

When embedding web content in iOS applications, it is necessary to become the delegate of the web view displaying the content so that you can handle errors during the loading process. Depending on the UI you have implemented around said web view(s), you might also have navigation controls as well as a cancel/reload button. For UIWebView, the [webView:didFailLodWithError:](https://developer.apple.com/documentation/uikit/uiwebviewdelegate/1617970-webview?language=objc) method is used to tell the delegate that some error occurred during page load, whereas WKWebView uses both [webView:didFailNavigation:withError](https://developer.apple.com/documentation/webkit/wknavigationdelegate/1455623-webview?language=objc) & [webView:didFailProvisionalNavigation:withError:](https://developer.apple.com/documentation/webkit/wknavigationdelegate/1455637-webview?language=objc) to inform the delegate. The typical pattern here is for the application to either show an alert to the end user with an explanation to the user as to why they aren't seeing what they are expecting or to present a view over the web view's content view with content explaining the error. Since there is an API for these kinds of scenarios, it is safe to rely on them, right? Well, as I've found out, that is not a safe assumption.

On both iOS 11 and 12, I've discovered that if a web view is loading content and the network connection drops suddenly (e.g. Airplane Mode), the load is cancelled due to a loss of Internet connectivity, but the delegate methods are not called. If there is network monitoring in place with either [SCNetworkReachability](https://developer.apple.com/documentation/systemconfiguration/scnetworkreachability?language=objc) or [NWPathMonitor](https://developer.apple.com/documentation/network/nw_path_monitor_t?language=objc), those actively reflect the situation (as well as the CFNetwork diagnostic logging). But, for some reason, the delegate is not called which leaves the user in a bad spot. To work around this, you can implement a timer with a timeout of 30 seconds or so that if the finish load method is not called within that time, the user can be prompted to take an action. I've also found that using [NSURLSession](https://developer.apple.com/documentation/foundation/nsurlsession?language=objc) to trigger a dummy transaction can sometimes make the network stack aware of the current status and then the web view delegate method gets called. 

---

I've logged this as radar 45126508, so hopefully Apple addresses this soon™. Since both web views are impacted, I'm guessing that there is some common code/subsystem in use that is not functioning properly.
