---
title: "Controlling Test Executions"
subtitle: ""
date: 2019-02-28T18:00:00-06:00
tags: ["tests", "monitoring"]
---

When running unit tests (and UI tests), [XCTest](https://developer.apple.com/documentation/xctest?language=objc) provides a few ways of interacting with the test runner. The most common scenario for interacting with your own tests is to provide common code to run before tests start and when they finish to ensure that each test has a clean environment.

### Setup and Tear Down

[XCTestCase](https://developer.apple.com/documentation/xctest/xctestcase?language=objc) provides APIs to modify the test environment before and after all tests have executed, but also before and after individual tests have run. Also, since `XCTestCase` is a class, you can subclass it and have your individual test files all use the same subclass to get the same setup and tear down implementations.

To actually implement common code that runs before or after all tests, you need to override the class level [setUp](https://developer.apple.com/documentation/xctest/xctestcase/1496262-setup?language=objc) and [tearDown](https://developer.apple.com/documentation/xctest/xctestcase/1496280-teardown?language=objc) methods. For individual tests, override the instance level [setUp](https://developer.apple.com/documentation/xctest/xctest/1500341-setup?language=objc) and [tearDown](https://developer.apple.com/documentation/xctest/xctest/1500463-teardown?language=objc) methods. That's all and good, but what if you need to do some cleanup of test resources that are not globally available? Enter [addTearDownBlock:](https://developer.apple.com/documentation/xctest/xctestcase/2887226-addteardownblock?language=objc).

These blocks can be registered in the instance level `setUp` but also in the test methods themselves and they are called before the `tearDown` instance method, making them good for cleaning up resources that must be removed before any other tests execute or other test code executes (e.g. files or database state/connections) and are also good for containing code that may throw exceptions since you can scope try/catch statements per block. There are a couple of things to keep in mind when using these blocks:

1. They are executed serially in Last-In, First-Out (LIFO) order.
2. They can be registered from any thread, but always execute on the main thread.
3. The blocks will execute (and `tearDown` methods) regardless if the test case sets [continueAfterFailure](https://developer.apple.com/documentation/xctest/xctestcase/1496260-continueafterfailure?language=objc) to `NO`.

Putting these options together, the test execution order of operations looks like this:

1. Class level `setUp`.
2. Instance level `setUp`.
3. Test Method.
4. Tear Down Blocks (LIFO).
5. Instance level `tearDown`.
6. Steps 2-5 repeat per test.
7. Class level `tearDown`.

### Test Monitoring

XCTest provides another avenue of being able to run code during the test process via the [XCTestObservation](https://developer.apple.com/documentation/xctest/xctestobservation?language=objc) protocol. The easiest way of of implementing this is to create a class that conforms to this protocol and then register it as the [principal class](https://developer.apple.com/documentation/bundleresources/information_property_list/nsprincipalclass?language=objc) of the test bundle. The reason to do this is that the principal class is automatically instantiated when XCTest loads the test bundle, so you don't have to worry about when to initialize the class yourself. However, you still need to register the class as an observer as that is not done automatically. The easiest way of doing that is to [add](https://developer.apple.com/documentation/xctest/xctestobservationcenter/1428317-addtestobserver?language=objc) your class as an observer to the [shared observation center](https://developer.apple.com/documentation/xctest/xctestobservationcenter/1428315-sharedtestobservationcenter?language=objc) in its [load](https://developer.apple.com/documentation/objectivec/nsobject/1418815-load?language=objc) or [initialize](https://developer.apple.com/documentation/objectivec/nsobject/1418639-initialize?language=objc). But what does it do?

The test observation protocol allows you to respond to test bundle, test suite, and test case events (including failures):

1. [testBundleWillStart:](https://developer.apple.com/documentation/xctest/xctestobservation/1500772-testbundlewillstart?language=objc) is called once (per bundle) when a test bundle is about to start.
2. [testSuiteWillStart:](https://developer.apple.com/documentation/xctest/xctestobservation/1501016-testsuitewillstart?language=objc) is called once (per suite) when a test suite is about to start.
3. [testCaseWillStart:](https://developer.apple.com/documentation/xctest/xctestobservation/1500527-testcasewillstart?language=objc) is called once (per case) when a test case is about to start.
4. [testCase:didFailWithDescription:inFile:atLine:](https://developer.apple.com/documentation/xctest/xctestobservation/1500371-testcase?language=objc) is called zero or more times (per case) at any point during test case execution if there are any failures.
5. [testCaseDidFinish:](https://developer.apple.com/documentation/xctest/xctestobservation/1500326-testcasedidfinish?language=objc) is called once (per case) when a test case is done executing.
6. [testSuite:didFailWithDescription:inFile:atLine:](https://developer.apple.com/documentation/xctest/xctestobservation/1500831-testsuite?language=objc) is called zero or more times at any point during test suite execution if there are any failures.
7. [testSuiteDidFinish:](https://developer.apple.com/documentation/xctest/xctestobservation/1500958-testsuitedidfinish?language=objc) is called once (per suite) when a test suite is done executing.
8. [testBundleDidFinish:](https://developer.apple.com/documentation/xctest/xctestobservation/1500819-testbundledidfinish?language=objc) is called once (per bundle) when a test bundle is done executing. Note: since the process will exit after this method returns, any asynchronous work needs to block until completion.

The main difference between this and the `XCTestCase` APIs is that no subclassing is involved nor overrides of methods. However, since the observation is as tightly integrated with each individual test suite/test case, you typically have to respond in a generic fashion, but you do get access to additional life cycle steps. Another thing to keep in mind though is that if there are multiple observers registered for events, the order of delivery is not guaranteed (i.e. the order of adding observers does not translate to event delivery).

---
 
Using custom test case subclasses as well as a test observer allows you to interact with the test run time in ways that allow you to completely control the test environment. Hopefully, you can use these tools to make your tests more stable, but also more efficient. 
