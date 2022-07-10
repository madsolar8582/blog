---
title: "Simplifying Authorization with LARight"
subtitle: ""
date: 2022-07-10T15:00:00-05:00
tags: ["LocalAuthentication", "LARight"]
---

Prior to iOS 16, storing credentials and then authenticating the user to authorize the use of said credential was cumbersome. First, you needed to create keychain queries for your CRUD operations which included ACLs. Second, you needed to create the credential (e.g. key pair) yourself and that was error prone (you could create the wrong kind and not be able to store it in the secure enclave). Lastly, you needed to deal with CoreFoundation objects instead of pure Objective-C/Swift objects. Luckily, the new method of doing this (assuming you don't need full control) is far simpler.

Let's start with the new class [`LARight`](https://developer.apple.com/documentation/localauthentication/laright?changes=latest_minor&language=objc). this class allows you to protect some aspect of your application with built-in authentication and state management (including notifications and KVO). You configure this right (i.e. authorization grant) with an authentication requirement: biometry, current set biometry (data change invalidates the right), or biometry with device passcode as a fallback. Once you create your right object, you hold onto it for the app's lifecycle as the object needs to be retained (similar to a `LAContext`) and call either [`authorizeWithLocalizedReason:completion:`](https://developer.apple.com/documentation/localauthentication/laright/3951095-authorizewithlocalizedreason?changes=latest_minor&language=objc) or [`deauthorizeWithCompletion:`](https://developer.apple.com/documentation/localauthentication/laright/3951097-deauthorizewithcompletion?changes=latest_minor&language=objc) as needed. 

If creating a framework, you might want to create a small helper method to make creating rights easier:

```obj-c
@import LocalAuthentication;

typedef NS_ENUM(NSUInteger, EXMPLCredentialRequirementType) {
    /// Credentials are accessed via biometry.
    EXMPLCredentialRequirementTypeBiometry = 0,
    /// Credentials are accessed via biometry and will fail if there was an enrollment data change.
    EXMPLCredentialRequirementTypeBiometryCurrentSet,
    /// Credentials are accessed via biometry or the device's passcode.
    EXMPLCredentialRequirementTypeBiometryWithPasscodeFallback
} NS_SWIFT_NAME(EXMPLCredentialManager.CredentialRequirementType);

@interface EXMPLCredentialManager : NSObject

+ (LARight *)createCredentialRightWithType:(EXMPLCredentialRequirementType)credentialRequirementType;

@end

@implementation EXMPLCredentialManager

+ (LARight *)createCredentialRightWithType:(EXMPLCredentialRequirementType)credentialRequirementType
{
    LAAuthenticationRequirement *requirement = nil;
    switch (credentialRequirementType) {
        case EXMPLCredentialRequirementTypeBiometry:
            requirement = LAAuthenticationRequirement.biometryRequirement;
            break;
        case EXMPLCredentialRequirementTypeBiometryCurrentSet:
            requirement = LAAuthenticationRequirement.biometryCurrentSetRequirement;
            break;
        case EXMPLCredentialRequirementTypeBiometryWithPasscodeFallback:
            requirement = [LAAuthenticationRequirement biometryRequirementWithFallback:LABiometryFallbackRequirement.devicePasscodeRequirement];
            break;
    }
    
    return [[LARight alloc] initWithRequirement:requirement];
}

@end
```

But what if you need an authorization to last for the session's lifecycle rather than the apps? That is where [`LAPersistedRight`](https://developer.apple.com/documentation/localauthentication/lapersistedright?changes=latest_minor&language=objc) comes into play. Instead of serializing a `LARight`, `LAPersistedRight`s are managed for you by the [`LARightStore`](https://developer.apple.com/documentation/localauthentication/larightstore?changes=latest_minor&language=objc) which, as the name suggests, stores the rights in the device's secure enclave. One thing to note is that since the secure enclave is used, the APIs are asynchronous (thus requiring blocks). However, the creation process still requires a LARight:

```obj-c
typedef void (^EXMPLCredentialManagerPersistentCredentialRightCreationCompletionHandler)(LAPersistedRight *_Nullable persistedRight, NSError *_Nullable error) NS_SWIFT_NAME(EXMPLCredentialManager.PersistentCredentialRightAccessCompletionHandler);

+ (void)createPersistedCredentialRightWithType:(EXMPLCredentialRequirementType)credentialRequirementType identifier:(NSString *)identifier completionHandler:(EXMPLCredentialManagerPersistentCredentialRightCreationCompletionHandler)completionHandler
{
    LARight *right = [self createCredentialRightWithType:credentialRequirementType];
    [LARightStore.sharedStore saveRight:right identifier:identifier completion:completionHandler];
}
```

Once stored, you refer to the saved right via its identifier (like the keychain-based API):
```obj-c
typedef void (^EXMPLCredentialManagerPersistentCredentialRightAccessCompletionHandler)(LAPersistedRight *_Nullable persistedRight, NSError *_Nullable error) NS_SWIFT_NAME(EXMPLCredentialManager.PersistentCredentialRightAccessCompletionHandler);

typedef void (^EXMPLCredentialManagerPersistentCredentialRightRemovalCompletionHandler)(NSError *_Nullable error) NS_SWIFT_NAME(EXMPLCredentialManager.PersistentCredentialRightRemovalCompletionHandler);

+ (void)persistedCredentialRightForIdentifier:(NSString *)identifier completionHandler:(EXMPLCredentialManagerPersistentCredentialRightAccessCompletionHandler)completionHandler
{
    [LARightStore.sharedStore rightForIdentifier:identifier completion:completionHandler];
}

+ (void)removeCredentialRightForIdentifier:(NSString *)identifier completionHandler:(EXMPLCredentialManagerPersistentCredentialRightRemovalCompletionHandler)completionHandler
{
    [LARightStore.sharedStore removeRightForIdentifier:identifier completion:completionHandler];
}

+ (void)removeCredentialRight:(LAPersistedRight *)credentialRight completionHandler:(EXMPLCredentialManagerPersistentCredentialRightRemovalCompletionHandler)completionHandler
{
    [LARightStore.sharedStore removeRight:credentialRight completion:completionHandler];
}
```

Ok, so that is all well and good, but how do you even use a `LAPersistedRight`? Well, under the covers, the `LAPersistedRight` has a key pair that you can use to sign data to prove that the data came from the specific user's device (e.g. signing requests) or encrypt/decrypt data:
```obj-c
- (void)createPersistedRight
{
  // 1. Create the persisted right
  [EXMPLCredentialManager createPersistedCredentialRightWithType:EXMPLCredentialRequirementTypeBiometryWithPasscodeFallback identifier:@"com.example.mycredential" completionHandler:^(LAPersistedRight *_Nonnull persistedRight, NSError *_Nullable error) {
    // Store persistedRight to some variable (e.g. myRight)
  }];

  // ...

  // 2. Get the public key
  LAPublicKey *publicKey = myRight.key.publicKey;
  [publicKey exportBytesWithCompletion:^(NSData *_Nullable bytes, NSError * _Nullable exportError) {
    // Save the data to some variable (e.g. publicKeyData)
  }];

  // ...

  // 3. Send the public key to your server
  // NEVER send the private key off the device!
  NSURL *endpoint = [NSURL URLWithString:@"https://www.example.com"];
  NSMutableURLRequest *request = [NSMutableURLRequest requestWithURL:endpoint];
  request.HTTPMethod = @"POST";
  // Content-Length is handled automatically
  [request setValue:@"application/x-www-form-urlencoded; charset=UTF-8" forHTTPHeaderField:@"Content-Type"];
  NSData *requestData = [publicKeyData base64EncodedDataWithOptions:NSDataBase64Encoding64CharacterLineLength];
  NSURLSessionTask *task = [NSURLSession.sharedSession uploadTaskWithRequest:request fromData:requestData completionHandler:^(NSData *_Nullable data, NSURLResponse *_Nullable response, NSError *_Nullable error) {
    // Handle response
  }];
  [task resume];
}

- (void)signData:(NSData *_Nonnull)someData
{
    // Retrieve (or pass in) the LAPersistedRight
    [myRight.key signData:someData secKeyAlgorithm:kSecKeyAlgorithmECIESEncryptionStandardVariableIVX963SHA256AESGCM completion:^(NSData *_Nullable signedData, NSError *_Nullable sigantureError) {
        // Do something with the signed data
    }];
}

- (void)verifyData:(NSData *_Nonnull)someData withSignature:(NSData *_Nonnull)signature
{
    // Retrieve (or pass in) the LAPersistedRight
    [myRight.key.publicKey verifyData:someData signature:signature secKeyAlgorithm:kSecKeyAlgorithmECIESEncryptionStandardVariableIVX963SHA256AESGCM completion:^(NSError *_Nullable signatureError) {
        // Handle success/failure
    }];
}

- (void)encryptData:(NSData *_Nonnull)someData
{
    // Retrieve (or pass in) the LAPersistedRight
    [myRight.key.publicKey encryptData:someData secKeyAlgorithm:kSecKeyAlgorithmECIESEncryptionStandardVariableIVX963SHA256AESGCM completion:^(NSData *_Nullable encryptedData, NSError *_Nullable encryptionError) {
        // Do something with the encrypted data
    }];
}

- (void)decryptData:(NSData *_Nonnull)someData
{
    // Retrieve (or pass in) the LAPersistedRight
    [myRight.key decryptData:someData secKeyAlgorithm:kSecKeyAlgorithmECIESEncryptionStandardVariableIVX963SHA256AESGCM completion:^(NSData *_Nullable decryptedData, NSError *_Nullable decryptionError) {
        // Do something with the decrypted data
    }];
}
```

And there you have it: an easy way to authorize access to parts of an application with built-in authentication as well as an easy way to cryptographically sign and verify data as well as encrypt/decrypt data payloads without interacting with the keychain.
