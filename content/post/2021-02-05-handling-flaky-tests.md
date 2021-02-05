---
title: "Handling Flaky Tests"
subtitle: ""
date: 2021-02-05T08:00:00-06:00
tags: ["xctest", "tests"]
---

When running tests, sometimes you may discover that some tests just fail sporadically. This may be due to conditions on the test host or perhaps some unexpected downtime in a service. It could even be caused by using [test randomization](https://qualitycoding.org/random-test-order/). Regardless of the cause, the goal is to have all tests passing at all times and that does become difficult if there are external dependencies or resource issues. [XCTest](https://developer.apple.com/documentation/xctest?language=objc) provides some utilities to help us make robust tests that can handle this.

## Skipping

Let's say you have tests that only need to be run on certain hardware or if there is a certain configuration in place. Rather than making different test bundles (or test plans), you can have these tests live in a single place and use the test skipping feature. XCTest provides 3 options: a blanket skip, a skip if a condition is met, and a skip unless a condition is met. These are super simply to use, just call the macros in the test being executed (ideally at the start):

```obj-c
- (void)testSomething
{
  XCTSkip(@"This feature is not yet implemented");
}

- (void)testADifferentThing
{
  XCTSkipIf(UIDevice.currentDevice.userInterfaceIdiom == UIUserInterfaceIdiomPad, @"Only for iPhone");
  // Rest of the test
}

- (void)testAnotherThing
{
  XCTSkipUnless(UIDevice.currentDevice.userInterfaceIdiom == UIUserInterfaceIdiomPad, @"Only for iPad");
  // Rest of the test
}
```

## Expected Failures

While skipping is good for test execution control, Xcode 12.5 adds new APIs to handle tests that may fail. These APIs are good for marking tests that are known to fail and cannot be addressed right now as well as add some wiggle room for services (e.g. timeouts, downtime, etc). At the simplest level, you would call `XCTExpectFailure` and then the code executed after that function is allowed to fail. However, a gotcha is that if it does not fail, the test will be marked as a failure. To make this conditional, there is a variant: `XCTExpectFailureWithOptions`. This variant requires you to construct an instance of [`XCTExpectedFailureOptions`](https://developer.apple.com/documentation/xctest/xctexpectedfailureoptions?language=objc) to control the behavior. By setting the `strict` property to `NO`, you can allow a test that is expected to have a failure pass if the failure does not occur (you can also adjust `enabled` like you would if setting up a skip conditional). Now we have a way of dealing with flaky tests!

The thing to keep in mind though is that any code executed in that test after setting up the expectation for failure is allowed to fail. If you want to limit it to a portion of code, you need to use the variants that take blocks: `XCTExpectFailureInBlock` & `XCTExpectFailureWithOptionsInBlock`. Lastly, if you want to control the expected failure even more, you can set the `matcher` property on your failure options and examine the generated [XCTIssue](https://developer.apple.com/documentation/xctest/xctissue?language=objc) directly to see if it is what you expect to be the failure.

```obj-c
- (void)testSomething
{
  XCTExpectedFailureOptions *options = [[XCTExpectedFailureOptions alloc] init];
  options.strict = NO;
  XCTExpectFailureWithOptions(@"The service is down", options);
  // Rest of the test
}

- (void)testADifferentThing
{
  XCTExpectedFailureOptions *options = [[XCTExpectedFailureOptions alloc] init];
  options.strict = NO;
  XCTExpectFailureWithOptionsInBlock(@"The file system is full", options, ^{
    // Some work writing large data to disk
  });
  // Rest of the test
}

- (void)testAnotherThing
{
  XCTExpectedFailureOptions *options = [[XCTExpectedFailureOptions alloc] init];
  options.issueMatcher = ^BOOL(XCTIssue *_Nonnull issue) {
    return issue.sourceCodeContext.location.lineNumber == 100;
  };
    
  XCTExpectFailureWithOptions(@"Hit known bug to be fixed in a future release", options);
  // Rest of the test
}
```

---

As always, you need to be judicious when you apply these APIs in your tests because they can be abused to hide real issues. However, if used correctly, you will have a set of very stable (and flexible) tests.
