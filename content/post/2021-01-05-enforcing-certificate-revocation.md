---
title: "Enforcing Certificate Revocation"
subtitle: ""
date: 2021-01-05T12:00:00-06:00
tags: ["crl", "ocsp"]
---

In addition to [TLS Pinning](https://cheatsheetseries.owasp.org/cheatsheets/Pinning_Cheat_Sheet.html), you can also enforce that the certificate in use has not been revoked by checking the [CRL](https://en.wikipedia.org/wiki/Certificate_revocation_list) or [OCSP](https://en.wikipedia.org/wiki/Online_Certificate_Status_Protocol) result for said certificate. To do this for [NSURLSession](https://developer.apple.com/documentation/foundation/nsurlsession?language=objc), you need to add an additional [`SecPolicyRef`](https://developer.apple.com/documentation/security/secpolicyref?language=objc) to the [`SecTrustRef`](https://developer.apple.com/documentation/security/sectrustref?language=objc) provided to you during the authentication challenge. The new policy needs to be created via [SecPolicyCreateRevocation](https://developer.apple.com/documentation/security/1400026-secpolicycreaterevocation?language=objc) and can be [tweaked](https://developer.apple.com/documentation/security/1563600-revocation_policy_constants?language=objc) depending on how strict you want to be. Once you have the new policy, add it to the trust and then evaluate it. Assuming the evaluation passes, you can proceed with the transaction.

### Example

```obj-c
- (void)URLSession:(NSURLSession *)session didReceiveChallenge:(NSURLAuthenticationChallenge *)challenge completionHandler:(void (^)(NSURLSessionAuthChallengeDisposition, NSURLCredential *_Nullable))completionHandler
{
  NSURLProtectionSpace *protectionSpace = challenge.protectionSpace;
  NSString *authenticationMethod = protectionSpace.authenticationMethod;
  NSURLSessionAuthChallengeDisposition disposition = NSURLSessionAuthChallengePerformDefaultHandling;
  NSURLCredential *credential = nil;

  if ([authenticationMethod isEqualToString:NSURLAuthenticationMethodServerTrust]) {
    SecTrustRef trust = protectionSpace.serverTrust;
    CFErrorRef error = nil;
    CFArrayRef policies = nil;
    SecPolicyRef revPolicy = nil;
    OSStatus polStatus = SecTrustCopyPolicies(trust, &policies); // Get the original policy which should be a SSL policy (not X.509)

    // It is up to you on how you want to handle failure; The example allows normal validation to proceed    
    if (polStatus != errSecSuccess) {
      NSString *polErrMsg = (__bridge_transfer NSString *)SecCopyErrorMessageString(polStatus, NULL) ?: @"Unknown";
      os_log_error(OS_LOG_DEFAULT, "Unable to create additional policies for TLS evaluation: %{public}@", polErrMsg);
    }
    else {
      CFMutableArrayRef mPolicies = CFArrayCreateMutableCopy(kCFAllocatorDefault, (CFArrayGetCount(policies) + 1), policies);
      CFRelease(policies);
      SecPolicyRef sslPolicy = (SecPolicyRef)CFArrayGetValueAtIndex(mPolicies, 0);
      CFDictionaryRef props = SecPolicyCopyProperties(sslPolicy);
      CFStringRef polType = (CFStringRef)CFDictionaryGetValue(props, kSecPolicyName);
            
      if (polType && CFEqual(kSecPolicyAppleSSL, polType)) { // Only apply revocation to SSL validation
        revPolicy = SecPolicyCreateRevocation(kSecRevocationUseAnyAvailableMethod | kSecRevocationRequirePositiveResponse); // Can change this to only use CRLs or OCSP
                
        if (revPolicy) {
          CFArrayAppendValue(mPolicies, revPolicy);
          OSStatus addStatus = SecTrustSetPolicies(trust, mPolicies);
                    
          if (addStatus != errSecSuccess) {
            NSString *addErrMsg = (__bridge_transfer NSString *)SecCopyErrorMessageString(addStatus, NULL) ?: @"Unknown";
            os_log_error(OS_LOG_DEFAULT, "Unable to add additional policies for TLS evaluation: %{public}@", addErrMsg);
          }
        }
      }
            
      if (revPolicy) {
        CFRelease(revPolicy);
      }
            
      if (props) {
        CFRelease(props);
      }
            
      if (mPolicies) {
        CFRelease(mPolicies);
      }
    }
        
    BOOL proceed = SecTrustEvaluateWithError(trust, &error); // Since this is blocking, call off of main or use SecTrustEvaluateAsyncWithError
    NSError *realError = (__bridge_transfer NSError *)error;
        
    if (proceed) {
      disposition = NSURLSessionAuthChallengeUseCredential;
      credential = [NSURLCredential credentialForTrust:trust];
    }
    else {
      os_log_error(OS_LOG_DEFAULT, "Unable to evaluate TLS trust chain. Error: %{public}@", realError.debugDescription);
      disposition = NSURLSessionAuthChallengeCancelAuthenticationChallenge;
    }
  }
  else {
    // Other authentication method handling
  }

  if (completionHandler) {
    completionHandler(disposition, (credential ?: challenge.proposedCredential));
  }
}
```

---

Combining TLS Pinning and certificate revocation status enforcement alongside the default system trust process (which includes [certificate transparency](https://en.wikipedia.org/wiki/Certificate_Transparency)), you end up with a solid defense to [man-in-the-middle](https://en.wikipedia.org/wiki/Man-in-the-middle_attack) attacks. However, if the device is jailbroken, additional defenses are needed.
