---
title: "Gathering Application Metrics"
subtitle: ""
date: 2019-09-29T09:00:00-05:00
tags: ["MetricKit"]
---

Typically, you need to purchase some SDK to gather application data, however, starting with iOS 13, Apple provides one for free: [MetricKit](https://developer.apple.com/documentation/metrickit?language=objc). To some, this solution may not provide as much *identifying* data as they want, but, it honors all user privacy settings and is more geared towards improving application performance than user data. Now, you don't actually have to implement this framework to get that data as Xcode 11 will provide graphs of certain data sets in the new Metrics tab, but if you do want to, setup is very simple.

## Getting Started

To begin receiving application metric data, you need to import MetricKit in your application's delegate and conform to the [MXMetricManagerSubscriber](https://developer.apple.com/documentation/metrickit/mxmetricmanagersubscriber?language=objc) protocol. Then, you need to add your delegate as a subscriber:

```obj-c
#import "AppDelegate.h"
@import MetricKit;

@interface AppDelegate () <MXMetricManagerSubscriber>
@end

@implementation AppDelegate

- (BOOL)application:(nonnull UIApplication *)application didFinishLaunchingWithOptions:(nullable NSDictionary<UIApplicationLaunchOptionsKey, id> *)launchOptions
{
  [MXMetricManager.sharedManager addSubscriber:self];  
  return YES;
}
```

## Processing Metrics

To conform to the subscription protocol, you need to implement the `didReceiveMetricPayloads:` method. Once implemented, you can query the [payload](https://developer.apple.com/documentation/metrickit/mxmetricpayload?language=objc) for individual kinds of metrics as well as [data](https://developer.apple.com/documentation/metrickit/mxmetadata?language=objc) about the payload itself. The types of metrics you can get are: [Cellular Network Data](https://developer.apple.com/documentation/metrickit/mxcellularconditionmetric?language=objc), [CPU Data](https://developer.apple.com/documentation/metrickit/mxcpumetric?language=objc), [Display Data](https://developer.apple.com/documentation/metrickit/mxdisplaymetric?language=objc), [GPU Data](https://developer.apple.com/documentation/metrickit/mxgpumetric?language=objc), [Location Activity Data](https://developer.apple.com/documentation/metrickit/mxlocationactivitymetric?language=objc), [Network Data](https://developer.apple.com/documentation/metrickit/mxnetworktransfermetric?language=objc), [App Launch Data](https://developer.apple.com/documentation/metrickit/mxapplaunchmetric?language=objc), [App Responsiveness Data](https://developer.apple.com/documentation/metrickit/mxappresponsivenessmetric?language=objc), [App Usage Data](https://developer.apple.com/documentation/metrickit/mxappruntimemetric?language=objc), [App Memory Data](https://developer.apple.com/documentation/metrickit/mxmemorymetric?language=objc), [Disk Usage Data](https://developer.apple.com/documentation/metrickit/mxdiskiometric?language=objc), & [Custom Signpost Data](https://developer.apple.com/documentation/metrickit/mxsignpostmetric?language=objc). 

Obviously, once you have the data you need you can inspect it as you see fit, but, it is easier to send the data to a server and then plug it into an analytics engine of some sort to get more out of it (as well as compare against historical data sets). Apple provides a convenience method on the payload and the metadata to convert it to JSON directly by calling `JSONRepresentation`.

Example:
```obj-c
- (void)didReceiveMetricPayloads:(nonnull NSArray<MXMetricPayload *> *)payloads
{
  for (MXMetricPayload *metric in payloads) {
    // Battery Metrics
    MXCellularConditionMetric *cellData = metric.cellularConditionMetrics;
    MXCPUMetric *cpuData = metric.cpuMetrics;
    MXDisplayMetric *displayData = metric.displayMetrics;
    MXGPUMetric *gpuData = metric.gpuMetrics;
    MXLocationActivityMetric *locationData = metric.locationActivityMetrics;
    MXNetworkTransferMetric *networkData = metric.networkTransferMetrics;
        
    // Performance Metrics
    MXAppLaunchMetric *appLaunchData = metric.applicationLaunchMetrics;
    MXAppResponsivenessMetric *appResponsivenessData = metric.applicationResponsivenessMetrics;
    MXAppRunTimeMetric *appTimeData = metric.applicationTimeMetrics;
    MXMemoryMetric *appMemoryData = metric.memoryMetrics;
        
    // Disk Metrics
    MXDiskIOMetric *diskData = metric.diskIOMetrics;
        
    // Custom Metrics
    for (MXSignpostMetric *customMetric in metric.signpostMetrics) {
      // Handle custom data
    }
        
    // Payload Metadata
    MXMetaData *metadata = metric.metaData;
    NSDate *startDate = metric.timeStampBegin;
    NSDate *endDate = metric.timeStampEnd;
    BOOL containsMultipleVersions = metric.includesMultipleApplicationVersions;
    NSString *latestAppVersion = metric.latestApplicationVersion;
        
    NSData *json = [metric JSONRepresentation];
    // Send data to server
  }
}
```

## The Data

The actual metric data is in the form of a [measurement](https://developer.apple.com/documentation/foundation/nsmeasurement?language=objc), [average](https://developer.apple.com/documentation/metrickit/mxaverage?language=objc), and/or [histogram](https://developer.apple.com/documentation/metrickit/mxhistogram?language=objc). However, measurements are key parts of both the averages and histograms. Since the API uses [dimension](https://developer.apple.com/documentation/foundation/nsdimension?language=objc) units, you get a lot of stuff for free like [conversions](https://developer.apple.com/documentation/foundation/nsunitconverter?language=objc) and [formatting](https://developer.apple.com/documentation/foundation/nsmeasurementformatter?language=objc). With those conveniences, you can customize the data for logging purposes or create your own customized JSON payload.

---

Overall, the MetricKit API is very well put together and should help developers improve their applications in regard to performance and energy usage. However, I would like an API to receive crash reports, including system terminations (e.g. Jetsam events), directly as that would allow developers to also drop SDKs that charge for crash reporting.

