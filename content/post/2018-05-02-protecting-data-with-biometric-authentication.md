---
title: "Protecting Data with Biometric Authentication"
subtitle: "Fingers, Faces, and Data"
date: 2018-05-02T16:00:00-05:00
tags: ["keychain", "biometrics", "biometry"]
---

On the Apple platforms, the [keychain](https://en.wikipedia.org/wiki/Keychain_(software)) is the database provided by the system to store small bits of data securely. Although the data can be anything, the keychain is geared towards certain types of data: passwords, certificates, keys, and identities. With the advent of [Touch ID](https://en.wikipedia.org/wiki/Touch_ID) and [Face ID](https://en.wikipedia.org/wiki/Face_ID), an additional layer of security was added since the keychain items themselves can be stored in the [secure enclave](https://www.theiphonewiki.com/wiki/Secure_Enclave) ([ECC](https://en.wikipedia.org/wiki/Elliptic-curve_cryptography) keys only) but also using biometrics instead of passwords to authenticate the use of items reduces the risk of accidental exposure. Therefore, new APIs were added to take advantage of these capabilities.

## Keychain Basics
The keychain [API](https://developer.apple.com/library/content/documentation/Security/Conceptual/keychainServConcepts/01introduction/introduction.html) is one of the most powerful and complex APIs that Apple provides in the [Security](https://developer.apple.com/documentation/security?language=objc) framework. This also means it is one of the most obtuse and painful APIs to interact with since 1) you are working with a C API with [Core Foundation](https://developer.apple.com/documentation/corefoundation?language=objc) (i.e. non-[ARC](https://developer.apple.com/library/content/releasenotes/ObjectiveC/RN-TransitioningToARC/Introduction/Introduction.html)) objects, 2) doesn't use [blocks](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/ProgrammingWithObjectiveC/WorkingwithBlocks/WorkingwithBlocks.html), & 3) uses [opaque](https://en.wikipedia.org/wiki/Opaque_pointer) return values and double pointers. I could spend a week going over the intricacies of the keychain, but that would not be an interesting read. So, I'll try to cover some of the basics in the examples below, but understand that there are way more configuration options and rules that need to be considered when leveraging the API.

Other than [key generation](https://developer.apple.com/documentation/security/certificate_key_and_trust_services/keys/generating_new_cryptographic_keys?language=objc), there are four common operations: [creation](https://developer.apple.com/documentation/security/1401659-secitemadd?language=objc), [update](https://developer.apple.com/documentation/security/1393617-secitemupdate?language=objc), [fetch](https://developer.apple.com/documentation/security/1398306-secitemcopymatching?language=objc), and [deletion](https://developer.apple.com/documentation/security/1395547-secitemdelete?language=objc). To execute these operations, you create a dictionary that represents the query for the keychain database. Now, certain attributes are only valid for some operations and not others, so reading the documentation is key. Additionally, some attributes are mutually exclusive with others (e.g. [SecAccessControlRef](https://developer.apple.com/documentation/security/secaccesscontrolref?language=objc) and [kSecAttrAccessible](https://developer.apple.com/documentation/security/ksecattraccessible?language=objc)). With all of that said, optimizing the query makes a huge difference since there can be a large number of items in the keychain and vague queries can lead to multiple results or errors.

### Using the Keychain
To begin using the biometric capabilities of the device (albeit in a round-about way), you need to create an [ACL](https://en.wikipedia.org/wiki/Access_control_list). The ACL defines the protection level of the item in the keychain. The options used to create the ACL can limit the export of the item (i.e. constrain the item to the current device only) but also include provisions for requiring the device have a passcode set or require the fingerprint set to not change or else delete the item. After creating the ACL, you also need to add the additional attributes to the query that defines the item such as its class, label, and access group. Once you have the query created, you pass that into the operation and then inspect the result.

Note: In the examples below, the keychain APIs aren't dispatched to a separate queue for simplicity's sake (keychain calls can block). Also, there should be a preflight check to ensure that biometrics are available, but that will be covered in the next section.

Add Example:
```obj-c
- (void)addItemToKeychain
{
    NSData *passwordData = [@"secret password" dataUsingEncoding:NSUTF8StringEncoding];
    
    if (!passwordData.length) {
        return;
    }
    
    CFErrorRef error = nil;
    SecAccessControlRef acl = SecAccessControlCreateWithFlags(kCFAllocatorDefault,
                                                              kSecAttrAccessibleWhenPasscodeSetThisDeviceOnly,
                                                              kSecAccessControlBiometryAny,
                                                              &error);
    NSError *errorObj = (__bridge_transfer NSError *)error;
    
    if (acl == NULL || errorObj != nil) {
        // Handle Error
        return;
    }
    
    NSDictionary *query = @{
                             (__bridge NSString *)kSecClass : (__bridge NSString *)kSecClassGenericPassword,
                             (__bridge NSString *)kSecAttrService : @"Example",
                             (__bridge NSString *)kSecAttrLabel : @"com.example.mypassword",
                             (__bridge NSString *)kSecValueData : passwordData,
                             (__bridge NSString *)kSecUseAuthenticationUI : (__bridge NSString *)kSecUseAuthenticationUIAllow,
#if !TARGET_IPHONE_SIMULATOR
                             (__bridge NSString *)kSecAttrAccessGroup : @"group.com.example.mygroup",
#endif
                             (__bridge NSString *)kSecAttrAccessControl : (__bridge_transfer id)acl
                           };
    
    OSStatus result = SecItemAdd((__bridge CFDictionaryRef)query, nil);
    if (result != errSecSuccess) {
        // Handle Failure
        if (@available(iOS 11.3, *)) {
            NSString *errorMessage = (__bridge_transfer NSString *)SecCopyErrorMessageString(result, NULL);
            os_log_debug(OS_LOG_DEFAULT, "Keychain add error: %{public}s", errorMessage.UTF8String);
        }
    }
}
```

Update Example:
```obj-c
- (void)updateKeychainItem
{
    NSData *passwordData = [@"new secret password" dataUsingEncoding:NSUTF8StringEncoding];
    
    if (!passwordData.length) {
        return;
    }
    
    CFErrorRef error = nil;
    SecAccessControlRef acl = SecAccessControlCreateWithFlags(kCFAllocatorDefault
                                                              kSecAttrAccessibleWhenPasscodeSetThisDeviceOnly,
                                                              kSecAccessControlBiometryAny,
                                                              &error);
    NSError *errorObj = (__bridge_transfer NSError *)error;
    
    if (acl == NULL || errorObj != nil) {
        // Handle Error
        return;
    }
    
    NSDictionary *query = @{
                             (__bridge NSString *)kSecClass : (__bridge NSString *)kSecClassGenericPassword,
                             (__bridge NSString *)kSecAttrService : @"Example",
                             (__bridge NSString *)kSecAttrLabel : @"com.example.mypassword",
                             (__bridge NSString *)kSecUseAuthenticationUI : (__bridge NSString *)kSecUseAuthenticationUIAllow,
#if !TARGET_IPHONE_SIMULATOR
                             (__bridge NSString *)kSecAttrAccessGroup : @"group.com.example.mygroup",
#endif
                             (__bridge NSString *)kSecAttrAccessControl : (__bridge_transfer id)acl
                           };
    
    NSDictionary *changeAttributes = @{
                                        (__bridge NSString *)kSecValueData : passwordData
                                      };
    
    OSStatus result = SecItemUpdate((__bridge CFDictionaryRef)query, (__bridge CFDictionaryRef)changeAttributes);
    if (result != errSecSuccess) {
        // Handle Failure
        if (@available(iOS 11.3, *)) {
            NSString *errorMessage = (__bridge_transfer NSString *)SecCopyErrorMessageString(result, NULL);
            os_log_debug(OS_LOG_DEFAULT, "Keychain update error: %{public}s", errorMessage.UTF8String);
        }
    }
}
```

Fetch Example:
```obj-c
- (void)fetchKeychainItem
{
    NSDictionary *query = @{
                             (__bridge NSString *)kSecClass : (__bridge NSString *)kSecClassGenericPassword,
                             (__bridge NSString *)kSecAttrService : @"Example",
                             (__bridge NSString *)kSecAttrLabel : @"com.example.mypassword",
                             (__bridge NSString *)kSecUseAuthenticationUI : (__bridge NSString *)kSecUseAuthenticationUIAllow,
#if !TARGET_IPHONE_SIMULATOR
                             (__bridge NSString *)kSecAttrAccessGroup : @"group.com.example.mygroup",
#endif
                             (__bridge NSString *)kSecUseOperationPrompt : @"Please authenticate to retrieve the password",
                             (__bridge NSString *)kSecReturnData : (__bridge NSNumber *)kCFBooleanTrue,
                             (__bridge NSString *)kSecMatchLimit : (__bridge NSString *)kSecMatchLimitOne
                           };
    
    CFTypeRef resultItem = nil;
    OSStatus result = SecItemCopyMatching((__bridge CFDictionaryRef)query, &resultItem);
    if (result != errSecSuccess) {
        // Handle Failure
        if (@available(iOS 11.3, *)) {
            NSString *errorMessage = (__bridge_transfer NSString *)SecCopyErrorMessageString(result, NULL);
            os_log_debug(OS_LOG_DEFAULT, "Keychain fetch error: %{public}s", errorMessage.UTF8String);
        }
    }
    else {
        NSString *password = [[NSString alloc] initWithData:(__bridge_transfer NSData *)resultItem encoding:NSUTF8StringEncoding];
        // Do something with password
    }
}
```

Delete Example:
```obj-c
- (void)deleteKeychainItem
{
    NSDictionary *query = @{
                             (__bridge NSString *)kSecClass : (__bridge NSString *)kSecClassGenericPassword,
                             (__bridge NSString *)kSecAttrService : @"Example",
#if !TARGET_IPHONE_SIMULATOR
                             (__bridge NSString *)kSecAttrAccessGroup : @"group.com.example.mygroup",
#endif
                             (__bridge NSString *)kSecAttrLabel : @"com.example.mypassword"
                           };
    
    OSStatus result = SecItemDelete((__bridge CFDictionaryRef)query);
    if (result != errSecSuccess) {
        // Handle Failure
        if (@available(iOS 11.3, *)) {
            NSString *errorMessage = (__bridge_transfer NSString *)SecCopyErrorMessageString(result, NULL);
            os_log_debug(OS_LOG_DEFAULT, "Keychain delete error: %{public}s", errorMessage.UTF8String);
        }
    }
}
```

## Local Authentication
In combination with the Security framework, Apple also provides the [Local Authentication](https://developer.apple.com/documentation/localauthentication?language=objc) framework as a means of interacting with the biometric capabilities of the device. The main reason to use this API over the normal keychain API is to use application provided passwords (optionally user provided) to generate the data encryption key.

### Using Local Authentication
To use the biometric capabilities of the device, you must create a [LAContext](https://developer.apple.com/documentation/localauthentication/lacontext?language=objc) object and verify biometrics are available. If biometrics are available, you then set the credential you want to use on it and then execute the keychain operation.

Preflight Example:
```obj-c
- (BOOL)canUseBiometrics
{
    NSError *error = nil;
    LAContext *context = [[LAContext alloc] init];
    BOOL available = [context canEvaluatePolicy:LAPolicyDeviceOwnerAuthenticationWithBiometrics error:&error];
    
    if (error) {
        // Handle Error
    }
    
    return available && context.biometryType != LABiometryTypeNone;
}
```

Add Example:
```obj-c
- (void)addKeychainItemWithPassword
{
    NSData *appPasswordData = [@"1234567890" dataUsingEncoding:NSUTF8StringEncoding];
    NSData *passwordData = [@"secret password" dataUsingEncoding:NSUTF8StringEncoding];
    
    if (!passwordData.length) {
        return;
    }
    
    CFErrorRef error = nil;
    SecAccessControlCreateFlags flags = kSecAccessControlBiometryAny | kSecAccessControlApplicationPassword | kSecAccessControlAnd;
    SecAccessControlRef acl = SecAccessControlCreateWithFlags(kCFAllocatorDefault,
                                                              kSecAttrAccessibleWhenPasscodeSetThisDeviceOnly,
                                                              flags,
                                                              &error);
    NSError *errorObj = (__bridge_transfer NSError *)error;
    
    if (acl == NULL || errorObj != nil) {
        // Handle Error
        return;
    }
    
    LAContext *context = [[LAContext alloc] init];
    [context setCredential:appPasswordData type:LACredentialTypeApplicationPassword];
    
    NSDictionary *query = @{
                             (__bridge NSString *)kSecClass : (__bridge NSString *)kSecClassGenericPassword,
                             (__bridge NSString *)kSecAttrService : @"Example",
                             (__bridge NSString *)kSecAttrLabel : @"com.example.mypassword",
                             (__bridge NSString *)kSecValueData : passwordData,
                             (__bridge NSString *)kSecUseAuthenticationUI : (__bridge NSString *)kSecUseAuthenticationUIAllow,
#if !TARGET_IPHONE_SIMULATOR
                             (__bridge NSString *)kSecAttrAccessGroup : @"group.com.example.mygroup",
#endif
                             (__bridge NSString *)kSecAttrAccessControl : (__bridge_transfer id)acl,
                             (__bridge NSString *)kSecUseAuthenticationContext : context
                           };
    
    [context evaluateAccessControl:acl
                         operation:LAAccessControlOperationCreateItem
                   localizedReason:@"Please authenticate to save the password"
                             reply:^(BOOL success, NSError *_Nullable error) {
        if (success) {
            OSStatus result = SecItemAdd((__bridge CFDictionaryRef)query, nil);
            if (result != errSecSuccess) {
                // Handle Failure
                if (@available(iOS 11.3, *)) {
                    NSString *errorMessage = (__bridge_transfer NSString *)SecCopyErrorMessageString(result, NULL);
                    os_log_debug(OS_LOG_DEFAULT, "Keychain add error: %{public}s", errorMessage.UTF8String);
                }
            }
        }
        else {
            // Handle Error
        }
    }];
}
```

Update Example:
```obj-c
- (void)updateKeychainItemWithPassword
{
    NSData *appPasswordData = [@"1234567890" dataUsingEncoding:NSUTF8StringEncoding];
    NSData *passwordData = [@"new secret password" dataUsingEncoding:NSUTF8StringEncoding];
    
    if (!passwordData.length) {
        return;
    }
    
    CFErrorRef error = nil;
    SecAccessControlCreateFlags flags = kSecAccessControlBiometryAny | kSecAccessControlApplicationPassword | kSecAccessControlAnd;
    SecAccessControlRef acl = SecAccessControlCreateWithFlags(kCFAllocatorDefault,
                                                              kSecAttrAccessibleWhenPasscodeSetThisDeviceOnly,
                                                              flags,
                                                              &error);
    NSError *errorObj = (__bridge_transfer NSError *)error;
    
    if (acl == NULL || errorObj != nil) {
        // Handle Error
        return;
    }
    
    LAContext *context = [[LAContext alloc] init];
    [context setCredential:appPasswordData type:LACredentialTypeApplicationPassword];
    
    NSDictionary *query = @{
                             (__bridge NSString *)kSecClass : (__bridge NSString *)kSecClassGenericPassword,
                             (__bridge NSString *)kSecAttrService : @"Example",
                             (__bridge NSString *)kSecAttrLabel : @"com.example.mypassword",
                             (__bridge NSString *)kSecUseAuthenticationUI : (__bridge NSString *)kSecUseAuthenticationUIAllow,
#if !TARGET_IPHONE_SIMULATOR
                             (__bridge NSString *)kSecAttrAccessGroup : @"group.com.example.mygroup",
#endif
                             (__bridge NSString *)kSecAttrAccessControl : (__bridge_transfer id)acl,
                             (__bridge NSString *)kSecUseAuthenticationContext : context
                          };
    
    NSDictionary *changeAttributes = @{
                                        (__bridge NSString *)kSecValueData : passwordData
                                      };
    
    [context evaluateAccessControl:acl
                         operation:LAAccessControlOperationUseItem
                   localizedReason:@"Please authenticate to update the password"
                             reply:^(BOOL success, NSError *_Nullable error) {
        if (success) {
            OSStatus result = SecItemUpdate((__bridge CFDictionaryRef)query, (__bridge CFDictionaryRef)changeAttributes);
            if (result != errSecSuccess) {
                // Handle Failure
                if (@available(iOS 11.3, *)) {
                    NSString *errorMessage = (__bridge_transfer NSString *)SecCopyErrorMessageString(result, NULL);
                    os_log_debug(OS_LOG_DEFAULT, "Keychain update error: %{public}s", errorMessage.UTF8String);
                }
            }
        }
        else {
            // Handle Error
        }
    }];
}
```

Fetch Example:
```obj-c
- (void)fetchKeychainItemWithPassword
{
    NSData *appPasswordData = [@"1234567890" dataUsingEncoding:NSUTF8StringEncoding];
    CFErrorRef error = nil;
    SecAccessControlCreateFlags flags = kSecAccessControlBiometryAny | kSecAccessControlApplicationPassword | kSecAccessControlAnd;
    SecAccessControlRef acl = SecAccessControlCreateWithFlags(kCFAllocatorDefault,
                                                              kSecAttrAccessibleWhenPasscodeSetThisDeviceOnly,
                                                              flags,
                                                              &error);
    NSError *errorObj = (__bridge_transfer NSError *)error;
    
    if (acl == NULL || errorObj != nil) {
        // Handle Error
        return;
    }
    
    LAContext *context = [[LAContext alloc] init];
    [context setCredential:appPasswordData type:LACredentialTypeApplicationPassword];
    
    NSDictionary *query = @{
                             (__bridge NSString *)kSecClass : (__bridge NSString *)kSecClassGenericPassword,
                             (__bridge NSString *)kSecAttrService : @"Example",
                             (__bridge NSString *)kSecAttrLabel : @"com.example.mypassword",
                             (__bridge NSString *)kSecUseAuthenticationUI : (__bridge NSString *)kSecUseAuthenticationUIAllow,
#if !TARGET_IPHONE_SIMULATOR
                             (__bridge NSString *)kSecAttrAccessGroup : @"group.com.example.mygroup",
#endif
                             (__bridge NSString *)kSecUseOperationPrompt : @"Please authenticate to retrieve the password",
                             (__bridge NSString *)kSecReturnData : (__bridge NSNumber *)kCFBooleanTrue,
                             (__bridge NSString *)kSecMatchLimit : (__bridge NSString *)kSecMatchLimitOne,
                             (__bridge NSString *)kSecUseAuthenticationContext : context
                          };
    
    [context evaluateAccessControl:acl
                         operation:LAAccessControlOperationUseItem
                   localizedReason:@"Please authenticate to use the password"
                             reply:^(BOOL success, NSError *_Nullable error) {
        if (success) {
            CFTypeRef resultItem = nil;
            OSStatus result = SecItemCopyMatching((__bridge CFDictionaryRef)query, &resultItem);
            if (result != errSecSuccess) {
                // Handle Failure
                if (@available(iOS 11.3, *)) {
                    NSString *errorMessage = (__bridge_transfer NSString *)SecCopyErrorMessageString(result, NULL);
                    os_log_debug(OS_LOG_DEFAULT, "Keychain fetch error: %{public}s", errorMessage.UTF8String);
                }
            }
            else {
                NSString *password = [[NSString alloc] initWithData:(__bridge_transfer NSData *)resultItem encoding:NSUTF8StringEncoding];
                // Do something with password
            }
        }
        else {
            // Handle Error
        }
    }];
}
```

Delete Example:
```obj-c
- (void)testKeychainQueryDeleteWithPassword
{
    NSData *appPasswordData = [@"1234567890" dataUsingEncoding:NSUTF8StringEncoding];
    CFErrorRef error = nil;
    SecAccessControlCreateFlags flags = kSecAccessControlBiometryAny | kSecAccessControlApplicationPassword | kSecAccessControlAnd;
    SecAccessControlRef acl = SecAccessControlCreateWithFlags(kCFAllocatorDefault,
                                                              kSecAttrAccessibleWhenPasscodeSetThisDeviceOnly,
                                                              flags,
                                                              &error);
    NSError *errorObj = (__bridge_transfer NSError *)error;
    
    if (acl == NULL || errorObj != nil) {
        // Handle Error
        return;
    }
    
    LAContext *context = [[LAContext alloc] init];
    [context setCredential:appPasswordData type:LACredentialTypeApplicationPassword];
    
    NSDictionary *query = @{
                             (__bridge NSString *)kSecClass : (__bridge NSString *)kSecClassGenericPassword,
                             (__bridge NSString *)kSecAttrService : @"Example",
#if !TARGET_IPHONE_SIMULATOR
                             (__bridge NSString *)kSecAttrAccessGroup : @"group.com.example.mygroup",
#endif
                             (__bridge NSString *)kSecAttrLabel : @"com.example.mypassword",
                             (__bridge NSString *)kSecUseAuthenticationContext : context
                          };
    
    [context evaluateAccessControl:acl
                         operation:LAAccessControlOperationUseItem
                   localizedReason:@"Please authenticate to delete the password"
                             reply:^(BOOL success, NSError *_Nullable error) {
        if (success) {
            OSStatus result = SecItemDelete((__bridge CFDictionaryRef)query);
            if (result != errSecSuccess) {
                // Handle Failure
                if (@available(iOS 11.3, *)) {
                    NSString *errorMessage = (__bridge_transfer NSString *)SecCopyErrorMessageString(result, NULL);
                    os_log_debug(OS_LOG_DEFAULT, "Keychain delete error: %{public}s", errorMessage.UTF8String);
                }
            }
        }
        else {
            // Handle Error
        }
    }];
}
```

## Conclusion
The keychain API is messy and complicated, but it is much better than trying to create your own "secure" database to store secrets in. Also, using biometric authentication provides an additional layer of security around passwords, certificates, and keys since biometrics are hard to spoof and the secure enclave has proven to be "unhackable" thus far. Going forward, it is looking like every Apple device will have some sort of biometric authentication mechanism, so leveraging it now should be future proof.
