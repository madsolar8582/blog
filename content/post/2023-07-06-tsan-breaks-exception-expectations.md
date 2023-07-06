---
title: "TSan Breaks Exception Expectations"
subtitle: ""
date: 2023-07-06T08:00:00-05:00
tags: ["xcode", "xctest", "tsan"]
---

Yesterday, Apple released Xcode 15 β3 (15A195k) and I noticed that my tests started failing. These tests verified that an expectation was thrown from calling a method that should be unavailable. However, these tests were failing with an uncaught exception:

```
xctest(8933,0x1f2049e00) malloc: nano zone abandoned due to inability to reserve vm space.
Test Suite 'ExampleClassTest' started at 2023-07-06 07:40:43.904.
Test Case '-[ExampleClassTest testInit]' started.
2023-07-06 07:40:43.907915-0500 xctest[8933:99736] *** Terminating app due to uncaught exception 'NSInternalInconsistencyException', reason: 'init is unavailable for class ExampleClass'
*** First throw call stack:
(
    0   CoreFoundation                      0x00000001971a3154 __exceptionPreprocess + 176
    1   libobjc.A.dylib                     0x0000000196cc24d4 objc_exception_throw + 60
    2   ExampleFramework                    0x000000010f091e1c -[ExampleClass init] + 348
    3   ExampleFrameworkTests               0x000000010f061bdc -[ExampleClassTest testInit] + 196
    4   CoreFoundation                      0x000000019710c7f4 __invoking___ + 148
    5   CoreFoundation                      0x000000019710c668 -[NSInvocation invoke] + 428
    6   XCTestCore                          0x0000000101dea1f4 +[XCTFailableInvocation invokeStandardConventionInvocation:completion:] + 68
    7   XCTestCore                          0x0000000101dea1a8 __90+[XCTFailableInvocation invokeInvocation:withTestMethodConvention:lastObservedErrorIssue:]_block_invoke_3 + 28
    8   XCTestCore                          0x0000000101de9900 __81+[XCTFailableInvocation invokeWithAsynchronousWait:lastObservedErrorIssue:block:]_block_invoke + 360
    9   XCTestCore                          0x0000000101da2a10 +[XCTSwiftErrorObservation observeErrorsInBlock:] + 280
    10  XCTestCore                          0x0000000101de9690 +[XCTFailableInvocation invokeWithAsynchronousWait:lastObservedErrorIssue:block:] + 228
    11  XCTestCore                          0x0000000101de9e90 +[XCTFailableInvocation invokeInvocation:withTestMethodConvention:lastObservedErrorIssue:] + 540
    12  XCTestCore                          0x0000000101dea298 +[XCTFailableInvocation invokeInvocation:lastObservedErrorIssue:] + 72
    13  XCTestCore                          0x0000000101ddb498 __24-[XCTestCase invokeTest]_block_invoke_2 + 88
    14  XCTestCore                          0x0000000101db1318 -[XCTMemoryChecker _assertInvalidObjectsDeallocatedAfterScope:] + 84
    15  XCTestCore                          0x0000000101dde974 -[XCTestCase assertInvalidObjectsDeallocatedAfterScope:] + 92
    16  XCTestCore                          0x0000000101ddb3f8 __24-[XCTestCase invokeTest]_block_invoke.96 + 172
    17  XCTestCore                          0x0000000101d99558 -[XCTestCase(XCTIssueHandling) _caughtUnhandledDeveloperExceptionPermittingControlFlowInterruptions:caughtInterruptionException:whileExecutingBlock:] + 168
    18  XCTestCore                          0x0000000101ddaf20 -[XCTestCase invokeTest] + 764
    19  XCTestCore                          0x0000000101ddc9a8 __26-[XCTestCase performTest:]_block_invoke.149 + 36
    20  XCTestCore                          0x0000000101d99558 -[XCTestCase(XCTIssueHandling) _caughtUnhandledDeveloperExceptionPermittingControlFlowInterruptions:caughtInterruptionException:whileExecutingBlock:] + 168
    21  XCTestCore                          0x0000000101ddc44c __26-[XCTestCase performTest:]_block_invoke.134 + 552
    22  XCTestCore                          0x0000000101dbd8e4 +[XCTContext _runInChildOfContext:forTestCase:markAsReportingBase:block:] + 180
    23  XCTestCore                          0x0000000101dbd7cc +[XCTContext runInContextForTestCase:markAsReportingBase:block:] + 104
    24  XCTestCore                          0x0000000101ddbea4 -[XCTestCase performTest:] + 308
    25  XCTestCore                          0x0000000101d84190 -[XCTest runTest] + 48
    26  XCTestCore                          0x0000000101dc0b7c -[XCTestSuite runTestBasedOnRepetitionPolicy:testRun:] + 68
    27  XCTestCore                          0x0000000101dc0a44 __27-[XCTestSuite performTest:]_block_invoke + 164
    28  XCTestCore                          0x0000000101dc04a8 __59-[XCTestSuite _performProtectedSectionForTest:testSection:]_block_invoke + 48
    29  XCTestCore                          0x0000000101dbd8e4 +[XCTContext _runInChildOfContext:forTestCase:markAsReportingBase:block:] + 180
    30  XCTestCore                          0x0000000101dbd7cc +[XCTContext runInContextForTestCase:markAsReportingBase:block:] + 104
    31  XCTestCore                          0x0000000101dc0418 -[XCTestSuite _performProtectedSectionForTest:testSection:] + 180
    32  XCTestCore                          0x0000000101dc06f8 -[XCTestSuite performTest:] + 220
    33  XCTestCore                          0x0000000101d84190 -[XCTest runTest] + 48
    34  XCTestCore                          0x0000000101d867b4 __89-[XCTTestRunSession executeTestsWithIdentifiers:skippingTestsWithIdentifiers:completion:]_block_invoke + 580
    35  XCTestCore                          0x0000000101dbd8e4 +[XCTContext _runInChildOfContext:forTestCase:markAsReportingBase:block:] + 180
    36  XCTestCore                          0x0000000101dbd7cc +[XCTContext runInContextForTestCase:markAsReportingBase:block:] + 104
    37  XCTestCore                          0x0000000101d86498 -[XCTTestRunSession executeTestsWithIdentifiers:skippingTestsWithIdentifiers:completion:] + 296
    38  XCTestCore                          0x0000000101dff5fc __103-[XCTExecutionWorker executeTestIdentifiers:skippingTestIdentifiers:completionHandler:completionQueue:]_block_invoke_2 + 136
    39  XCTestCore                          0x0000000101dfe360 -[XCTExecutionWorker runWithError:] + 132
    40  XCTestCore                          0x0000000101dba0a0 __25-[XCTestDriver _runTests]_block_invoke.261 + 56
    41  XCTestCore                          0x0000000101d9143c -[XCTestObservationCenter _observeTestExecutionForTestBundle:inBlock:] + 212
    42  XCTestCore                          0x0000000101db9af4 -[XCTestDriver _runTests] + 1100
    43  XCTestCore                          0x0000000101d84874 _XCTestMain + 92
    44  xctest                              0x000000010000577c main + 156
    45  dyld                                0x0000000196cf3f28 start + 2236
)
libc++abi: terminating due to uncaught exception of type NSException
```

This stood out as weird immediately because the exception that was determined to be uncaught was the expectation that the `XCTAssertThrows` macro was supposed to be catching. Looking at the code, there wasn't anything that seemed wrong considering this worked in Xcode 15 beta 2 and earlier:

```obj-c
// Class Definition
@import Foundation;

NS_ASSUME_NONNULL_BEGIN

@interface ExampleClass : NSObject

- (instancetype)init NS_UNAVAILABLE;

@end

NS_ASSUME_NONNULL_END

// Class Implementation
#import "ExampleClass.h"

@implementation ExampleClass

- (instancetype)__attribute__((noreturn)) init
{
    @throw [NSException exceptionWithName:NSInternalInconsistencyException reason:[NSString stringWithFormat:@"init is unavailable for class %@", NSStringFromClass([self class])] userInfo:nil];
}

@end

// The Test
@import XCTest;
@import ExampleFramework;

@interface ExampleClassTest : XCTestCase

@end

@implementation ExampleClassTest

- (void)testInit
{
    ExampleClass *example = [ExampleClass alloc];
#pragma clang diagnostic push
#pragma clang diagnostic ignored "-Warc-performSelector-leaks"
    XCTAssertThrows([example performSelector:NSSelectorFromString(@"init")]);
#pragma clang diagnostic pop
}

@end
```

My XCTestPlan includes configuration for running the tests with diagnostics enabled (ASan, TSan, and UBSan). I started turning them off one by one to see if that would resolve the issue and it did. Turns out, TSan is interfering somehow and causing the exception to not get caught appropriately.

---

I've logged this as FB12532865 and hopefully it is resolved soon. If not, an alternative could be to replace the macro and use a try/catch/finally block with a sentinel value that is checked and, if the exception is not thrown, call `XCTFail`.
