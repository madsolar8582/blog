---
title: "Asynchronous Testing"
subtitle: "Setting Expectations"
date: 2018-04-18T16:00:00-05:00
tags: ["concurrency", "parallelism", "testing"]
---

Writing concurrent code has many pitfalls and because of the inherit complexity, testing that code may also prove difficult. By leveraging [XCTest](https://developer.apple.com/documentation/xctest?language=objc), the [test cases](https://developer.apple.com/documentation/xctest/xctestcase?language=objc) default to synchronous tests (i.e. when the test hits the last line in scope, it ends). Now, some developers may try to solve this by adding [sleeps](https://developer.apple.com/documentation/foundation/nsthread/1413673-sleepfortimeinterval?language=objc) or manipulating the [run loop](https://developer.apple.com/documentation/foundation/nsrunloop?language=objc), but those methods are unreliable or can cause side effects. To properly handle parallel scenarios, XCTest provides an API that allows you to create expectations for the outcome of an asynchronous operation by adhering to the [XCTWaiterDelegate](https://developer.apple.com/documentation/xctest/xctwaiterdelegate?language=objc) protocol.

## XCTestExpectation
The simplest way of creating an asynchronous-aware test is to use [XCTestExpectation](https://developer.apple.com/documentation/xctest/xctestexpectation?language=objc). This is the base class for all of the other expectations and provides the ability to set a required number of fulfillment calls, create an inverted expectation (success if it isn't fulfilled), fail the test if too many fulfillments are triggered, and expectations can be set to only be successful if they are fulfilled in order.

Example:
```obj-c
- (void)someTest
{
  XCTestExpectation *expectation = [self expectationWithDescription:@"My Expectation"];
  [someObject doWorkWithCompletionHandler:^{
    [expectation fulfill];
  }];
    
  /**
  * This method waits on expectations created with XCTestCase's convenience methods only.
  * This method does not wait on expectations created manually via 
  * initializers on XCTestExpectation or its subclasses.
  * Timeout is in seconds.
  */
  [self waitForExpectationsWithTimeout:2 handler:^(NSError * _Nullable error) {
    // Test cleanup
  }];
}
```

Inverse Example:
```obj-c
- (void)someTest
{
  XCTestExpectation *expectation = [self expectationWithDescription:@"My Expectation"];
  expectation.inverted = YES;
    
  [someObject doWorkWithCompletionHandler:^{
    // Something failed, so no fulfill
  }];
    
  [self waitForExpectationsWithTimeout:2 handler:^(NSError * _Nullable error) {
    // Test cleanup
  }];
}
```

Multiple Fulfillment Example:
```obj-c
- (void)someTest
{
  XCTestExpectation *expectation = [self expectationWithDescription:@"My Expectation"];
  expectation.expectedFulfillmentCount = 3; // Default is 1
    
  [someObject doWorkWithCompletionHandler:^{
    [expectation fulfill];
  }];
  [someObject doWorkWithCompletionHandler:^{
    [expectation fulfill];
  }];
  [someObject doWorkWithCompletionHandler:^{
    [expectation fulfill];
  }];
    
  [self waitForExpectationsWithTimeout:2 handler:^(NSError * _Nullable error) {
    // Test cleanup
  }];
}
```

Over Fulfill Example:
```obj-c
- (void)someTest
{
  XCTestExpectation *expectation = [self expectationWithDescription:@"My Expectation"];
  /**
  * If set, calls to fulfill() after the expectation has already been fulfilled
  * Exceeding the fulfillment count will raise an exception. 
  * This is the legacy behavior of expectations created through APIs on XCTestCase
  * but is not enabled for expectations created using XCTestExpectation initializers.
  */
  expectation.assertForOverFulfill = YES;

  [someObject doWorkWithCompletionHandler:^{
    [expectation fulfill];
  }];
  [someObject doWorkWithCompletionHandler:^{
    [expectation fulfill]; // Exception will be thrown
  }];
    
  [self waitForExpectationsWithTimeout:2 handler:^(NSError * _Nullable error) {
    // Test cleanup
  }];
}
```

Ordering Example:
```obj-c
- (void)someTest
{
  XCTestExpectation *exp = [self expectationWithDescription:@"My Expectation"];
  XCTestExpectation *exp2 = [self expectationWithDescription:@"My Expectation2"];

  [someObject doWorkWithCompletionHandler:^{
    [exp2 fulfill]; // Causes failure due to it completing first
  }];
  [someObject doWorkWithCompletionHandler:^{
    [exp fulfill];
  }];

  [self waitForExpectations:@[expectation, expectation2] timeout:2 enforceOrder:YES];
}
```

## XCTNSNotificationExpectation
For work that interacts with [NSNotificationCenter](https://developer.apple.com/documentation/foundation/nsnotificationcenter?language=objc), you can use [XCTNSNotificationExpectation](https://developer.apple.com/documentation/xctest/xctnsnotificationexpectation?language=objc). This type of expectation allows you to wait for a certain [notification](https://developer.apple.com/documentation/foundation/nsnotification?language=objc) to be fired from a certain sender and allows you to wait on another instance of a notification center that isn't the default.

Example:
```obj-c
- (void)someTest
{
  XCTestExpectation *expectation = [self expectationForNotification:@"com.ex.notification" 
                                                             object:someObject
                                                            handler:^BOOL(NSNotification *_Nonnull notification) {
    /**
    * If not provided, the expectation will be fulfilled by the first 
    * notification matching the specified name from the observed object.
    */
    // Return YES/NO based on introspection
  }];
    
  [someObject doWork];
    
  [self waitForExpectations:@[expectation] timeout:2];
}
```

Custom Notification Center Example:
```obj-c
- (void)someTest
{
  XCTNSNotificationExpectation *expectation = [[XCTNSNotificationExpectation alloc] 
                                                initWithName:@"com.example.notification" 
                                                object:someObject
                                                notificationCenter:someOtherCenter];
    
  [someObject doWork];
    
  [self waitForExpectations:@[expectation] timeout:2];
}
```

## XCTKVOExpectation
Another powerful feature of Objective-C is [KVO](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/KeyValueObserving/KeyValueObserving.html). Since KVO notifies an interested party in the event of a value change (thanks to [KVC](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/KeyValueCoding/index.html)), you can use [XCTKVOExpectation](https://developer.apple.com/documentation/xctest/xctkvoexpectation?language=objc) to wait until your work has completed with the expected result by specifying the [NSKeyValueObservingOptions](https://developer.apple.com/documentation/foundation/nskeyvalueobservingoptions?language=objc) you want.

Example:
```obj-c
- (void)someTest
{
  NSKeyValueObservingOptions options = NSKeyValueObservingOptionNew;
  XCTKVOExpectation *expectation = [[XCTKVOExpectation alloc] initWithKeyPath:@"somePath" 
                                                              object:someObject
                                                              expectedValue:someValue 
                                                              options:options];
    
  [someObject doWork];
    
  [self waitForExpectations:@[expectation] timeout:2];
}
```

## XCTNSPredicateExpectation
Finally, you can use the [NSPredicate](https://developer.apple.com/documentation/foundation/nspredicate?language=objc) API to [create](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/Predicates/AdditionalChapters/Introduction.html) logical conditions that can be tested for a `YES` or `NO` value. Therefore, you can create a [XCTNSPredicateExpectation](https://developer.apple.com/documentation/xctest/xctnspredicateexpectation?language=objc) that the system will test until timeout (or success) in order to wait until some operation(s) completes.

Example:
```obj-c
- (void)someTest
{
  NSPredicate *predicate = [NSPredicate predicateWithBlock:^BOOL(id _Nullable object, NSDictionary<NSString *,id> *_Nullable bindings) {
    // Perform logical validation
  }];

  XCTNSPredicateExpectation *expectation = [[XCTNSPredicateExpectation alloc] 
                                             initWithPredicate:predicate 
                                             object:someObject];
  
  [someObject doWork];

  [self waitForExpectations:@[expectation] timeout:2];
}
```

## Conclusion
Using expectations is an easy way of making good asynchronous-aware tests. Now, there are other ways of handling asynchronous code such as using [OCMock](http://ocmock.org/) to [stub](https://en.wikipedia.org/wiki/Test_stub) and then execute arbitrary [NSInvocation](https://developer.apple.com/documentation/foundation/nsinvocation?language-objc)s or to use the [Objective-C runtime](https://developer.apple.com/documentation/objectivec/objective_c_runtime?language=objc) to [swizzle](http://nshipster.com/method-swizzling/) methods to do something else during test executions. However, the expectation API allows you to avoid the complexity of adding a 3rd-party library and avoid the runtime while still being powerful.
