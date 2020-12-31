---
title: "Camera and Microphone in WKWebView"
subtitle: ""
date: 2021-01-01T07:00:00-05:00
tags: ["wkwebview", "getUserMedia", "camera", "microphone", "webrtc"]
---

With the release of iOS 14.3, Apple has [made available](https://webkit.org/blog/11353/mediarecorder-api/) the camera and microphone in [WKWebView](https://developer.apple.com/documentation/webkit/wkwebview?language=objc). This now allows applications with a focus on web content as well as other browsers on iOS to leverage WebRTC (amongst other things). 

To use the camera and microphone, applications must declare `NSCameraUsageDescription` and `NSMicrophoneUsageDescription` in the Info.plist and request permission using [AVFoundation](https://developer.apple.com/documentation/avfoundation/cameras_and_media_capture/requesting_authorization_for_media_capture_on_ios?language=objc). Once that is done, you are free to use the camera and microphone in web content with the right configuration. An example is below:

### Shared WKProcessPool

This process pool ensures that all instances of WKWebView share the same process and memory.

#### Header

```obj-c
@import Foundation;
@import WebKit;

NS_ASSUME_NONNULL_BEGIN

/**
 * @class DemoProcessPool
 * The process pool to be used by all WKWebViews in the application.
 */
@interface DemoProcessPool : WKProcessPool

/**
 * The process pool to be used by all WKWebViews in the application.
 */
@property (nonatomic, strong, class, readonly) DemoProcessPool *sharedProcessPool;

@end

NS_ASSUME_NONNULL_END
```

#### Implementation

```obj-c
#import "DemoProcessPool.h"

@implementation DemoProcessPool

#pragma mark - Private API

- (instancetype)__init
{
    self = [super init];
    return self;
}

#pragma mark - Public API

+ (DemoProcessPool *)sharedProcessPool
{
    static DemoProcessPool *sharedPool = nil;
    static dispatch_once_t onceToken = 0;
    dispatch_once(&onceToken, ^{
        sharedPool = [[self alloc] __init];
    });
    
    return sharedPool;
}

@end
```

### Shared WKWebViewConfiguration

This configuration ensures that all instances of WKWebView share the same settings.

#### Header

```obj-c
@import Foundation;
@import WebKit;

NS_ASSUME_NONNULL_BEGIN

/**
 * @class DemoConfiguration
 * The configuration to be used by all WKWebViews in the application.
 */
@interface DemoConfiguration : WKWebViewConfiguration

/**
 * The configuration to be used by all WKWebViews in the application.
 */
@property (nonatomic, copy, class, readonly) DemoConfiguration *sharedConfiguration;

@end

NS_ASSUME_NONNULL_END
```

#### Implementation

```obj-c
#import "DemoConfiguration.h"
#import "DemoProcessPool.h"

@implementation DemoConfiguration

#pragma mark - Private API

- (instancetype)__init
{
    if ((self = [super init])) {
        self.processPool = DemoProcessPool.sharedProcessPool;
        self.suppressesIncrementalRendering = YES;
        self.dataDetectorTypes = WKDataDetectorTypeAll;
        self.allowsInlineMediaPlayback = YES;
        self.mediaTypesRequiringUserActionForPlayback = WKAudiovisualMediaTypeNone;
        self.websiteDataStore = [WKWebsiteDataStore defaultDataStore];
        self.limitsNavigationsToAppBoundDomains = NO;
    }
    
    return self;
}

#pragma mark - Object Life Cycle

- (instancetype)init
{
    return [self __init]; // Required because WebKit calls this on copy operations
}

#pragma mark - Public API

+ (DemoConfiguration *)sharedConfiguration
{
    static DemoConfiguration *sharedConfig = nil;
    static dispatch_once_t onceToken = 0;
    dispatch_once(&onceToken, ^{
        sharedConfig = [[self alloc] __init];
    });
    
    return sharedConfig;
}

@end
```

### Sample Controller

This controller simply requests the appropriate permissions and then loads a sample page in a web view that takes up the entire screen.

```obj-c
#import "DemoViewController.h"
@import AVFoundation;
#import "DemoConfiguration.h"

@interface DemoViewController () <WKNavigationDelegate, WKUIDelegate>

@property (nonatomic, strong) WKWebView *webView;

@end

@implementation DemoViewController

#pragma mark - Object Life Cycle

- (void)dealloc
{
    if (self.webView) {
        [self.webView stopLoading];
        self.webView.navigationDelegate = nil;
        self.webView.UIDelegate = nil;
        [self.webView removeFromSuperview];
    }
}

#pragma mark - View Life Cycle

- (void)viewDidLoad
{
    [super viewDidLoad];
    self.edgesForExtendedLayout = UIRectEdgeNone;
    self.view.backgroundColor = UIColor.systemBackgroundColor;
    self.navigationController.view.backgroundColor = UIColor.systemBackgroundColor;
    [self setupWebView];
    [self.view addSubview:self.webView];
}

- (void)viewWillAppear:(BOOL)animated
{
    [super viewWillAppear:animated];
    [self setupWebViewConstraints];
    [self.view layoutIfNeeded];
}

- (void)viewDidAppear:(BOOL)animated
{
    [super viewDidAppear:animated];
    
    AVAuthorizationStatus audioStatus = [AVCaptureDevice authorizationStatusForMediaType:AVMediaTypeAudio];
    AVAuthorizationStatus videoStatus = [AVCaptureDevice authorizationStatusForMediaType:AVMediaTypeVideo];
    
    __weak __auto_type weakSelf = self;
    if (audioStatus != AVAuthorizationStatusAuthorized) {
        [self requestMicrophonePermissionWithCompletionHandler:^{
            if (videoStatus == AVAuthorizationStatusAuthorized) {
                dispatch_async(dispatch_get_main_queue(), ^{
                    if (!weakSelf.webView.URL) {
                        [weakSelf loadDemoURL];
                    }
                });
            }
            else {
                [weakSelf requestCameraPermissionWithCompletionHandler:^{
                    dispatch_async(dispatch_get_main_queue(), ^{
                        if (!weakSelf.webView.URL) {
                            [weakSelf loadDemoURL];
                        }
                    });
                }];
            }
        }];
    }
    else if (audioStatus == AVAuthorizationStatusAuthorized && videoStatus != AVAuthorizationStatusAuthorized) {
        [self requestCameraPermissionWithCompletionHandler:^{
            dispatch_async(dispatch_get_main_queue(), ^{
                if (!weakSelf.webView.URL) {
                    [weakSelf loadDemoURL];
                }
            });
        }];
    }
    else if (audioStatus == AVAuthorizationStatusAuthorized && videoStatus == AVAuthorizationStatusAuthorized) {
        if (!self.webView.URL) {
            [self loadDemoURL];
        }
    }
}

#pragma mark - Private API

- (void)setupWebViewConstraints
{
    UILayoutGuide *safeArea = self.view.safeAreaLayoutGuide;
    [NSLayoutConstraint activateConstraints:@[
        [self.webView.leadingAnchor constraintEqualToAnchor:safeArea.leadingAnchor],
        [self.webView.trailingAnchor constraintEqualToAnchor:safeArea.trailingAnchor],
        [self.webView.topAnchor constraintEqualToAnchor:safeArea.topAnchor],
        [self.webView.bottomAnchor constraintEqualToAnchor:safeArea.bottomAnchor],
    ]];
}

- (void)setupWebView
{
    self.webView = [[WKWebView alloc] initWithFrame:(self.viewIfLoaded ? self.view.frame : CGRectZero) configuration:DemoConfiguration.sharedConfiguration];
    self.webView.scrollView.bounces = YES;
    self.webView.navigationDelegate = self;
    self.webView.UIDelegate = self;
    self.webView.translatesAutoresizingMaskIntoConstraints = NO;
}

- (void)requestMicrophonePermissionWithCompletionHandler:(void (^)(void))completionHandler
{
    [AVCaptureDevice requestAccessForMediaType:AVMediaTypeAudio completionHandler:^(BOOL granted) {
        if (granted) {
            completionHandler();
        }
    }];
}

- (void)requestCameraPermissionWithCompletionHandler:(void (^)(void))completionHandler
{
    [AVCaptureDevice requestAccessForMediaType:AVMediaTypeVideo completionHandler:^(BOOL granted) {
        if (granted) {
            completionHandler();
        }
    }];
}

- (void)loadDemoURL
{
    // Can also use https://webrtc.github.io/samples/src/content/devices/input-output/
    NSURL *url = [NSURL URLWithString:@"https://webrtc.github.io/samples/src/content/peerconnection/pc1/"];
    [self.webView loadRequest:[NSURLRequest requestWithURL:url cachePolicy:NSURLRequestReloadRevalidatingCacheData timeoutInterval:15]];
}

#pragma mark - WKNavigationDelegate

- (void)webView:(WKWebView *)webView decidePolicyForNavigationAction:(WKNavigationAction *)navigationAction preferences:(WKWebpagePreferences *)preferences decisionHandler:(void (^)(WKNavigationActionPolicy, WKWebpagePreferences *_Nonnull))decisionHandler
{
    if (decisionHandler) {
        decisionHandler(WKNavigationActionPolicyAllow, preferences);
    }
}

- (void)webView:(WKWebView *)webView didStartProvisionalNavigation:(null_unspecified WKNavigation *)navigation
{
    
}

- (void)webView:(WKWebView *)webView didFinishNavigation:(null_unspecified WKNavigation *)navigation
{
    
}

- (void)webView:(WKWebView *)webView didFailProvisionalNavigation:(null_unspecified WKNavigation *)navigation withError:(NSError *)error
{
    
}

- (void)webView:(WKWebView *)webView didFailNavigation:(null_unspecified WKNavigation *)navigation withError:(NSError *)error
{
    
}

- (void)webView:(WKWebView *)webView didReceiveAuthenticationChallenge:(NSURLAuthenticationChallenge *)challenge completionHandler:(void (^)(NSURLSessionAuthChallengeDisposition disposition, NSURLCredential *_Nullable credential))completionHandler
{
    NSURLProtectionSpace *protectionSpace = challenge.protectionSpace;
    NSString *authenticationMethod = challenge.protectionSpace.authenticationMethod;
    NSURLSessionAuthChallengeDisposition disposition = NSURLSessionAuthChallengePerformDefaultHandling;
    NSURLCredential *credential = nil;
    
    if ([authenticationMethod isEqualToString:NSURLAuthenticationMethodServerTrust]) {
        SecTrustRef trust = protectionSpace.serverTrust;
        BOOL proceed = SecTrustEvaluateWithError(trust, nil);
        if (proceed) {
            disposition = NSURLSessionAuthChallengeUseCredential;
            credential = [NSURLCredential credentialForTrust:trust];
        }
        else {
            disposition = NSURLSessionAuthChallengeCancelAuthenticationChallenge;
        }
    }
    
    if (completionHandler) {
        completionHandler(disposition, (credential ?: challenge.proposedCredential));
    }
}

- (void)webView:(WKWebView *)webView authenticationChallenge:(NSURLAuthenticationChallenge *)challenge shouldAllowDeprecatedTLS:(void (^)(BOOL))decisionHandler
{
    if (decisionHandler) {
        decisionHandler(NO);
    }
}

#pragma mark - WKUIDelegate

- (void)webView:(WKWebView *)webView runJavaScriptAlertPanelWithMessage:(NSString *)message initiatedByFrame:(WKFrameInfo *)frame completionHandler:(void (^)(void))completionHandler
{
    UIAlertController *alert = [UIAlertController alertControllerWithTitle:nil message:message preferredStyle:UIAlertControllerStyleAlert];
    UIAlertAction *okAction = [UIAlertAction actionWithTitle:@"OK" style:UIAlertActionStyleDefault handler:^(UIAlertAction *_Nonnull action) {
        if (completionHandler) {
            completionHandler();
        }
    }];
    
    [alert addAction:okAction];
    [self presentViewController:alert animated:YES completion:nil];
}

- (void)webView:(WKWebView *)webView runJavaScriptConfirmPanelWithMessage:(NSString *)message initiatedByFrame:(WKFrameInfo *)frame completionHandler:(void (^)(BOOL))completionHandler
{
    UIAlertController *alert = [UIAlertController alertControllerWithTitle:nil message:message preferredStyle:UIAlertControllerStyleAlert];
    UIAlertAction *okAction = [UIAlertAction actionWithTitle:@"OK" style:UIAlertActionStyleDefault handler:^(UIAlertAction *_Nonnull action) {
        if (completionHandler) {
            completionHandler(YES);
        }
    }];
    
    UIAlertAction *cancelAction = [UIAlertAction actionWithTitle:@"Cancel" style:UIAlertActionStyleCancel handler:^(UIAlertAction *_Nonnull action) {
        if (completionHandler) {
            completionHandler(NO);
        }
    }];
    
    [alert addAction:okAction];
    [alert addAction:cancelAction];
    [self presentViewController:alert animated:YES completion:nil];
    
}

- (void)webView:(WKWebView *)webView runJavaScriptTextInputPanelWithPrompt:(NSString *)prompt defaultText:(nullable NSString *)defaultText initiatedByFrame:(WKFrameInfo *)frame completionHandler:(void (^)(NSString *_Nullable))completionHandler
{
    UIAlertController *alert = [UIAlertController alertControllerWithTitle:nil message:prompt preferredStyle:UIAlertControllerStyleAlert];
    [alert addTextFieldWithConfigurationHandler:^(UITextField *_Nonnull textField) {
        textField.text = defaultText;
    }];
    
    __weak __auto_type weakAlert = alert;
    UIAlertAction *okAction = [UIAlertAction actionWithTitle:@"OK" style:UIAlertActionStyleDefault handler:^(UIAlertAction *_Nonnull action) {
        NSString *text = weakAlert.textFields.firstObject.text ?: defaultText;
        
        if (completionHandler) {
            completionHandler(text);
        }
    }];
    
    UIAlertAction *cancelAction = [UIAlertAction actionWithTitle:@"Cancel" style:UIAlertActionStyleCancel handler:^(UIAlertAction *_Nonnull action) {
        if (completionHandler) {
            completionHandler(nil);
        }
    }];
    
    [alert addAction:okAction];
    [alert addAction:cancelAction];
    [self presentViewController:alert animated:YES completion:nil];
}

@end
```

---

One remaining issue that needs to be cleaned up is that each time the web view goes to use the camera or microphone, Apple requires the user to address a permission prompt even though the application has already been granted access.
