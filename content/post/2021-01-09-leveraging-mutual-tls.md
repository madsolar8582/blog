---
title: "Leveraging Mutual TLS"
subtitle: ""
date: 2021-01-09T07:00:00-06:00
tags: ["tls", "mtls"]
---

When communicating over [TLS](https://en.wikipedia.org/wiki/Transport_Layer_Security), the client verifies the identity of the server using certificates. However, it is also possible for the server to verify the identity of the client using certificates as well for a more trusted communication channel. This mutual authentication is know as Mutual TLS (mTLS) and while it is a good security measure, it is a bit tricky to setup. 

The main problem is that the client must register itself with the server so that the server know how to verify the clients connecting to it. Typically this is done via a registration process where the service sends a [PKCS12](https://en.wikipedia.org/wiki/PKCS_12) data blob containing the certificate and key for the client to use. This requires the client to know the password to decrypt the data (either this is established during the registration or a shared algorithm is used). Since mTLS is not is use yet, it is technically possible for a malicious actor to intercept the data being transferred. An alternative could be that the client gets its certificate and key via MDM. Regardless, once the client has the certificate and key it can send along identity verification during TLS negotiation.

For iOS applications, you leverage the [Security](https://developer.apple.com/documentation/security?language=objc) framework to process the certificate and keys and then pass it along to [NSURLCredential](https://developer.apple.com/documentation/foundation/nsurlcredential?language=objc) so that it can be used via [NSURLSession](https://developer.apple.com/documentation/foundation/nsurlsession?language=objc).

Let's start with the first part: getting PKCS12 data. From some network response (or from the file system), you end up with an instance of [NSData](https://developer.apple.com/documentation/foundation/nsdata?language=objc) that needs to be converted to something more meaningful. To do this, the [`SecPKCS12Import`](https://developer.apple.com/documentation/security/1396915-secpkcs12import?language=objc) function is used and then we [store](https://developer.apple.com/documentation/security/1401659-secitemadd?language=objc) the [`SecIdentityRef`](https://developer.apple.com/documentation/security/secidentityref?language=objc) to the [keychain](https://developer.apple.com/documentation/security/keychain_services?language=objc) for use later. Note: The examples very basic keychain queries are used; they can be significantly changed depending on the desired security controls.

```obj-c
@import Foundation;
@import Security;

- (BOOL)storeClientCredentials:(NSData *)payload
{
  CFArrayRef items;
  NSString *password = getCredentials(); // This is a C function that knows the password; implementation specific
  OSStatus status = SecPKCS12Import((__bridge CFDataRef)payload, (__bridge CFDictionaryRef)@{(__bridge NSString *)kSecImportExportPassphrase : password}, &items);

  if (status == errSecSuccess) {
    NSDictionary<NSString *, id> *pkcsData = ((__bridge_transfer NSArray *)items).firstObject;

    if (pkcsData) {
      SecIdentityRef identity = (__bridge SecIdentityRef)pkcsData[(__bridge NSString *)kSecImportItemIdentity];
      // Here you could call SecIdentityCopyCertificate and SecIdentityCopyPrivateKey if needed

      NSDictionary *addQuery = @{
        (__bridge NSString *)kSecValueRef : (__bridge id)identity,
        (__bridge NSString *)kSecClass : (__bridge NSString *)kSecClassIdentity,
        (__bridge NSString *)kSecAttrLabel : @"com.example.client-credentials",
      };

      OSStatus addStatus = SecItemAdd((__bridge CFDictionaryRef)addQuery, NULL);
      // Error handling here is optional

      return addStatus == errSecSuccess;
    }
  }
  
  return NO;
}
```

Now we need another method to [return](https://developer.apple.com/documentation/security/1398306-secitemcopymatching?language=objc) the client credentials for use:

```obj-c
- (nullable SecIdentityRef)retrieveClientCredentials
{
  SecIdentityRef identity = nil;
  NSDictionary *getQuery = @{
    (__bridge NSString *)kSecClass : (__bridge NSString *)kSecClassIdentity,
    (__bridge NSString *)kSecAttrLabel : @"com.example.client-credentials",
    (__bridge NSString *)kSecReturnRef : @YES
  };

  OSStatus getStatus = SecItemCopyMatching((__bridge CFDictionaryRef)getQuery, (CFTypeRef *)&identity);
  // Error handling here is optional

  return getStatus == errSecSuccess ? identity : nil;
}
```

Lastly, we need to use the certificate and key (packaged as an identity) in the [`URLSession:didReceiveChallenge:completionHandler:`](https://developer.apple.com/documentation/foundation/nsurlsessiondelegate/1409308-urlsession?language=objc) method to create the [credential](https://developer.apple.com/documentation/foundation/nsurlcredential/1428192-credentialwithidentity?language=objc):

```obj-c
- (void)URLSession:(NSURLSession *)session didReceiveChallenge:(NSURLAuthenticationChallenge *)challenge completionHandler:(void (^)(NSURLSessionAuthChallengeDisposition, NSURLCredential *_Nullable))completionHandler
{
  NSURLProtectionSpace *protectionSpace = challenge.protectionSpace;
  NSString *authenticationMethod = protectionSpace.authenticationMethod;

  // Ensure that our server is being requested  
  if ([authenticationMethod isEqualToString:NSURLAuthenticationMethodClientCertificate] && [protectionSpace.host isEqualToString:@"www.example.com"]) {
    SecIdentityRef identity = [self retrieveClientCredentials];

    if (identity) {
      // You need to extract the intermediary certificates as well if the server does not have them (typically it does)
      NSURLCredential *credential = [NSURLCredential credentialWithIdentity:identity certificates:nil persistence:NSURLCredentialPersistenceForSession];
      completionHandler(NSURLSessionAuthChallengeUseCredential, credential);
      CFRelease(identity);
    }
    else {
      completionHandler(NSURLSessionAuthChallengeCancelAuthenticationChallenge, nil); // No certificate, so authentication needs to fail
    }
  }
  else {
    // Other method handling
  }
```

---

Combining mTLS with additional security measures helps improve your application's security posture so you shouldn't solely rely on it. For best results, [TLS 1.3](https://tlswg.org/tls13-spec/draft-ietf-tls-rfc8446bis.html), [TLS Pinning](https://owasp.org/www-community/controls/Certificate_and_Public_Key_Pinning), and [HTTP/2](https://http2.github.io/) should be leveraged to ensure that security cannot be compromised by older protocols.
