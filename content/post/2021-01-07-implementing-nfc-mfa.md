---
title: "Implementing NFC MFA"
subtitle: ""
date: 2021-01-07T12:00:00-06:00
tags: ["nfc", "mfa", "2fa"]
---

Modern authentication workflows allows for users to leverage [multi-factor authentication](https://en.wikipedia.org/wiki/Multi-factor_authentication) (MFA) and sometimes [passwordless authentication](https://en.wikipedia.org/wiki/Passwordless_authentication) for a more secure experience. If using a browser (e.g. Safari via [ASWebAuthenticationSession](https://developer.apple.com/documentation/authenticationservices/aswebauthenticationsession?language=objc)), most of this is handled for your application via [WebAuthn](https://webauthn.io/), but if you have a native application you must implement part of the workflow yourself via USB or [NFC](https://en.wikipedia.org/wiki/Near-field_communication) via the [ISO 7816 standard](https://www.iso.org/obp/ui/#iso:std:iso-iec:7816:-4:ed-4:v1:en).

To use NFC on iOS devices (iOS 13+), Apple provides the [CoreNFC framework](https://developer.apple.com/documentation/corenfc?language=objc). Prior to using the framework though, the application must configure itself appropriately:

1. Add the Near Field Communication Tag Reading (`com.apple.developer.nfc.readersession.formats`) entitlement.
2. Add the Privacy - NFC Scan Usage Description (`NFCReaderUsageDescription`) string to the application's plist.
3. Add the ISO 7816 application identifiers for NFC Tag Reader Session (`com.apple.developer.nfc.readersession.iso7816.select-identifiers`) to the application's plist.

The ISO 7816 AIDs will depend on what kind of NFC tags you will be supporting, but a small list of common ones are:

* `A0000006472F0001` - For [FIDO](https://fidoalliance.org/) / [U2F](https://en.wikipedia.org/wiki/Universal_2nd_Factor)
* `A0000005272101` - For [OATH](https://openauthentication.org/)
* `A000000308` - For Personal Identity Verification ([PIV](https://csrc.nist.gov/Projects/PIV/)) Common Access Cards ([CAC](https://en.wikipedia.org/wiki/Common_Access_Card))

In the event that you need additional AIDs (e.g. [OTP](https://en.wikipedia.org/wiki/One-time_password) tags), you can look at the [registry](https://www.eftlab.com/knowledge-base/211-emv-aid-rid-pix/).

With the configuration items complete the entitlements should look like

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>com.apple.developer.nfc.readersession.formats</key>
  <array>
    <string>NDEF</string>
    <string>TAG</string>
  </array>
</dict>
</plist>

```

and the plist (abridged)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>com.apple.developer.nfc.readersession.iso7816.select-identifiers</key>
  <array>
    <string>A0000006472F0001</string>
    <string>A0000005272101</string>
    <string>A000000308</string>
  </array>
  <key>NFCReaderUsageDescription</key>
  <string>Access is required for strong authentication</string>
</dict>
</plist>

```

Now for the implementation, an object needs to maintain an instance of [`NFCTagReaderSession`](https://developer.apple.com/documentation/corenfc/nfctagreadersession?language=objc) and act as the [`NFCTagReaderSessionDelegate`](https://developer.apple.com/documentation/corenfc/nfctagreadersessiondelegate?language=objc). Once you have that, the process is straightforward but relies on application logic to determine when to start and stop:

1. Create the NFC session.
2. Start polling using [ISO 14443](https://www.iso.org/obp/ui/#iso:std:iso-iec:14443:-2:ed-4:v1:en).
3. Search for ISO 7816 compatible tags.
4. If one is found, connect to it. Otherwise, restart polling (or abort).
5. Assuming the connection to the tag is made successfully, execute your commands. These commands depend on the operation (registration, sign, etc).
6. Once your desired operations are complete, end the session and proceed to whatever the next step in your application is.

The object maintaining the session should also have a serial queue for command execution to prevent errors from occurring when the application attempts multiple commands. In the example below, that queue implements basic timeout logic, but if you need something more advanced, it is recommended to create a subclass of [NSOperation](https://developer.apple.com/documentation/foundation/nsoperation?language=objc).

### Example

In this example, a class called `ExampleNFCAuthenticator` is used for maintaining the NFC session and executing commands. The application should create a singular instance of the authenticator when authentication is required and then deallocate it once authentication complete. Since the commands are specific to the given authentication method and operation, they are omitted for simplicity's sake. In a real implementation, the authenticator could have convenience methods to generate the commands on behalf of the caller.

#### Header

```obj-c
@import Foundation;
@import CoreNFC;

NS_ASSUME_NONNULL_BEGIN

/**
 * Block definition for code to be executed when a command finishes.
 * @param responseData The result data from the command's execution.
 * @param sw1          Command processing status byte 1.
 * @param sw2          Command processing status byte 2.
 * @param error        The error associated with the command's execution.
 */
typedef void (^CommandCompletionHandler)(NSData *responseData, uint8_t sw1, uint8_t sw2, NSError *_Nullable error) NS_SWIFT_NAME(ExampleNFCAuthenticator.CommandCompletionHandler);

/**
 * Block definition for code to be executed when a command does not complete in time.
 */
typedef void (^CommandTimeoutHandler)(void) NS_SWIFT_NAME(ExampleNFCAuthenticator.CommandTimeoutHandler);

/**
 * Maintains an NFC scanning session to connect to a NFC tag for authentication via commands.
 */
@interface ExampleNFCAuthenticator : NSObject

/**
 * Starts polling for an NFC tag.
 * @param message The message to show to the user when scanning begins.
 * @note Will invalidate the existing session to start a new one if called without stopping the current session.
 */
- (void)startSessionWithMessage:(NSString *)message;

/**
 * Restarts the polling for an NFC tag.
 * @note Cannot restart a stopped session.
 */
- (void)restartSession;

/**
 * Stops polling for an NFC tag.
 * @note Cannot reuse a stopped session.
 */
- (void)stopSession;

/**
 * Stops polling for an NFC tag and presents a message to the user.
 * @note Cannot reuse a stopped session.
 */
- (void)stopSessionWithMessage:(NSString *)message;

/**
 * Sends a command to the currently active NFC tag.
 * @param command           The command (Application Data Unit) to execute.
 * @param timeout           The amount of time to wait before canceling the execution. Set this to a value < 0 to not have a timeout.
 * @param timeoutHandler    Optional block to execute when the command execution time expires. Will be called on the main thread.
 * @param completionHandler Block to execute when the command execution completes. The timeout handler may be called if the timeout occurs during execution. Since the block will execute on some queue, dispatching to a specific queue for processing is your responsiblity if desired.
 */
- (void)executeCommand:(NFCISO7816APDU *)command withTimeout:(NSTimeInterval)timeout timeoutHandler:(nullable CommandTimeoutHandler)timeoutHandler completionHandler:(CommandCompletionHandler)completionHandler;

/**
 * Cancels all queued commands. If a command had a timeout handler, it may be called if the timeout occurs right before the cancelation.
 * @note Currently executing commands may still complete.
 */
- (void)cancelAllCommandExecutions;

@end

NS_ASSUME_NONNULL_END
```

#### Implementation

```obj-c
#import "ExampleNFCAuthenticator.h"
@import os.log;

NS_ASSUME_NONNULL_BEGIN

@interface ExampleNFCAuthenticator () <NFCTagReaderSessionDelegate>

/**
 * The currently active NFC tag reader session
 */
@property (atomic, strong, nullable) NFCTagReaderSession *activeSession;

/**
 * The queue to execute tag commands on
 */
@property (nonatomic, strong) NSOperationQueue *operationQueue;

@end

@implementation ExampleNFCAuthenticator

#pragma mark - Object Life Cycle

- (instancetype)init
{
  if ((self = [super init])) {
    // Setting the queue type to serial and the max operation count to 1 ensure that we don't process multiple commands at the same time
    _operationQueue = [[NSOperationQueue alloc] init];
    dispatch_queue_attr_t attributes = dispatch_queue_attr_make_with_qos_class(DISPATCH_QUEUE_SERIAL, QOS_CLASS_USER_INITIATED, -1);
    dispatch_queue_t opQueue = dispatch_queue_create("com.example.nfc.reader.operations", attributes);
    _operationQueue.underlyingQueue = opQueue;
    _operationQueue.maxConcurrentOperationCount = 1;
  }
    
  return self;
}

#pragma mark - Public API

- (void)startSessionWithMessage:(NSString *)message
{
  if (!NFCTagReaderSession.readingAvailable) {
    os_log_debug(OS_LOG_DEFAULT, "Cannot perform NFC scanning on an incompatible device");
    return;
  }
    
  if (self.activeSession) {
    [self stopSession];
  }

  // If you want the delegate to be called on a specific queue, replace nil with that queue  
  NFCTagReaderSession *tagSession = [[NFCTagReaderSession alloc] initWithPollingOption:NFCPollingISO14443 delegate:self queue:nil];
  tagSession.alertMessage = message
  self.activeSession = tagSession;
  [tagSession beginSession];
}

- (void)restartSession
{
  os_log_debug(OS_LOG_DEFAULT, "Restarting NFC polling");
  [self.activeSession restartPolling];
}

- (void)stopSession
{
  os_log_debug(OS_LOG_DEFAULT, "Stopping NFC polling");
  [self.activeSession invalidateSession];
}

- (void)stopSessionWithMessage:(NSString *)message
{
  os_log_debug(OS_LOG_DEFAULT, "Stopping NFC polling with message: %{public}@", message);
  [self.activeSession invalidateSessionWithErrorMessage:message];
}

- (void)executeCommand:(NFCISO7816APDU *)command withTimeout:(NSTimeInterval)timeout timeoutHandler:(nullable CommandTimeoutHandler)timeoutHandler completionHandler:(CommandCompletionHandler)completionHandler
{
  NSAssert(self.activeSession.connectedTag.isAvailable, @"Cannot execute command on a non-existent or inactive tag");
    
  __weak __auto_type weakSelf = self;
  NSBlockOperation *op = [NSBlockOperation blockOperationWithBlock:^{
    [(id<NFCISO7816Tag>)weakSelf.activeSession.connectedTag sendCommandAPDU:command completionHandler:completionHandler];
  }];
    
  [self.operationQueue addOperation:op];
    
  if (timeout > 0) {
    __weak __auto_type weakOp = op;
    dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(timeout * NSEC_PER_SEC)), dispatch_get_main_queue(), ^{
      if (weakOp && !weakOp.isFinished) {
        [weakOp cancel];
                
        if (timeoutHandler) {
          timeoutHandler();
        }
      }
    });
  }
}

- (void)cancelAllCommandExecutions
{
  os_log_debug(OS_LOG_DEFAULT, "Canceling all enqueued command operations");
  [self.operationQueue cancelAllOperations];
}

#pragma mark - NFCTagReaderSessionDelegate

- (void)tagReaderSessionDidBecomeActive:(NFCTagReaderSession *)session
{
  os_log_debug(OS_LOG_DEFAULT, "NFC scanning activated");
}

- (void)tagReaderSession:(NFCTagReaderSession *)session didDetectTags:(NSArray<__kindof id<NFCTag>> *)tags
{
  os_log_debug(OS_LOG_DEFAULT, "NFC scanning detected %lu tags", tags.count);
    
  if (!tags.count) {
    os_log_debug(OS_LOG_DEFAULT, "Restarting polling as no tags were detected");
    [self restartSession]; // Attempt to get a new tag
    return;
  }
    
  static NSPredicate *filter;
  static dispatch_once_t onceToken;
  dispatch_once(&onceToken, ^{
    filter = [NSPredicate predicateWithBlock:^BOOL(id<NFCTag> _Nullable tag, NSDictionary<NSString *,id> *_Nullable bindings) {
      return tag.type == NFCTagTypeISO7816Compatible;
    }];
  });
    
  NSArray<id<NFCISO7816Tag>> *validTags = [tags filteredArrayUsingPredicate:filter];
  id<NFCISO7816Tag> validTag = validTags.firstObject;
    
  if (!validTag) {
    os_log_debug(OS_LOG_DEFAULT, "Restarting polling as no ISO 7816 tags were detected");
    [self restartSession]; // Attempt to get a new tag
    return;
  }
    
  __weak __auto_type weakSelf = self;
  [self.activeSession connectToTag:validTag completionHandler:^(NSError *_Nullable error) {
    if (error) {
      os_log_debug(OS_LOG_DEFAULT, "Unable to conenct to detected ISO 7816 tag: %{public}@", error.debugDescription);
      [weakSelf restartSession]; // Attempt to get a new tag
      return;
    }
        
    os_log_debug(OS_LOG_DEFAULT, "Successfully connected to ISO 7816 tag");
  }];
}

- (void)tagReaderSession:(NFCTagReaderSession *)session didInvalidateWithError:(NSError *)error
{
  os_log_debug(OS_LOG_DEFAULT, "NFC scanning stopped: %{public}@", error.debugDescription);
  self.activeSession = nil;
}

@end

NS_ASSUME_NONNULL_END
```

---

As noted earlier, you can also use USB as a fallback in the event the device does not have NFC capabilities or some weird adapter is required. However, these devices must be [MFi](https://mfi.apple.com/en/home.html) certified.
