---
title: "Implementing TLS Pinning with NSURLSession"
subtitle: ""
date: 2018-07-11T17:00:00-05:00
tags: ["tls", "ssl", "NSURLSession"]
---

With the advent of [App Transport Security](https://developer.apple.com/library/archive/documentation/General/Reference/InfoPlistKeyReference/Articles/CocoaKeys.html#//apple_ref/doc/uid/TP40009251-SW33) in iOS 9 and macOS 10.11, Apple began pushing developers into using more secure transportation channels for their applications' data. When an application is linked to those SDKs, the underlying networking stack enforces ATS compliance and the application becomes more secure. ATS's requirements are quite easy to satisfy:

* Connections use [HTTPS](https://en.wikipedia.org/wiki/HTTPS) instead of [HTTP](https://en.wikipedia.org/wiki/Hypertext_Transfer_Protocol)
* Servers must present a [certificate](https://en.wikipedia.org/wiki/Public_key_certificate) from a valid [Certificate Authority](https://en.wikipedia.org/wiki/Certificate_authority)
* The certificate must be signed with a [RSA](https://en.wikipedia.org/wiki/RSA_(cryptosystem)) 2048+ bit or an [ECC](https://en.wikipedia.org/wiki/Elliptic-curve_cryptography) 256+ bit key
* The certificate must have a Secure Hash Algorithm 2 ([SHA-2](https://en.wikipedia.org/wiki/SHA-2)) [digest](https://en.wikipedia.org/wiki/Cryptographic_hash_function) of at least SHA-256
* The certificate must be present in the Certificate Transparency ([CT](https://en.wikipedia.org/wiki/Certificate_Transparency)) logs
* The connection must use Transport Layer Security ([TLS](https://en.wikipedia.org/wiki/Transport_Layer_Security)) 1.2 or higher
* The connection must use [AES-128 or AES-256](https://en.wikipedia.org/wiki/Advanced_Encryption_Standard) symmetric [ciphers](https://en.wikipedia.org/wiki/Cipher_suite) that support perfect forward secrecy ([PFS](https://en.wikipedia.org/wiki/Forward_secrecy)) via Elliptic Curve Diffie-Hellman Ephemeral ([ECDHE](https://en.wikipedia.org/wiki/Elliptic-curve_Diffie%E2%80%93Hellman)) key exchange

Once an application meets these requirements, it is common for the developer to think that they are done securing their application (and by extension their users). However, there is another step that can be taken to add an additional layer of security to the application: [certificate pinning](https://en.wikipedia.org/wiki/Transport_Layer_Security#Certificate_pinning). Certificate pinning is used to ensure that the connection being made is to the right server (i.e. not being [Man-in-the-Middled](https://en.wikipedia.org/wiki/Man-in-the-middle_attack)) as HTTPS only guarantees that the communication channel is secure (private, not trusted). To implement certificate pinning, you must implement [URLSession:didReceiveChallenge:completionHandler:](https://developer.apple.com/documentation/foundation/nsurlsessiondelegate/1409308-urlsession?language=objc) in the [NSURLSessionDelegate](https://developer.apple.com/documentation/foundation/nsurlsessiondelegate?language=objc) by either pinning the whole certificate or by pinning the public key:

## Pinning the Certificate

This method is less common, but just as effective as pinning the public key. To accomplish this, the application must contain the certificate ([DER encoded](https://en.wikipedia.org/wiki/X.690#DER_encoding)) used for the connection.

```obj-c
- (void)URLSession:(NSURLSession *)session didReceiveChallenge:(NSURLAuthenticationChallenge *)challenge completionHandler:(void (^)(NSURLSessionAuthChallengeDisposition, NSURLCredential *_Nullable))completionHandler
{
  NSURLProtectionSpace *protectionSpace = challenge.protectionSpace;
  NSString *authenticationMethod = protectionSpace.authenticationMethod;

  if ([authenticationMethod isEqualToString:NSURLAuthenticationMethodServerTrust]) {
    SecTrustRef trust = protectionSpace.serverTrust; // 1. Get the trust
    SecTrustResultType result = kSecTrustResultFatalTrustFailure;
    OSStatus status = SecTrustEvaluate(trust, &result); // 2. Perform trust evaluation

    if (status == errSecSuccess && (result == kSecTrustResultProceed || result == kSecTrustResultUnspecified)) {
      // Note: If you support connections to more than one endpoint, additional logic is necessary to get the right certificate for the right host
      SecCertificateRef leafCertificate = SecTrustGetCertificateAtIndex(trust, 0); // 3. Get the leaf certificate
      NSData *leafCertificateData = (__bridge_transfer NSData *)SecCertificateCopyData(leafCertificate); // 4. Convert it to its data representation
      NSData *certificateData = [NSData dataWithContentsOfURL:[NSBundle.mainBundle URLForResource:@"certificate" withExtension:@"der"]]; // 5. Load the known correct certificate data

      if ([certificateData isEqualToData:leafCertificateData]) { // 6. Verify the certificates match
        completionHandler(NSURLSessionAuthChallengeUseCredential, [NSURLCredential credentialForTrust:trust]);
      }
      else {
        completionHandler(NSURLSessionAuthChallengeCancelAuthenticationChallenge, nil);
      }
    }
    else {
      completionHandler(NSURLSessionAuthChallengeCancelAuthenticationChallenge, nil);
    }
  }
  else {
    // Handle other authentication methods
  }
}
```

## Pinning the Public Key

The more common approach is to just compare the public key's fingerprint (hash) to a known value rather than ship certificates in the application. However, this requires some more computation on the client side. You can omit information you don't need, but to support all certificates generally, more helper methods are required (for the [ASN.1](https://en.wikipedia.org/wiki/Abstract_Syntax_Notation_One) encoded headers). To compute the fingerprint, [Stack Overflow](https://stackoverflow.com/a/46309453) provides a very in-depth guide.

```obj-c
static const unsigned char rsa2048PublicKeyInfoASN1Header[] = { 0x30, 0x82, 0x01, 0x22, 0x30, 0x0d, 0x06, 0x09, 0x2a, 0x86, 0x48, 0x86, 0xf7, 0x0d, 0x01, 0x01, 0x01, 0x05, 0x00, 0x03, 0x82, 0x01, 0x0f, 0x00 };

static const unsigned char rsa4096PublicKeyInfoASN1Header[] = { 0x30, 0x82, 0x02, 0x22, 0x30, 0x0d, 0x06, 0x09, 0x2a, 0x86, 0x48, 0x86, 0xf7, 0x0d, 0x01, 0x01, 0x01, 0x05, 0x00, 0x03, 0x82, 0x02, 0x0f, 0x00 };

static const unsigned char ecdsasecp256r1PublicKeyInfoASN1Header[] = { 0x30, 0x59, 0x30, 0x13, 0x06, 0x07, 0x2a, 0x86, 0x48, 0xce, 0x3d, 0x02, 0x01, 0x06, 0x08, 0x2a, 0x86, 0x48, 0xce, 0x3d, 0x03, 0x01, 0x07, 0x03, 0x42, 0x00 };

static const unsigned char ecdsasecp384r1PublicKeyInfoASN1Header[] = { 0x30, 0x76, 0x30, 0x10, 0x06, 0x07, 0x2a, 0x86, 0x48, 0xce, 0x3d, 0x02, 0x01, 0x06, 0x05, 0x2b, 0x81, 0x04, 0x00, 0x22, 0x03, 0x62, 0x00 };

static const unsigned char *asn1PublicKeyInfoHeaderBytes(NSString *publicKeyType, NSUInteger publicKeySize)
{
  if ([publicKeyType isEqualToString:(__bridge NSString *)kSecAttrKeyTypeRSA] && publicKeySize == 2048) {
    return rsa2048PublicKeyInfoASN1Header;
  }
  else if ([publicKeyType isEqualToString:(__bridge NSString *)kSecAttrKeyTypeRSA] && publicKeySize == 4096) {
    return rsa4096PublicKeyInfoASN1Header;
  }
  else if ([publicKeyType isEqualToString:(__bridge NSString *)kSecAttrKeyTypeECSECPrimeRandom] && publicKeySize == 256) {
    return ecdsasecp256r1PublicKeyInfoASN1Header;
  }
  else if ([publicKeyType isEqualToString:(__bridge NSString *)kSecAttrKeyTypeECSECPrimeRandom] && publicKeySize == 384) {
    return ecdsasecp384r1PublicKeyInfoASN1Header;
  }
    
  return nil;
}

static size_t asn1PublicKeyInfoHeaderSize(NSString *publicKeyType, NSUInteger publicKeySize)
{
  if ([publicKeyType isEqualToString:(__bridge NSString *)kSecAttrKeyTypeRSA] && publicKeySize == 2048) {
    return sizeof(rsa2048PublicKeyInfoASN1Header);
  }
  else if ([publicKeyType isEqualToString:(__bridge NSString *)kSecAttrKeyTypeRSA] && publicKeySize == 4096) {
    return sizeof(rsa4096PublicKeyInfoASN1Header);
  }
  else if ([publicKeyType isEqualToString:(__bridge NSString *)kSecAttrKeyTypeECSECPrimeRandom] && publicKeySize == 256) {
    return sizeof(ecdsasecp256r1PublicKeyInfoASN1Header);
  }
  else if ([publicKeyType isEqualToString:(__bridge NSString *)kSecAttrKeyTypeECSECPrimeRandom] && publicKeySize == 384) {
    return sizeof(ecdsasecp384r1PublicKeyInfoASN1Header);
  }
    
  return 0;
}

- (void)URLSession:(NSURLSession *)session didReceiveChallenge:(NSURLAuthenticationChallenge *)challenge completionHandler:(void (^)(NSURLSessionAuthChallengeDisposition, NSURLCredential *_Nullable))completionHandler
{
  NSURLProtectionSpace *protectionSpace = challenge.protectionSpace;
  NSString *authenticationMethod = protectionSpace.authenticationMethod;
    
  if ([authenticationMethod isEqualToString:NSURLAuthenticationMethodServerTrust]) {
    SecTrustRef trust = protectionSpace.serverTrust; // 1. Get the trust
    SecTrustResultType result = kSecTrustResultFatalTrustFailure;
    OSStatus status = SecTrustEvaluate(trust, &result); // 2. Perform trust evaluation
        
    if (status == errSecSuccess && (result == kSecTrustResultProceed || result == kSecTrustResultUnspecified)) {
      // Note: If you support connections to more than one endpoint, additional logic is necessary to get the right certificate for the right host
      SecCertificateRef leafCertificate = SecTrustGetCertificateAtIndex(trust, 0); // 3. Get the leaf certificate
      SecKeyRef leafPublicKey = SecCertificateCopyPublicKey(leafCertificate); // 4. Get the public key of the leaf certificate
      NSData *leafPublicKeyData = (__bridge_transfer NSData *)SecKeyCopyExternalRepresentation(leafPublicKey, NULL); // 5. Convert the public key into its data representation
      NSDictionary *leafPublicKeyAttributes = (__bridge_transfer NSDictionary *)SecKeyCopyAttributes(leafPublicKey); // 6. Get the pubic key's attributes
      NSString *leafKeyType = leafPublicKeyAttributes[(__bridge NSString *)kSecAttrKeyType]; // 7. Get the type of the key
      NSUInteger leafKeySize = ((NSNumber *)leafPublicKeyAttributes[(__bridge NSString *)kSecAttrKeySizeInBits]).unsignedIntegerValue; // 8. Get the size of the key

      const unsigned char *headerBytes = asn1PublicKeyInfoHeaderBytes(leafKeyType, leafKeySize); // 9. Get the SubjectPublicKeyInfo header
      size_t headerSize = asn1PublicKeyInfoHeaderSize(leafKeyType, leafKeySize); // 10. Get the SubjectPublicKeyInfo header size

      NSMutableData *publicKeyInfoHash = [NSMutableData dataWithLength:CC_SHA256_DIGEST_LENGTH]; // 11. Create a data object to store the result of the SHA-256 computation
      NSMutableData *publicKeyAndHeaderData = [NSMutableData dataWithBytes:headerBytes length:headerSize]; // 12. Combine the key and header data
      [publicKeyAndHeaderData appendData:leafPublicKeyData];

      CC_SHA256(publicKeyAndHeaderData.bytes, (CC_LONG)publicKeyAndHeaderData.length, (unsigned char *)publicKeyInfoHash.bytes); // 13. Compute the SHA-256 hash

      NSString *computedHash = [publicKeyInfoHash base64EncodedStringWithOptions:kNilOptions]; // 14. Encode the hash as a Base64 string
      NSString *expectedHash = @"your value goes here";

      if ([expectedHash isEqualToString:computedHash]) { // 15. Verify the hashes match
        completionHandler(NSURLSessionAuthChallengeUseCredential, [NSURLCredential credentialForTrust:trust]);
      }
      else {
        completionHandler(NSURLSessionAuthChallengeCancelAuthenticationChallenge, nil);
      }

      if (leafPublicKey) {
        CFRelease(leafPublicKey); // 16. Release the public key
      }
    }
    else {
      completionHandler(NSURLSessionAuthChallengeCancelAuthenticationChallenge, nil);
    }
  }
  else {
    // Handle other authentication methods
  }
}
```

## Closing Thoughts

By implementing certificate pinning in any of the ways mentioned above, an additional layer of security is added to prevent malicious actors from compromising applications or snooping on unsuspecting users. One thing to keep in mind though is that if a certificate is revoked or is replaced (since the old one was about to expire), you must coordinate application releases that can support both certificates or else your application will stop working. Getting your users to update their installation is a whole other problem...
