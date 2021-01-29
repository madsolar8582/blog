---
title: "Signing Requests"
subtitle: ""
date: 2021-01-29T11:00:00-06:00
tags: ["digital signatures"]
---

When sending requests to a server, you do not want a malicious actor to tamper with the payload and try to change what the server should do. To add a layer of security to the request, you can add a cryptographic digital signature to the request headers (encoded with [Base64](https://en.wikipedia.org/wiki/Base64)). For iOS devices, [ECC](https://en.wikipedia.org/wiki/Elliptic_curve_cryptography) is preferred and the generated keys can be stored in the [secure enclave](https://support.apple.com/guide/security/secure-enclave-overview-sec59b0b31ff/web). Using these technologies, requests can be protected with [ECDSA](https://en.wikipedia.org/wiki/Elliptic_Curve_Digital_Signature_Algorithm).

To start, a client can generate a key pair via [`SecKeyCreateRandomKey`](https://developer.apple.com/documentation/security/1823694-seckeycreaterandomkey?language=objc) (the example below) or the server can provide a certificate via a [PKCS12](https://en.wikipedia.org/wiki/PKCS_12) data blob. If the server provides the certificate, then additional security measures are required to secure the exchange (e.g. [TLS pinning](https://owasp.org/www-community/controls/Certificate_and_Public_Key_Pinning)). Regardless, once you have your public/private key pair, you need to store it in the secure enclave:

```obj-c
@import Foundation;
@import Security;

static void generateKeyPair(NSString *tag)
{
  // Extra security could be added by adding kSecAccessControlBiometryCurrentSet to the SecAccessControlCreateFlags
  // Could also reduce security by changing the protection attribute to kSecAttrAccessibleWhenUnlockedThisDeviceOnly
  CFErrorRef aclCFError = NULL;
  SecAccessControlRef acl = SecAccessControlCreateWithFlags(kCFAllocatorDefault, kSecAttrAccessibleWhenPasscodeSetThisDeviceOnly, kSecAccessControlPrivateKeyUsage, &aclCFError);
  NSError *aclError = (__bridge_transfer NSError *)aclCFError;
    
  if (aclError) {
    // handle error
  }
    
  NSDictionary<NSString *, id> *attrs = @{
    (__bridge NSString *)kSecAttrKeyType : (__bridge NSString *)kSecAttrKeyTypeECSECPrimeRandom,
    (__bridge NSString *)kSecAttrKeySizeInBits : @256, // Can't use kSecp256r1 constant because it isn't available on iOS...
    (__bridge NSString *)kSecAttrTokenID : (__bridge NSString *)kSecAttrTokenIDSecureEnclave, // Because we are using the enclave, key size is limited to 256 bits
    (__bridge NSString *)kSecPrivateKeyAttrs : @{
      (__bridge NSString *)kSecAttrIsPermanent : @YES,
      (__bridge NSString *)kSecAttrApplicationTag : tag, // Needs to be unique if creating multiple
      (__bridge NSString *)kSecAttrAccessControl : (__bridge id)acl
    }
  };
    
  CFErrorRef keyCFError = NULL;
  SecKeyRef privateKey = SecKeyCreateRandomKey((__bridge CFDictionaryRef)attrs, &keyCFError);
  NSError *keyError = (__bridge_transfer NSError *)keyCFError;
    
  if (keyError) {
    // Handler error
  }
    
  CFRelease(acl);
  if (privateKey) {
    CFRelease(privateKey);
  }
}
```

Once the key pair is stored, you need to create a way to retrieve it so that you can use it for signing:

```obj-c
static SecKeyRef getPrivateKey(NSString *tag)
{
  // Simple query that can be refined
  NSDictionary<NSString *, id> *query = @{
    (__bridge NSString *)kSecClass : (__bridge NSString *)kSecClassKey,
    (__bridge NSString *)kSecAttrApplicationTag : tag,
    (__bridge NSString *)kSecAttrKeyType : (__bridge NSString *)kSecAttrKeyTypeECSECPrimeRandom,
    (__bridge NSString *)kSecReturnRef : @YES
  };
    
  SecKeyRef privateKey = NULL;
  OSStatus status = SecItemCopyMatching((__bridge CFDictionaryRef)query, (CFTypeRef *)&privateKey);
    
  if (status != errSecSuccess) {
    // Handle failure, e.g. SecCopyErrorMessageString
  }
    
  return privateKey;
}
```

You can then use the private key to sign data payloads using [`SecKeyCreateSignature`](https://developer.apple.com/documentation/security/1643916-seckeycreatesignature?language=objc). In the example below, the algorithm used generates a [message digest](https://en.wikipedia.org/wiki/Cryptographic_hash_function) automatically:

```obj-c
static NSString *_Nullable signData(NSData *data)
{
  SecKeyRef privateKey = getPrivateKey();
  if (!privateKey) {
    // Handle error
    return nil;
  }
    
  BOOL canSign = SecKeyIsAlgorithmSupported(privateKey, kSecKeyOperationTypeSign, kSecKeyAlgorithmECDSASignatureMessageX962SHA512);
    
  if (!canSign) {
    // Handle error
    CFRelease(privateKey);
    return nil;
  }
    
  CFErrorRef sigCFError = NULL;
  NSData *rawSignature = (__bridge_transfer NSData *)SecKeyCreateSignature(privateKey, kSecKeyAlgorithmECDSASignatureMessageX962SHA512, (__bridge CFDataRef)data, &sigCFError);
  NSError *sigError = (__bridge_transfer NSError *)sigCFError;
    
  if (sigError) {
    // Handle error
    CFRelease(privateKey);
    return nil;
  }
    
  CFRelease(privateKey);
  return [rawSignature base64EncodedStringWithOptions:kNilOptions];
}
```

With the encoded signature, you then add that to a request:

```obj-c
- (void)someMethod
{
  NSDictionary *payload = @{@"test" : @"test"};
  NSData *data = [NSJSONSerialization dataWithJSONObject:payload options:kNilOptions error:nil];
  NSString *signature = signData(data);
    
  NSMutableURLRequest *request = [NSMutableURLRequest requestWithURL:[NSURL URLWithString:@"https://www.example.com/"]];
  request.HTTPMethod = @"POST";
  request.HTTPBody = data;
  [request setValue:signature forHTTPHeaderField:@"signature"];
    
  NSURLSessionDataTask *task = [NSURLSession.sharedSession dataTaskWithRequest:request completionHandler:^(NSData *_Nullable data, NSURLResponse *_Nullable response, NSError *_Nullable error) {
    // Handle response
  }];
  [task resume];
}
```

---

Obviously, the next step for even more security would be to add signature verification to server responses using the same technique. To do so, you need the trusted public key of the server and you can use [`SecKeyVerifySignature`](https://developer.apple.com/documentation/security/1643715-seckeyverifysignature?language=objc).

An additional note is that if Mutual TLS is in use, this kind of strategy is pretty much obviated.
