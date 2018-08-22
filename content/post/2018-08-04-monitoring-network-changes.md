---
title: "Monitoring Network Changes"
subtitle: ""
date: 2018-08-04T11:00:00-05:00
tags: ["reachability", "nwmonitor"]
---

Whether it is Wi-Fi or LTE, most people have near constant access to a high-speed network. However, there are times when you have either no network access or are on a degraded connection. Therefore, it is important to adjust the behavior of your application(s) to handle the change in network capabilities. Luckily, Apple provides a few ways of obtaining information about the network the device is connected to.

## The Old Way

The original framework of providing information about the device's network configuration is the [SystemConfiguration](https://developer.apple.com/documentation/systemconfiguration?language=objc) framework. In it, the [SCNetworkReachability](https://developer.apple.com/documentation/systemconfiguration/scnetworkreachability?language=objc) APIs provide a way of obtaining the status of the current network connection and whether or not a specific host or address is reachable via the network. There are some downsides to this framework as it us based off of Core Foundation objects, does not use blocks, and some of the required objects for the API don't have convenient methods for initialization. Because of this, Apple is wanting developers to switch to their replacement (below). Regardless, this API does provide some functionality that is not available in the replacement, so it is worth covering.

In order to get status updates from the system, you need to create a queue for callbacks as well as create a context object for the updates which allows you to pass Objective-C objects into a self provided C callback function. Once you have the prerequisites setup, you schedule the updates to occur on a [run loop](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/Multithreading/RunLoopManagement/RunLoopManagement.html) and then the system will send you updates. Once you want to stop receiving updates, you unschedule your reachability object from the run loop, clear the callback function, and then clear the callback queue.

### Example

In the following example, a sample class is created to listen for network updates and then post a notification containing the update information. This can be enhanced to do specific actions rather than post a notification update, provide a way to get the current status without a callback via [SCNetworkReachabilityGetFlags](https://developer.apple.com/documentation/systemconfiguration/1514924-scnetworkreachabilitygetflags?language=objc), and/or provide updates for an address rather than a host. However, because addresses are relevant to [DNS](https://en.wikipedia.org/wiki/Domain_Name_System) (or the local network), it is usually best to use a hostname rather than an address since the remote address can change or the network changes from [IPv4](https://en.wikipedia.org/wiki/IPv4) to [IPv6](https://en.wikipedia.org/wiki/IPv6) (or vice versa).

```obj-c
@import Foundation;
@import SystemConfiguration;

static void ExampleReachabiltiyCallback(SCNetworkReachabilityRef target, SCNetworkReachabilityFlags flags, void *_Nullable info)
{
    BOOL reachableViaWiFi = (flags & kSCNetworkReachabilityFlagsReachable) && !(flags & kSCNetworkReachabilityFlagsIsWWAN);
    BOOL reachableViaCellular = (flags & kSCNetworkReachabilityFlagsReachable) && (flags & kSCNetworkReachabilityFlagsIsWWAN);
    BOOL connectionRequired = (flags & kSCNetworkReachabilityFlagsConnectionRequired);
    BOOL connectionIsDynamic = (flags & kSCNetworkReachabilityFlagsConnectionRequired) && (flags & (kSCNetworkReachabilityFlagsConnectionOnTraffic | kSCNetworkReachabilityFlagsConnectionOnDemand));
    BOOL userActionRequired = (flags & kSCNetworkReachabilityFlagsConnectionRequired) && (flags & kSCNetworkReachabilityFlagsInterventionRequired);
    
    NSDictionary *userInfo = @{
                                @"reachableViaWiFi" : @(reachableViaWiFi),
                                @"reachableViaCellular" : @(reachableViaCellular),
                                @"connectionRequired" : @(connectionRequired),
                                @"connectionIsDynamic" : @(connectionIsDynamic),
                                @"userActionRequired" : @(userActionRequired),
                                @"hostname" : [(__bridge ExampleReachability *)info valueForKey:@"hostname"]
                             };
    NSNotification *notification = [NSNotification notificationWithName:@"com.example.reachability.status-change" object:nil userInfo:userInfo];
    
    dispatch_async(dispatch_get_main_queue(), ^{
      [NSNotificationCenter.defaultCenter postNotification:notification];
    });
}

@interface ExampleReachability ()

@property (nonatomic, copy) NSString *hostname;

@property (nonatomic, strong) __attribute__((NSObject)) SCNetworkReachabilityRef reachability;

@property (nonatomic, strong) dispatch_queue_t reachabilityQueue;

@end

@implementation ExampleReachability

- (void)dealloc
{
    [self stopMonitoringConnectivity];
    
    /**
     * Need to always nil out the Core Foundation objects since the NSObject attribute only applies to the getters/setters, not the backing iVar:
     * https://clang.llvm.org/docs/AutomaticReferenceCounting.html#retainable-object-pointers
     */
    self.reachability = nil;
}

- (void)startMonitoringConnectivityForHostName:(nonnull NSString *)hostname
{
    self.hostname = hostname;
    
    dispatch_queue_attr_t attrs = dispatch_queue_attr_make_with_qos_class(DISPATCH_QUEUE_SERIAL, QOS_CLASS_UTILITY, DISPATCH_QUEUE_PRIORITY_DEFAULT);
    self.reachabilityQueue = dispatch_queue_create("com.example.reachability-monitor", attrs);
    
    SCNetworkReachabilityContext reachabilityContext = { 0, (__bridge void *)self, NULL, NULL, NULL };
    self.reachability = CFAutorelease(SCNetworkReachabilityCreateWithName(kCFAllocatorDefault, hostname.UTF8String));

    SCNetworkReachabilitySetDispatchQueue(self.reachability, self.reachabilityQueue);
    SCNetworkReachabilitySetCallback(self.reachability, ExampleReachabiltiyCallback, &reachabilityContext);
    SCNetworkReachabilityScheduleWithRunLoop(self.reachability, CFRunLoopGetMain(), kCFRunLoopDefaultMode);
}

- (void)stopMonitoringConnectivity
{
    SCNetworkReachabilityUnscheduleFromRunLoop(self.reachability, CFRunLoopGetMain(), kCFRunLoopDefaultMode);
    SCNetworkReachabilitySetCallback(self.reachability, NULL, NULL);
    SCNetworkReachabilitySetDispatchQueue(self.reachability, NULL);
}

@end
```

## The New Way

At this year's WWDC (2018), Apple made their internal networking framework, [Network.framework](https://developer.apple.com/documentation/network?language=objc), public. The benefit of this framework is that it is all based off of a [user space](https://en.wikipedia.org/wiki/User_space) networking stack rather than being done in the kernel which means that it is more secure and more performant. The other benefit is that the APIs are ARC compatible and use blocks, so the implementation is much cleaner.

To obtain network status updates, you create a queue, create an instance of [nw_path_monitor](https://developer.apple.com/documentation/network/nw_path_monitor_t?language=objc), and then set a block as the handler for updates. Now, the information available via the API on [nw_path](https://developer.apple.com/documentation/network/nw_path_t?language=objc) is very different from the [SCNetworkReachabilityFlags](https://developer.apple.com/documentation/systemconfiguration/scnetworkreachabilityflags?language=objc), but you also get some information that isn't readily accessible from the older API.

### Example

Just like in the previous example, the update handler just posts a notification when the network changes and can be updated to take a specific action. The biggest difference though is that this API does not target a specific host or address, rather, it just monitors the network in general.

```obj-c
@import Foundation;
@import Network;

@interface ExampleNetworkMonitor ()

@property (nonatomic, strong) nw_path_monitor_t monitor;

@property (nonatomic, strong) dispatch_queue_t monitorQueue;

@end

@implementation ExampleNetworkMonitor

- (void)startNetworkMonitoring
{
    dispatch_queue_attr_t attrs = dispatch_queue_attr_make_with_qos_class(DISPATCH_QUEUE_SERIAL, QOS_CLASS_UTILITY, DISPATCH_QUEUE_PRIORITY_DEFAULT);
    self.monitorQueue = dispatch_queue_create("com.example.network.monitor", attrs);
    
    self.monitor = nw_path_monitor_create();
    nw_path_monitor_set_queue(self.monitor, self.monitorQueue);
    nw_path_monitor_set_update_handler(self.monitor, ^(nw_path_t _Nonnull path) {
        nw_path_status_t status = nw_path_get_status(path);
        BOOL isWiFi = nw_path_uses_interface_type(path, nw_interface_type_wifi);
        BOOL isCellular = nw_path_uses_interface_type(path, nw_interface_type_cellular);
        BOOL isEthernet = nw_path_uses_interface_type(path, nw_interface_type_wired);
        BOOL isExpensive = nw_path_is_expensive(path);
        BOOL hasIPv4 = nw_path_has_ipv4(path);
        BOOL hasIPv6 = nw_path_has_ipv6(path);
        BOOL hasNewDNS = nw_path_has_dns(path);
        
        NSDictionary *userInfo = @{
                                    @"isWiFi" : @(isWiFi),
                                    @"isCellular" : @(isCellular),
                                    @"isEthernet" : @(isEthernet),
                                    @"status" : @(status),
                                    @"isExpensive" : @(isExpensive),
                                    @"hasIPv4" : @(hasIPv4),
                                    @"hasIPv6" : @(hasIPv6),
                                    @"hasNewDNS" : @(hasNewDNS)
                                 };
        
        dispatch_async(dispatch_get_main_queue(), ^{
            [NSNotificationCenter.defaultCenter postNotificationName:@"com.example.network.status-change" object:nil userInfo:userInfo];
        });
    });
    
    nw_path_monitor_start(self.monitor);
}

- (void)stopNetworkMonitoring
{
    nw_path_monitor_cancel(self.monitor);
}

@end
```

## The Apple Way

The Apple mantra is to forego any pre-flight checks and just execute the operation and fail gracefully. However, those operations may be expensive so there is a need to be informed of whether or not the operation will succeed. To solve this problem, there is a [waitsForConnectivity](https://developer.apple.com/documentation/foundation/nsurlsessionconfiguration/2908812-waitsforconnectivity?language=objc) option on NSURLSessionConfiguration that allows the [URLSession:taskIsWaitingForConnectivity:](https://developer.apple.com/documentation/foundation/nsurlsessiontaskdelegate/2908819-urlsession?language=objc) method to be called. This method allows the networking operation to be queued for when the network regains connectivity and is automatically retried rather than failing immediately or allows you to cancel the operation if you believe that the operation would be stale by the time that connectivity returns.

---

With two different approaches to monitoring for network changes and a way to queue network requests until network connectivity returns, you can build very resilient applications and your users will thank you for it.
