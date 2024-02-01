---
title: "Monitoring Transaction Performance"
subtitle: ""
date: 2024-01-31T19:20:00-05:00
tags: ["network", "performance"]
---

Apple provides the [`MetricKit`](https://developer.apple.com/documentation/metrickit?language=objc) framework for developers to periodically get diagnostic data about how the app is performing and how it is using resources. One of those metrics is [`MXNetworkTransferMetric`](https://developer.apple.com/documentation/metrickit/mxnetworktransfermetric?language=objc), but it only provides an overview of how many bytes the app has transferred. While Apple provides an API for custom metrics, the implementation limitations are enough such that creating custom metrics for transactions wouldn't work. Rather, you'd need to create a custom collector of sorts leveraging the data from [`NSURLSessionTaskTransactionMetrics`](https://developer.apple.com/documentation/foundation/nsurlsessiontasktransactionmetrics).


For every transaction (which includes redirects), the [URL loading system](https://developer.apple.com/documentation/foundation/url_loading_system?language=objc) collects many valuable pieces of data regarding how the system handled the request and performance characteristics. For just looking at performance, we need to look at the `fetchstartDate`, `domainLookupStartDate`, `domainLookupEndDate`, `connectStartDate`, `secureConnectionStartDate`, `secureConnectionEndDate`, `connectEndDate`, `requestStartDate`, `requestEndDate`, `responseStartDate`, & `responseEndDate`. 

These were presented in this order as it is the order of how a connection is used for a transfer. More specifically, we can use these values to get:

* domainLookupEndDate - domainLookupStartDate – the total amount of time spent using DNS.
* connectEndDate - connectStartDate – the total amount of time spent on TCP socket establishment.
* secureConnectionEndDate - secureConnectionStartDate – the total amount of time spent on TLS negotiation.
* responseStartDate - requestStartDate – how much time it took for the request to get to the client.

Having your [`NSURLSessionTaskDelegate`](https://developer.apple.com/documentation/foundation/nsurlsessiontaskdelegate?language=objc) implement [`URLSession:task:didFinishCollectingMetrics:`](https://developer.apple.com/documentation/foundation/nsurlsessiontaskdelegate/1643148-urlsession?language=objc) lets you capture this data and you can then store it off for upload to a remote endpoint for data analytics.

Of course, you may also have web content. In your web content, you can create an instance of [`PerformanceObserver`](https://developer.mozilla.org/en-US/docs/Web/API/PerformanceObserver) like such:

```javascript
const observer = new PerformanceObserver((metrics) => {
  metrics.getEntries().forEach((metric) => {
    // Do something with the data
  });
});

observer.observe({ type: "resource", buffered: true });
```

Each entry is an instance of [`PerformanceResourceTiming`](https://developer.mozilla.org/en-US/docs/Web/API/PerformanceResourceTiming) which has very similar attributes as the metric data from the URL loading system. As such, you can make the same calculations. Now if you need to transmit the data, you can send it via [`fetch`](https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API), but WebKit offers another option.

If you implement an object that conforms to the [`WKScriptMessageHandler`](https://developer.apple.com/documentation/webkit/wkscriptmessagehandler?language=objc) protocol (specifically [`userContentController:didReceiveScriptMessage:`](https://developer.apple.com/documentation/webkit/wkscriptmessagehandler/1396222-usercontentcontroller?language=objc)), and add it to the instance of [`WKWebView`](https://developer.apple.com/documentation/webkit/wkwebview?language=objc) by way of [`WKWebViewConfiguration`](https://developer.apple.com/documentation/webkit/wkwebviewconfiguration?language=objc) and its associated [`WKUserContentController`](https://developer.apple.com/documentation/webkit/wkusercontentcontroller?language=objc) with the [`addScriptMessageHandler:contentWorld:name:`](https://developer.apple.com/documentation/webkit/wkusercontentcontroller/3585112-addscriptmessagehandler?language=objc) method, the web content can call `window.webkit.messageHandlers.<name>.postMessage(<messageBody>)` and the native layer will be able to process that data and send it using the URL loading system instead.

Lastly, if you need to test connections outside of the application, you can use cURL.

For most cases, you can execute something along the lines of 

```bash
curl -vsi --trace-time -o /dev/null https://www.example.com
```

to get informative output, but, performance data is also available. Instead, you can execute this:

```bash
curl -svo /dev/null https://example.com/ -w "\nHTTP Code: %{http_code} \
\nHTTP Connect:%{http_connect} \
\nNumber Connects: %{num_connects} \
\nNumber Redirects: %{num_redirects} \
\nRedirect URL: %{redirect_url} \
\nSize Download: %{size_download} \
\nSize Upload: %{size_upload} \
\nTLS Verify: %{ssl_verify_result} \
\nTime Handshake: %{time_appconnect} \
\nTime Connect: %{time_connect} \
\nDNS Lookup Time: %{time_namelookup} \
\nTime Pretransfer: %{time_pretransfer} \
\nTime Redirect: %{time_redirect} \
\nTime Start Transfer: %{time_starttransfer} \
\nTime Total: %{time_total} \
\nEffective URL: %{url_effective}\n" 2>&1
```

If you need additional information, the `-w` flag has all of its configuration [documented](https://curl.se/docs/manpage.html#-w).
