---
title: "Disappearing SFAuthenticationSession"
subtitle: ""
date: 2018-08-23T18:00:00-05:00
tags: ["bug", "sfauthenticationsession", "aswebauthenticationsession"]
---

If you have an iOS application that contains sensitive information, you should be obscuring it somehow when the user leaves the application so that the system does not save a screenshot of the content and so that it is not viewable in the app switcher. Typically, you would do this by replacing the view hierarchy of the application's window with a view controller with some static content and then restore it when the user returns. However, you can also swap key windows to avoid the complexity of managing a view hierarchy that you may not be in complete control of.

An interesting side effect of window swapping on iOS 12 is that it renders [SFAuthenticationSession](https://developer.apple.com/documentation/safariservices/sfauthenticationsession?language=objc)/[ASWebAuthenticationSession](https://developer.apple.com/documentation/authenticationservices/aswebauthenticationsession?language=objc) unusable as the presentation of the remote view controller fails. I'm not sure if the window swap cancels the presentation or if it presents on the placeholder window, but, it prevents user login nonetheless. 

To reproduce this, you only need a small amount of code. In the sample below, it contains an application delegate and view controller. The view controller triggers the presentation of the authentication workflow and the application delegate controls the window flows. As far as the actual UI is concerned, you only really need background colors to differentiate the application and the placeholder. The easiest way of doing this is to make the placeholder have a dark background and then put a [UIVisualEffectsView](https://developer.apple.com/documentation/uikit/uivisualeffectview?language=objc) on it with a dark blur effect.

### App Delegate
```obj-c
#import "AppDelegate.h"

@implementation UIResponder (FirstResponder)

- (void)__identifyFirstResponder:(NSMutableArray<id> *)sender
{
    [sender addObject:self];
}

@end

@implementation UIApplication (FirstResponder)

- (nullable id)currentFirstResponder
{
    NSMutableArray<id> *container = [[NSMutableArray alloc] initWithCapacity:1];
    [self sendAction:@selector(__identifyFirstResponder:) to:nil from:container forEvent:nil];
    return container.lastObject;
}

@end

@interface AppDelegate ()

@property (nonatomic, strong) UIWindow *placeholderWindow;

@property (nonatomic, strong) UIWindow *applicationWindow;

@property (nonatomic, weak) UIResponder *oldFirstResponder;

@end

@implementation AppDelegate

#pragma mark - UIApplicationDelegate

- (void)applicationWillResignActive:(UIApplication *)application
{
    if (self.placeholderWindow) {
        self.placeholderWindow.windowLevel = UIWindowLevelNormal;
        self.placeholderWindow.hidden = YES;
    }
    
    self.applicationWindow = UIApplication.sharedApplication.keyWindow;
    self.oldFirstResponder = [UIApplication.sharedApplication currentFirstResponder];
    [self.oldFirstResponder resignFirstResponder];
    
    self.placeholderWindow = [self createPlaceholderWindow];
    [self.placeholderWindow makeKeyAndVisible];
    
    [self disappearApplicationWindow];
}

- (void)applicationDidBecomeActive:(UIApplication *)application
{
    if (self.placeholderWindow) {
        [self appearApplicationWindow];
    }
}

#pragma mark - Helpers

- (void)disappearApplicationWindow
{
    self.applicationWindow.hidden = YES;
}

- (void)appearApplicationWindow
{
    self.applicationWindow.hidden = NO;
    self.placeholderWindow.windowLevel = UIWindowLevelNormal;
    [self.applicationWindow makeKeyAndVisible];
    
    self.placeholderWindow.hidden = YES;
    self.placeholderWindow = nil;
    
    [self.oldFirstResponder becomeFirstResponder];
}

- (nonnull UIWindow *)createPlaceholderWindow
{
    UIWindow *placeholderWindow = [[UIWindow alloc] initWithFrame:UIApplication.sharedApplication.keyWindow.frame];
    UIViewController *blurController = [[UIStoryboard storyboardWithName:@"Main" bundle:NSBundle.mainBundle] instantiateViewControllerWithIdentifier:@"blurView"];
    placeholderWindow.rootViewController = blurController;
    placeholderWindow.windowLevel = UIWindowLevelNormal + 100;
    
    return placeholderWindow;
}

@end
```

### View Controller
```obj-c
#import "ViewController.h"
@import SafariServices;
#if __IPHONE_OS_VERSION_MAX_ALLOWED >= 120000
@import AuthenticationServices;
#endif

@interface ViewController ()
#if __IPHONE_OS_VERSION_MAX_ALLOWED >= 120000
@property (nonatomic, strong) ASWebAuthenticationSession *authSession API_AVAILABLE(ios(12.0));
#endif
@property (nonatomic, strong) SFAuthenticationSession *safariSession;

@end

@implementation ViewController

- (void)viewDidAppear:(BOOL)animated
{
    [super viewDidAppear:animated];
    
    NSURL *launchURL = [NSURL URLWithString:@"https://www.example.com/"];
#if __IPHONE_OS_VERSION_MAX_ALLOWED >= 120000
    if (@available(iOS 12, *)) {
        ASWebAuthenticationSession *session = [[ASWebAuthenticationSession alloc] initWithURL:launchURL callbackURLScheme:nil completionHandler:^(NSURL * _Nullable callbackURL, NSError * _Nullable error) {
            
        }];
        self.authSession = session;
        [session start];
    }
    else {
#endif
        SFAuthenticationSession *session = [[SFAuthenticationSession alloc] initWithURL:launchURL callbackURLScheme:nil completionHandler:^(NSURL * _Nullable callbackURL, NSError * _Nullable error) {
            
        }];
        self.safariSession = session;
        [session start];
#if __IPHONE_OS_VERSION_MAX_ALLOWED >= 120000
    }
#endif
}

@end
```

With that implemented, this is what you get:
<center>
<video controls height="600" width="400">
  <source src="/videos/DisappearingSFAuthenticationSession.mp4" type="video/mp4">
  <source src="/videos/DisappearingSFAuthenticationSession.webm" type="video/webm">
</video>
</center>

Obviously, this isn't good, so I logged this as radar 43523219. Hopefully, it'll get fixed soon™.
