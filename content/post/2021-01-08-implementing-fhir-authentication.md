---
title: "Implementing FHIR Authentication"
subtitle: ""
date: 2021-01-08T10:00:00-06:00
tags: ["smart on fhir", "fhir", "smart", "ehr"]
---

Patients and Providers can access [SMART](https://smarthealthit.org/) (Substitutable Medical Applications, Reusable Technologies) applications like Apple's [Health app](https://www.apple.com/healthcare/health-records/) or open Electronic Health Record ([EHR](https://en.wikipedia.org/wiki/Electronic_health_record)) software through [FHIR](https://hl7.org/fhir/) (Fast Healthcare Interoperability Resources). This allows for the sharing of health data across multiple platforms via an open standard. 

For mobile applications, accessing health data for read and/or write access requires being registered with the particular FHIR instance (e.g. [Cerner](https://fhir.cerner.com/)) and requesting the correct scopes. Once authorization has been granted, you use [JSON Web Tokens](https://jwt.io/introduction) to call specific endpoints for data. This is typically done via [Authorization headers](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Authorization), so you can create a custom [NSURLProtocol](https://developer.apple.com/documentation/foundation/nsurlprotocol?language=objc) to apply the header to all requests to FHIR endpoints. However, most SMART applications are web-based, so you will need to implement request modification for [WKWebView](https://developer.apple.com/documentation/webkit/wkwebview?language=objc). Additionally, [OpenID Connect](https://openid.net/connect/) is used to get authentication information for the user using the application and you would use these tokens to authenticate the application to its own endpoints.

To jump-start an application's implementation, you can use some model object definitions as well as an example class that handles the authentication and refresh operations.

## Models

### Token Protocol

This protocol defines the common property all of the types of tokens have as well as the additional protocols they should conform to.

```obj-c
@import Foundation;

@protocol Token <NSObject, NSSecureCoding, NSCopying>

@required

/**
 * The JWT to use for operations.
 */
@property (nonatomic, copy, readonly) NSString *token;

@end
```

### FHIR Token

This is the token used for authenticating to FHIR endpoints. The patient and encounter are optional depending on what scopes the application requests, but they are on this token for convenience.

#### Header

```obj-c
@import Foundation;
#import "Token.h"

NS_ASSUME_NONNULL_BEGIN

/**
 * @class FHIRToken
 * Class that models a FHIR token.
 */
@interface FHIRToken : NSObject <Token>

/**
 * The scope of the token.
 */
@property (nonatomic, copy, readonly) NSString *scope;

/**
 * The type of the token.
 */
@property (nonatomic, copy, readonly) NSString *type;

/**
 * The FHIR id of the patient in scope.
 */
@property (nonatomic, copy, nullable, readonly) NSString *patient;

/**
 * The FHIR id of the encounter in scope.
 */
@property (nonatomic, copy, nullable, readonly) NSString *encounter;

/**
 * Amount of time (in seconds since the token was issued) when the token will become expired.
 */
@property (nonatomic, assign, readonly) NSUInteger expiresIn;

- (instancetype)init NS_UNAVAILABLE;

+ (instancetype)new NS_UNAVAILABLE;

/**
 * Creates a new token with the given configuration.
 * @param accessToken The JWT to be used for authentication.
 * @param scope       The scope of the token.
 * @param type        The type of the token.
 * @param expiresIn   The amount of time (seconds) until the token is expired.
 * @param patient     The patient in the current FHIR scope.
 * @param encounter   The encounter in the current FHIR scope.
 * @return A new token to be used for authentication.
 */
- (instancetype)initWithAccessToken:(NSString *)accessToken scope:(NSString *)scope type:(NSString *)type expiresIn:(NSUInteger)expiresIn patient:(nullable NSString *)patient encounter:(nullable NSString *)encounter NS_DESIGNATED_INITIALIZER;

/**
 * Convenience initializer that creates a new token with the given configuration.
 * @param accessToken The JWT to be used for authentication.
 * @param scope       The scope of the token.
 * @param type        The type of the token.
 * @param expiresIn   The amount of time (seconds) until the token is expired.
 * @param patient     The patient in the current FHIR scope.
 * @param encounter   The encounter in the current FHIR scope.
 * @return A new token to be used for authentication.
 */
+ (instancetype)tokenWithAccessToken:(NSString *)accessToken scope:(NSString *)scope type:(NSString *)type expiresIn:(NSUInteger)expiresIn patient:(nullable NSString *)patient encounter:(nullable NSString *)encounter;

/**
 * Determines if one @c FHIRToken is equal to another.
 * @param token The token to compare to.
 * @return YES if the tokens are equal or NO otherwise.
 */
- (BOOL)isEqualToToken:(nullable FHIRToken *)token;

@end

NS_ASSUME_NONNULL_END
```

#### Implementation

```obj-c
#import "FHIRToken.h"

NS_ASSUME_NONNULL_BEGIN

@interface FHIRToken ()

@property (nonatomic, copy, readwrite) NSString *token;

@property (nonatomic, copy, readwrite) NSString *scope;

@property (nonatomic, copy, readwrite) NSString *type;

@property (nonatomic, assign, readwrite) NSUInteger expiresIn;

@property (nonatomic, copy, nullable, readwrite) NSString *patient;

@property (nonatomic, copy, nullable, readwrite) NSString *encounter;

@end

@implementation FHIRToken

#pragma mark - Object Life Cycle

- (instancetype)initWithAccessToken:(NSString *)accessToken scope:(NSString *)scope type:(NSString *)type expiresIn:(NSUInteger)expiresIn patient:(nullable NSString *)patient encounter:(nullable NSString *)encounter
{
    if ((self = [super init])) {
        _token = [accessToken copy];
        _scope = [scope copy];
        _type = [type copy];
        _expiresIn = expiresIn;
        _patient = [patient copy];
        _encounter = [encounter copy];
    }
    
    return self;
}

- (instancetype)__attribute__((noreturn)) init
{
    @throw [NSException exceptionWithName:NSInternalInconsistencyException reason:[NSString stringWithFormat:@"init is unavailable for class %@", NSStringFromClass([self class])] userInfo:nil];
}

+ (instancetype)__attribute__((noreturn)) new
{
    @throw [NSException exceptionWithName:NSInternalInconsistencyException reason:[NSString stringWithFormat:@"new is unavailable for class %@", NSStringFromClass(self)] userInfo:nil];
}

+ (instancetype)tokenWithAccessToken:(NSString *)accessToken scope:(NSString *)scope type:(NSString *)type expiresIn:(NSUInteger)expiresIn patient:(nullable NSString *)patient encounter:(nullable NSString *)encounter
{
    return [[self alloc] initWithAccessToken:accessToken scope:scope type:type expiresIn:expiresIn patient:patient encounter:encounter];
}

#pragma mark - Public API

- (BOOL)isEqualToToken:(nullable FHIRToken *)token
{
    if (token == self) {
        return YES;
    }
    
    if (!token || ![token isKindOfClass:[FHIRToken class]]) {
        return NO;
    }
    
    return [token.token isEqualToString:self.token]
    && [token.scope isEqualToString:self.scope]
    && [token.type isEqualToString:self.type]
    && token.expiresIn == self.expiresIn
    && ((!token.patient && !self.patient) || [token.patient isEqualToString:self.patient])
    && ((!token.encounter && !self.encounter) || [token.encounter isEqualToString:self.encounter]);
}

#pragma mark - NSCopying

- (instancetype)copyWithZone:(nullable NSZone *)zone
{
    return [[[self class] allocWithZone:zone] initWithAccessToken:self.token scope:self.token type:self.type expiresIn:self.expiresIn patient:self.patient encounter:self.encounter];
}

#pragma mark - NSCoding

- (nullable instancetype)initWithCoder:(NSCoder *)coder
{
    NSString *token = [coder decodeObjectOfClass:[NSString class] forKey:@"token"];
    NSString *scope = [coder decodeObjectOfClass:[NSString class] forKey:@"scope"];
    NSString *type = [coder decodeObjectOfClass:[NSString class] forKey:@"type"];
    NSNumber *expiresIn = [coder decodeObjectOfClass:[NSNumber class] forKey:@"expiresIn"];
    NSString *patient = [coder decodeObjectOfClass:[NSString class] forKey:@"patient"];
    NSString *encounter = [coder decodeObjectOfClass:[NSString class] forKey:@"encounter"];
    
    return [self initWithAccessToken:token scope:scope type:type expiresIn:expiresIn.unsignedIntegerValue patient:patient encounter:encounter];
}

- (void)encodeWithCoder:(NSCoder *)coder
{
    [coder encodeObject:self.token forKey:@"token"];
    [coder encodeObject:self.scope forKey:@"scope"];
    [coder encodeObject:self.type forKey:@"type"];
    [coder encodeObject:@(self.expiresIn) forKey:@"expiresIn"];
    
    if (self.patient) {
        [coder encodeObject:self.patient forKey:@"patient"];
    }
    
    if (self.encounter) {
        [coder encodeObject:self.encounter forKey:@"encounter"];
    }
}

#pragma mark - NSSecureCoding

+ (BOOL)supportsSecureCoding
{
    return YES;
}

#pragma mark - NSObject

- (BOOL)isEqual:(id)object
{
    if (object == self) {
        return YES;
    }
    
    if (!object || ![object isKindOfClass:[FHIRToken class]]) {
        return NO;
    }
    
    return [self isEqualToToken:object];
}

- (NSUInteger)hash
{
    return self.token.hash ^ self.scope.hash ^ self.type.hash ^ self.expiresIn ^ self.patient.hash ^ self.encounter.hash;
}

- (NSString *)debugDescription
{
    return [NSString stringWithFormat:@"<%@: %p> {\n\tToken: %@\n\tScope: %@\n\tType: %@\n\tExpires In: %lu\n\tPatient: %@\n\tEncounter: %@\n}",
            NSStringFromClass([self class]),
            (void *)self,
            self.token,
            self.scope,
            self.type,
            self.expiresIn,
            self.patient,
            self.encounter];
}

@end

NS_ASSUME_NONNULL_END
```

### FHIR Refresh Token

This token is used for obtaining new FHIR tokens as those only last for a limited amount of time. If the application only needs to use a FHIR token within its lifetime, it is possible to not have one as it must be requested in the scopes.

#### Header

```obj-c
import Foundation;
#import "Token.h"

NS_ASSUME_NONNULL_BEGIN

/**
 * @class FHIRRefreshToken
 * Class that models a FHIR refresh token.
 */
@interface FHIRRefreshToken : NSObject <Token>

- (instancetype)init NS_UNAVAILABLE;

+ (instancetype)new NS_UNAVAILABLE;

/**
 * Creates a new refresh token with the given configuration.
 * @param token The JWT to be used for refresh operations.
 * @return A new token to be used for refresh operations.
 */
- (instancetype)initWithToken:(NSString *)token;

/**
 * Convenience initializer that creates a new refresh token with the given configuration.
 * @param token The JWT to be used for refresh operations.
 * @return A new token to be used for refresh operations.
 */
+ (instancetype)tokenWithToken:(NSString *)token;

/**
 * Determines if one @c FHIRRefreshToken is equal to another.
 * @param token The token to compare to.
 * @return YES if the tokens are equal or NO otherwise.
 */
- (BOOL)isEqualToToken:(nullable FHIRRefreshToken *)token;

@end

NS_ASSUME_NONNULL_END
```

#### Implementation

```obj-c
#import "FHIRRefreshToken.h"

NS_ASSUME_NONNULL_BEGIN

@interface FHIRRefreshToken ()

@property (nonatomic, copy, readwrite) NSString *token;

@end

@implementation FHIRRefreshToken

#pragma mark - Object Life Cycle

- (instancetype)initWithToken:(NSString *)token
{
    if ((self = [super init])) {
        _token = [token copy];
    }
    
    return self;
}

- (instancetype)__attribute__((noreturn)) init
{
    @throw [NSException exceptionWithName:NSInternalInconsistencyException reason:[NSString stringWithFormat:@"init is unavailable for class %@", NSStringFromClass([self class])] userInfo:nil];
}

+ (instancetype)__attribute__((noreturn)) new
{
    @throw [NSException exceptionWithName:NSInternalInconsistencyException reason:[NSString stringWithFormat:@"new is unavailable for class %@", NSStringFromClass(self)] userInfo:nil];
}

+ (instancetype)tokenWithToken:(NSString *)token
{
    return [[self alloc] initWithToken:token];
}

#pragma mark - Public API

- (BOOL)isEqualToToken:(nullable FHIRRefreshToken *)token
{
    if (token == self) {
        return YES;
    }
    
    if (!token || ![token isKindOfClass:[FHIRRefreshToken class]]) {
        return NO;
    }
    
    return [token.token isEqualToString:self.token];
}

#pragma mark - NSCopying

- (instancetype)copyWithZone:(nullable NSZone *)zone
{
    return [[[self class] allocWithZone:zone] initWithToken:self.token];
}

#pragma mark - NSCoding

- (nullable instancetype)initWithCoder:(NSCoder *)coder
{
    NSString *token = [coder decodeObjectOfClass:[NSString class] forKey:@"token"];
    
    return [self initWithToken:token];
}

- (void)encodeWithCoder:(NSCoder *)coder
{
    [coder encodeObject:self.token forKey:@"token"];
}

#pragma mark - NSSecureCoding

+ (BOOL)supportsSecureCoding
{
    return YES;
}

#pragma mark - NSObject

- (BOOL)isEqual:(id)object
{
    if (object == self) {
        return YES;
    }
    
    if (!object || ![object isKindOfClass:[FHIRRefreshToken class]]) {
        return NO;
    }
    
    return [self isEqualToToken:object];
}

- (NSUInteger)hash
{
    return self.token.hash;
}

- (NSString *)debugDescription
{
    return [NSString stringWithFormat:@"<%@: %p> {\n\tToken: %@\n}",
            NSStringFromClass([self class]),
            (void *)self,
            self.token];
}

@end

NS_ASSUME_NONNULL_END
```

### FHIR OpenID Connect Token

This token is used for authenticating the application to its own endpoints and contains user information.

#### Header

```obj-c
@import Foundation;
#import "Token.h"

NS_ASSUME_NONNULL_BEGIN

/**
 * @class FHIRIDToken
 * Class that models a OpenID Connect token.
 */
@interface FHIRIDToken : NSObject <Token>

- (instancetype)init NS_UNAVAILABLE;

+ (instancetype)new NS_UNAVAILABLE;

/**
 * Creates a new ID token with the given configuration.
 * @param token The JWT to be used for ID operations.
 * @return A new token to be used for ID operations.
 */
- (instancetype)initWithToken:(NSString *)token;

/**
 * Convenience initializer that creates a new ID token with the given configuration.
 * @param token The JWT to be used for ID operations.
 * @return A new token to be used for ID operations.
 */
+ (instancetype)tokenWithToken:(NSString *)token;

/**
 * Determines if one @c FHIRIDToken is equal to another.
 * @param token The token to compare to.
 * @return YES if the tokens are equal or NO otherwise.
 */
- (BOOL)isEqualToToken:(nullable FHIRIDToken *)token;

@end

NS_ASSUME_NONNULL_END
```

#### Implementation

```obj-c
#import "FHIRIDToken.h"

NS_ASSUME_NONNULL_BEGIN

@interface FHIRIDToken ()

@property (nonatomic, copy, readwrite) NSString *token;

@end

@implementation FHIRIDToken

#pragma mark - Object Life Cycle

- (instancetype)initWithToken:(NSString *)token
{
    if ((self = [super init])) {
        _token = [token copy];
    }
    
    return self;
}

- (instancetype)__attribute__((noreturn)) init
{
    @throw [NSException exceptionWithName:NSInternalInconsistencyException reason:[NSString stringWithFormat:@"init is unavailable for class %@", NSStringFromClass([self class])] userInfo:nil];
}

+ (instancetype)__attribute__((noreturn)) new
{
    @throw [NSException exceptionWithName:NSInternalInconsistencyException reason:[NSString stringWithFormat:@"new is unavailable for class %@", NSStringFromClass(self)] userInfo:nil];
}

+ (instancetype)tokenWithToken:(NSString *)token
{
    return [[self alloc] initWithToken:token];
}

#pragma mark - Public API

- (BOOL)isEqualToToken:(nullable FHIRIDToken *)token
{
    if (token == self) {
        return YES;
    }
    
    if (!token || ![token isKindOfClass:[FHIRIDToken class]]) {
        return NO;
    }
    
    return [token.token isEqualToString:self.token];
}

#pragma mark - NSCopying

- (instancetype)copyWithZone:(nullable NSZone *)zone
{
    return [[[self class] allocWithZone:zone] initWithToken:self.token];
}

#pragma mark - NSCoding

- (nullable instancetype)initWithCoder:(NSCoder *)coder
{
    NSString *token = [coder decodeObjectOfClass:[NSString class] forKey:@"token"];
    
    return [self initWithToken:token];
}

- (void)encodeWithCoder:(NSCoder *)coder
{
    [coder encodeObject:self.token forKey:@"token"];
}

#pragma mark - NSSecureCoding

+ (BOOL)supportsSecureCoding
{
    return YES;
}

#pragma mark - NSObject

- (BOOL)isEqual:(id)object
{
    if (object == self) {
        return YES;
    }
    
    if (!object || ![object isKindOfClass:[FHIRIDToken class]]) {
        return NO;
    }
    
    return [self isEqualToToken:object];
}

- (NSUInteger)hash
{
    return self.token.hash;
}

- (NSString *)debugDescription
{
    return [NSString stringWithFormat:@"<%@: %p> {\n\tToken: %@\n}",
            NSStringFromClass([self class]),
            (void *)self,
            self.token];
}

@end

NS_ASSUME_NONNULL_END
```

## Authentication

Authentication is handled via [ASWebAuthenticationSession](https://developer.apple.com/documentation/authenticationservices/aswebauthenticationsession?language=objc). This is due to Safari supporting all modern authentication standards, but it does require the user to acknowledge a permission prompt. If you do not need the state of the authentication after the initial login, you can set [`prefersEphemeralWebBrowserSession`](https://developer.apple.com/documentation/authenticationservices/aswebauthenticationsession/3237231-prefersephemeralwebbrowsersessio?language=objc) to bypass the prompt.

Applications are required to register a callback URL for FHIR, so the URL scheme but be configured in the application's plist in order for the app to receive it. An example scheme would be `com.example.cb` and then the authenticator appends `://fhir` to make it a valid URL (the registered scheme would be `com.example.cb://fhir`). 

The authentication process is a series of steps:

1. Call the `metadata` route on the FHIR base URL to determine the additional URLs to use for authorization and token operations.
2. Construct and make an authorization grant request.
3. Assuming the grant request was successful, exchange the grant for a FHIR token (and optionally a refresh token).

Since the grant request depends on scopes, a simple set of scopes can be used for testing:

* user/Patient.read
* patient/Patient.read
* openid
* fhirUser
* launch/patient
* launch/encounter
* online_access

The one thing that can be annoying to manage during authorization is that errors are typically URLs and you cannot change the URL of an instance of `ASWebAuthenticationSession` once it is launched. Therefore, you will need to present the error in a web view or launch out to Safari itself. In the example below, the `FHIRAuthenticator` assumes that an application provides it with an instance of `WKWebView`. Additionally, some well-defined error states are given native definitions and some logging is included. This is meant to be adjusted and customized based on the application's logic.

Note: Since the callback URL does not need to get passed to the `ASWebAuthenticationSession` API, you should also implement the [`application:openURL:options:`](https://developer.apple.com/documentation/uikit/uiapplicationdelegate/1623112-application?language=objc) method in the application's delegate as a fallback.

### Header

```obj-c
@import Foundation;
@class WKWebView;
@class FHIRToken;
@class FHIRRefreshToken;
@class FHIRIDToken;

NS_ASSUME_NONNULL_BEGIN

/**
 * Block to be executed once the endpoint discovery process completes.
 * @param success Boolean indicator of whether or not the process completed successfully.
 * @param error   The error associated with the operation. Nil if the operation succeeded.
 */
typedef void (^FHIRAuthenticatorDiscoveryCompletionHandler)(BOOL success, NSError *_Nullable error) NS_SWIFT_NAME(FHIRAuthenticator.DiscoveryCompletionHandler);

/**
 * Block to be executed once the authorization grant process completes.
 * @param success            Boolean indicator of whether or not the process completed successfully.
 * @param authorizationGrant The authorization grant given to the application. Nil if the operation failed.
 * @param error              The error associated with the operation. Nil if the operation succeeded.
 */
typedef void (^FHIRAuthenticatorAuthorizationGrantCompletionHandler)(BOOL success, NSString *_Nullable authorizationGrant, NSError *_Nullable error) NS_SWIFT_NAME(FHIRAuthenticator.AuthorizationGrantCompletionHandler);

/**
 * Block to be executed once the grant exchange process completes.
 * @param success      Boolean indicator of whether or not the process completed successfully.
 * @param token        The token given to the application. Nil if the operation failed.
 * @param refreshToken The refreshToken used to obtain new tokens. Nil if not requested.
 * @param idToken      The OpenID Connect token associated with the user.
 * @param error        The error associated with the operation. Nil if the operation succeeded.
 */
typedef void (^FHIRAuthenticatorAuthorizationGrantExchangeCompletionHandler)(BOOL success, FHIRToken *_Nullable token, FHIRRefreshToken *_Nullable refreshToken, FHIRIDToken *_Nullable idToken, NSError *_Nullable error) NS_SWIFT_NAME(FHIRAuthenticator.AuthorizationGrantExchangeCompletionHandler);

/**
 * Block to be executed once the refresh process completes.
 * @param success    Boolean indicator of whether or not the process completed successfully.
 * @param token      The new token given to the application. Nil if the operation failed.
 * @param idToken    The OpenID Connect token associated with the user.
 * @param error      The error associated with the operation. Nil if the operation succeeded.
 */
typedef void (^FHIRAuthenticatorTokenRefreshCompletionHandler)(BOOL success, FHIRToken *_Nullable token, FHIRIDToken *_Nullable idToken, NSError *_Nullable error) NS_SWIFT_NAME(FHIRAuthenticator.TokenRefreshCompletionHandler);

/**
 * The error domain for @c FHIRAuthenticator errors.
 */
static NSErrorDomain const FHIRAuthenticatorErrorDomain;

/**
 * @enum FHIRAuthenticatorErrorCode
 * The various definitions for errors that can occur during FHIR transactions.
 */
typedef NS_ERROR_ENUM(FHIRAuthenticatorErrorDomain, FHIRAuthenticatorErrorCode) {
    /// An unknown error occurred.
    FHIRAuthenticatorErrorUnknown = -1,
    /// A network transaction was unable to be created.
    FHIRAuthenticatorErrorRequestCreationFailed = 0,
    /// A network transaction failed.
    FHIRAuthenticatorErrorRequestFailed,
    /// A network transaction received an invalid response.
    FHIRAuthenticatorErrorInvalidResponse,
    /// A network transaction received an empty response body.
    FHIRAuthenticatorErrorEmptyResponse,
    /// A network transaction received a response that it could not parse (e.g. malformed data).
    FHIRAuthenticatorErrorUnparseableResponse,
    /// A network transaction received a response that it was not expecting (e.g. wrong data type).
    FHIRAuthenticatorErrorIncorrectResponse,
    /// A network transaction received a response that was missing required data.
    FHIRAuthenticatorErrorMissingRequiredResponseFields,
    /// The authorization grant workflow failed.
    FHIRAuthenticatorErrorAuthorizationGrantWorkflowFailed,
    /// The authorization grant workflow received a response with the wrong state parameter.
    FHIRAuthenticatorErrorAuthorizationGrantStateMismatch,
    /// The authorization grant workflow received a callback that was invalid.
    FHIRAuthenticatorErrorAuthorizationGrantInvalidCallback,
    /// The authorization grant exchange workflow failed.
    FHIRAuthenticatorErrorAuthorizationExchangeWorkflowFailed,
    /// The refresh token workflow failed
    FHIRAuthenticatorErrorTokenRefreshWorkflowFailed
};

/**
 * @class FHIRAuthenticator
 * Class that orchestrates FHIR authentication operations. To obtain a FHIR access token for authentication you must:
 * 1. Discover the FHIR authorization, token, and introspection URLs for a given FHIR instance.
 * 2. Request an Authorization Grant using the discovered authorization endpoint (extremely short lived).
 * 3. Exchange the authorization grant using the discovered token endpoint.
 * Once you have your token, you can use it in an authorization header with Bearer authentication.
 * If you need to continue using tokens after the original one expires, you will need to request and leverage refresh tokens.
 */
@interface FHIRAuthenticator : NSObject

/**
 * The base FHIR EHR URL in use.
 */
@property (nonatomic, copy, readonly) NSURL *FHIRBaseURL;

/**
 * The callback scheme used by the application to handle FHIR callbacks.
 */
@property (nonatomic, copy, readonly) NSString *callbackScheme;

/**
 * The application's identifier registered for the FHIR instance.
 */
@property (nonatomic, copy, readonly) NSString *clientIdentifier;

/**
 * The authorization endpoint for the FHIR instance.
 */
@property (nonatomic, copy, nullable, readonly) NSURL *authorizationURL;

/**
 * The introspection endpoint for the FHIR instance.
 */
@property (nonatomic, copy, nullable, readonly) NSURL *introspectionURL;

/**
 * The token endpoint for the FHIR instance.
 */
@property (nonatomic, copy, nullable, readonly) NSURL *tokenURL;

- (instancetype)init NS_UNAVAILABLE;

+ (instancetype)new NS_UNAVAILABLE;

/**
 * Creates a new authenticator with the given configuration.
 * @param baseURL          The FHIR EHR URL.
 * @param callbackScheme   The callback scheme to be used to handle FHIR callbacks.
 * @param clientIdentifier The identifier of the application registered with the FHIR instance.
 * @return A new authenticator to be used for a specific FHIR instance.
 */
- (instancetype)initWithFHIRBaseURL:(NSURL *)baseURL callbackScheme:(NSString *)callbackScheme clientIdentifier:(NSString *)clientIdentifier NS_DESIGNATED_INITIALIZER;

/**
 * Convenience initializer that creates a new authenticator with the given configuration.
 * @param baseURL          The FHIR EHR URL.
 * @param callbackScheme   The callback scheme to be used to handle FHIR callbacks.
 * @param clientIdentifier The identifier of the application registered with the FHIR instance.
 * @return A new authenticator to be used for a specific FHIR instance.
 */
+ (instancetype)authenticatorWithFHIRBaseURL:(NSURL *)baseURL callbackScheme:(NSString *)callbackScheme clientIdentifier:(NSString *)clientIdentifier;

/**
 * Discovers and then configures the authenticator with the discovered endpoint values at a given FHIR instance.
 * @param completionHandler Block to execute once the operation completes.
 */
- (void)discoverEndpointsWithCompletionHandler:(FHIRAuthenticatorDiscoveryCompletionHandler)completionHandler;

/**
 * Obtains an authorization grant to be exchanged for a FHIR token.
 * @param webView           The web view to perform login in as well as display errors.
 * @param scopes            The space separated scopes to be requested.
 * @param launch            The launch context of the application.
 * @param completionHandler Block to execute once the operation completes.
 */
- (void)obtainAuthorizationGrantInWebView:(WKWebView *)webView withScopes:(NSString *)scopes launch:(nullable NSString *)launch completionHandler:(FHIRAuthenticatorAuthorizationGrantCompletionHandler)completionHandler;

/**
 * Obtains a FHIR token to be used for authentication.
 * @param webView            The web view to view MPages Reach in as well as display errors.
 * @param authorizationGrant The authorization grant to exchange for a FHIR token.
 * @param completionHandler  Block to execute once the operation completes.
 */
- (void)obtainFHIRTokenInWebView:(WKWebView *)webView withAuthorizationGrant:(NSString *)authorizationGrant completionHandler:(FHIRAuthenticatorAuthorizationGrantExchangeCompletionHandler)completionHandler;

/**
 * Obtains a new FHIR token leveraging the application's refresh token.
 * @param refreshToken      The refresh token issued to the application during the grant exchange.
 * @param completionHandler Block to execute once the operation completes.
 */
- (void)obtainFHIRTokenWithRefreshToken:(FHIRRefreshToken *)refreshToken completionHandler:(FHIRAuthenticatorTokenRefreshCompletionHandler)completionHandler;

@end

NS_ASSUME_NONNULL_END
```

### Implementation

```obj-c
#import "FHIRAuthenticator.h"
@import os.log;
@import WebKit;
@import AuthenticationServices;
#import "FHIRToken.h"
#import "FHIRRefreshToken.h"
#import "FHIRIDToken.h"

static NSErrorDomain const FHIRAuthenticatorErrorDomain = @"com.example.FHIRAuthenticator.error";

@interface FHIRAuthenticator ()

@property (nonatomic, copy, readwrite) NSURL *FHIRBaseURL;

@property (nonatomic, copy, readwrite) NSString *callbackScheme;

@property (nonatomic, copy, readwrite) NSString *clientIdentifier;

@property (nonatomic, copy, nullable, readwrite) NSURL *authorizationURL;

@property (nonatomic, copy, nullable, readwrite) NSURL *introspectionURL;

@property (nonatomic, copy, nullable, readwrite) NSURL *tokenURL;

/**
 * The current state value used to protect requests.
 */
@property (atomic, copy, nullable) NSString *currentState;

/**
 * The web view in use for operations.
 */
@property (nonatomic, weak, nullable) WKWebView *webView;

/**
 * The completion handler to call when the authorization grant process completes.
 */
@property (nonatomic, copy, nullable) FHIRAuthenticatorAuthorizationGrantCompletionHandler authGrantCompletionHandler;

/**
 * The authentication session in use for login.
 */
@property (nonatomic, strong, nullable) ASWebAuthenticationSession *currentSession;

@end

@implementation FHIRAuthenticator

#pragma mark - Object Life Cycle

- (instancetype)initWithFHIRBaseURL:(NSURL *)baseURL callbackScheme:(NSString *)callbackScheme clientIdentifier:(NSString *)clientIdentifier
{
    if ((self = [super init])) {
        _FHIRBaseURL = [baseURL copy];
        _callbackScheme = [callbackScheme copy];
        _clientIdentifier = [clientIdentifier copy];
    }
    
    return self;
}

- (instancetype)__attribute__((noreturn)) init
{
    @throw [NSException exceptionWithName:NSInternalInconsistencyException reason:[NSString stringWithFormat:@"init is unavailable for class %@", NSStringFromClass([self class])] userInfo:nil];
}

+ (instancetype)__attribute__((noreturn)) new
{
    @throw [NSException exceptionWithName:NSInternalInconsistencyException reason:[NSString stringWithFormat:@"new is unavailable for class %@", NSStringFromClass(self)] userInfo:nil];
}

+ (instancetype)authenticatorWithFHIRBaseURL:(NSURL *)baseURL callbackScheme:(NSString *)callbackScheme clientIdentifier:(NSString *)clientIdentifier
{
    return [[self alloc] initWithFHIRBaseURL:baseURL callbackScheme:callbackScheme clientIdentifier:clientIdentifier];
}

#pragma mark - Public API

- (void)discoverEndpointsWithCompletionHandler:(FHIRAuthenticatorDiscoveryCompletionHandler)completionHandler
{
    NSURL *url = [self.FHIRBaseURL URLByAppendingPathComponent:@"metadata" isDirectory:NO];
    NSMutableURLRequest *request = [NSMutableURLRequest requestWithURL:url];
    request.HTTPMethod = @"GET";
    [request setValue:@"application/fhir+json" forHTTPHeaderField:@"Accept"];
    
    __weak __auto_type weakSelf = self;
    NSURLSessionDataTask *task = [NSURLSession.sharedSession dataTaskWithRequest:request completionHandler:^(NSData *_Nullable data, NSURLResponse *_Nullable response, NSError *_Nullable error) {
        dispatch_async(dispatch_get_main_queue(), ^{
            [weakSelf handleEndpointDiscoveryResponse:response withData:data error:error completionHandler:completionHandler];
        });
    }];
    
    if (task) {
        os_log_debug(OS_LOG_DEFAULT, "Sending FHIR discovery request to %{public}@", task.currentRequest.URL.absoluteString);
        [task resume];
    }
    else {
        completionHandler(NO, [NSError errorWithDomain:FHIRAuthenticatorErrorDomain code:FHIRAuthenticatorErrorRequestCreationFailed userInfo:nil]);
    }
}

- (void)obtainAuthorizationGrantInWebView:(WKWebView *)webView withScopes:(NSString *)scopes launch:(nullable NSString *)launch completionHandler:(FHIRAuthenticatorAuthorizationGrantCompletionHandler)completionHandler
{
    NSURLComponents *components = [NSURLComponents componentsWithURL:self.authorizationURL resolvingAgainstBaseURL:NO];
    self.currentState = NSUUID.UUID.UUIDString;
    self.webView = webView;
    self.authGrantCompletionHandler = completionHandler;
    
    NSArray<NSURLQueryItem *> *queryParams = @[
        [NSURLQueryItem queryItemWithName:@"client_id" value:self.clientIdentifier],
        [NSURLQueryItem queryItemWithName:@"response_type" value:@"code"],
        [NSURLQueryItem queryItemWithName:@"redirect_uri" value:[NSString stringWithFormat:@"%@://fhir", self.callbackScheme]],
        [NSURLQueryItem queryItemWithName:@"scope" value:scopes],
        [NSURLQueryItem queryItemWithName:@"launch" value:launch ?: @""],
        [NSURLQueryItem queryItemWithName:@"aud" value:self.FHIRBaseURL.absoluteString],
        [NSURLQueryItem queryItemWithName:@"state" value:self.currentState]
    ];
    [components setQueryItems:queryParams];
    NSURL *requestURL = components.URL;
    
    os_log_debug(OS_LOG_DEFAULT, "Sending authorization grant request to %{public}@", requestURL.absoluteString);
    __weak __auto_type weakSelf = self;
    ASWebAuthenticationSession *session = [[ASWebAuthenticationSession alloc] initWithURL:requestURL callbackURLScheme:self.callbackScheme completionHandler:^(NSURL *_Nullable callbackURL, NSError *_Nullable error) {
        if (!error) {
            [weakSelf validateAuthorizationGrantCallback:callbackURL];
        }
        
        weakSelf.currentSession = nil;
    }];
    
    session.presentationContextProvider = (id<ASWebAuthenticationPresentationContextProviding>)UIApplication.sharedApplication.delegate; // This needs to be whatever is managing the window (UIApplicationDelegate or UISceneDelegate)
    
    self.currentSession = session;
    [session start];
}

- (void)obtainFHIRTokenInWebView:(WKWebView *)webView withAuthorizationGrant:(NSString *)authorizationGrant completionHandler:(FHIRAuthenticatorAuthorizationGrantExchangeCompletionHandler)completionHandler
{
    self.webView = webView;
    self.currentState = NSUUID.UUID.UUIDString;
    NSString *body = [NSString stringWithFormat:@"client_id=%@&grant_type=authorization_code&redirect_uri=%@://fhir&code=%@&state=%@", self.clientIdentifier, self.callbackScheme, authorizationGrant, self.currentState];
    NSData *bodyPayload = [body dataUsingEncoding:NSASCIIStringEncoding];
    NSMutableURLRequest *request = [NSMutableURLRequest requestWithURL:self.tokenURL];
    request.HTTPMethod = @"POST";
    [request setValue:@"application/json" forHTTPHeaderField:@"Accept"];
    [request setValue:@"application/x-www-form-urlencoded" forHTTPHeaderField:@"Content-Type"];
    request.HTTPBody = bodyPayload;
    
    __weak __auto_type weakSelf = self;
    NSURLSessionDataTask *task = [NSURLSession.sharedSession dataTaskWithRequest:request completionHandler:^(NSData *_Nullable data, NSURLResponse *_Nullable response, NSError *_Nullable error) {
        dispatch_async(dispatch_get_main_queue(), ^{
            [weakSelf handleGrantExchangeResponse:response withData:data error:error completionHandler:completionHandler];
        });
    }];
    
    if (task) {
        os_log_debug(OS_LOG_DEFAULT, "Sending authorization grant exchange request to %{public}@ with payload %{public}@", task.currentRequest.URL.absoluteString, body);
        [task resume];
    }
    else {
        completionHandler(NO, nil, nil, nil, [NSError errorWithDomain:FHIRAuthenticatorErrorDomain code:FHIRAuthenticatorErrorRequestCreationFailed userInfo:nil]);
    }
}

- (void)obtainFHIRTokenWithRefreshToken:(FHIRRefreshToken *)refreshToken completionHandler:(FHIRAuthenticatorTokenRefreshCompletionHandler)completionHandler
{
    NSString *body = [NSString stringWithFormat:@"grant_type=refresh_token&refresh_token=%@", refreshToken.token];
    NSData *bodyPayload = [body dataUsingEncoding:NSASCIIStringEncoding];
    NSMutableURLRequest *request = [NSMutableURLRequest requestWithURL:self.tokenURL];
    request.HTTPMethod = @"POST";
    [request setValue:@"application/json" forHTTPHeaderField:@"Accept"];
    [request setValue:@"application/x-www-form-urlencoded" forHTTPHeaderField:@"Content-Type"];
    request.HTTPBody = bodyPayload;
    
    __weak __auto_type weakSelf = self;
    NSURLSessionDataTask *task = [NSURLSession.sharedSession dataTaskWithRequest:request completionHandler:^(NSData *_Nullable data, NSURLResponse *_Nullable response, NSError *_Nullable error) {
        dispatch_async(dispatch_get_main_queue(), ^{
            [weakSelf handleRefreshResponse:response withData:data error:error completionHandler:completionHandler];
        });
    }];
    
    if (task) {
        os_log_debug(OS_LOG_DEFAULT, "Sending token refresh request to %{public}@ with payload %{public}@", task.currentRequest.URL.absoluteString, body);
        [task resume];
    }
    else {
        completionHandler(NO, nil, nil, [NSError errorWithDomain:FHIRAuthenticatorErrorDomain code:FHIRAuthenticatorErrorRequestCreationFailed userInfo:nil]);
    }
}

#pragma mark - Private API

/**
 * Handles the results of discovering the FHIR instance endpoints.
 * @param response          The response from the server.
 * @param data              The response data from the server.
 * @param error             The error associated with the operation.
 * @param completionHandler Block to execute once processing is complete.
 */
- (void)handleEndpointDiscoveryResponse:(NSURLResponse *)response withData:(nullable NSData *)data error:(nullable NSError *)error completionHandler:(FHIRAuthenticatorDiscoveryCompletionHandler)completionHandler
{
    if (![response isKindOfClass:[NSHTTPURLResponse class]]) {
        completionHandler(NO, [NSError errorWithDomain:FHIRAuthenticatorErrorDomain code:FHIRAuthenticatorErrorInvalidResponse userInfo:nil]);
        return;
    }
    
    if (error) {
        os_log_error(OS_LOG_DEFAULT, "Encountered an error during FHIR endpoint discovery: %{public}@", error.debugDescription);
        completionHandler(NO, [NSError errorWithDomain:FHIRAuthenticatorErrorDomain code:FHIRAuthenticatorErrorRequestFailed userInfo:@{NSUnderlyingErrorKey : error}]);
        return;
    }
    
    NSHTTPURLResponse *httpResponse = (NSHTTPURLResponse *)response;
    NSInteger httpStatus = httpResponse.statusCode;
    if (httpStatus != 200) {
        os_log_error(OS_LOG_DEFAULT, "FHIR endpoint discovery did not complete successfully: %ld", httpStatus);
        completionHandler(NO, [NSError errorWithDomain:FHIRAuthenticatorErrorDomain code:FHIRAuthenticatorErrorRequestFailed userInfo:@{@"status" : @(httpStatus)}]);
        return;
    }
    
    if (!data) {
        completionHandler(NO, [NSError errorWithDomain:FHIRAuthenticatorErrorDomain code:FHIRAuthenticatorErrorEmptyResponse userInfo:nil]);
        return;
    }
    
    NSError *jsonError = nil;
    id payload = [NSJSONSerialization JSONObjectWithData:data options:kNilOptions error:&jsonError];
    
    if (!payload || jsonError) {
        os_log_error(OS_LOG_DEFAULT, "Encountered a conversion error during FHIR endpoint discovery: %{public}@", jsonError.debugDescription);
        completionHandler(NO, [NSError errorWithDomain:FHIRAuthenticatorErrorDomain code:FHIRAuthenticatorErrorUnparseableResponse userInfo:(jsonError ? @{NSUnderlyingErrorKey : jsonError} : nil)]);
        return;
    }
    
    if (![payload isKindOfClass:[NSDictionary class]]) {
        completionHandler(NO, [NSError errorWithDomain:FHIRAuthenticatorErrorDomain code:FHIRAuthenticatorErrorIncorrectResponse userInfo:nil]);
        return;
    }
    
    NSDictionary *discoveryPayload = (NSDictionary *)payload;
    NSArray<NSDictionary *> *securityExtension = ((NSDictionary *)((NSArray *)(NSDictionary *)(((NSArray *)discoveryPayload[@"rest"]).firstObject)[@"security"][@"extension"]).firstObject)[@"extension"];
    NSURL *authorizationURL = nil;
    NSURL *introspectionURL = nil;
    NSURL *tokenURL = nil;
    
    for (NSDictionary *dict in securityExtension) {
        NSString *uri = dict[@"url"];
        if ([uri isEqualToString:@"token"]) {
            tokenURL = [NSURL URLWithString:dict[@"valueUri"]];
        }
        else if ([uri isEqualToString:@"authorize"]) {
            authorizationURL = [NSURL URLWithString:dict[@"valueUri"]];
        }
        else if ([uri isEqualToString:@"introspect"]) {
            introspectionURL = [NSURL URLWithString:dict[@"valueUri"]];
        }
    }
    
    if (!tokenURL || !authorizationURL || !introspectionURL) {
        completionHandler(NO, [NSError errorWithDomain:FHIRAuthenticatorErrorDomain code:FHIRAuthenticatorErrorMissingRequiredResponseFields userInfo:nil]);
        return;
    }
    
    self.authorizationURL = authorizationURL;
    self.introspectionURL = introspectionURL;
    self.tokenURL = tokenURL;
    
    os_log_debug(OS_LOG_DEFAULT, "Discovered authorization endpoint: %{public}@", authorizationURL);
    os_log_debug(OS_LOG_DEFAULT, "Discovered introspection endpoint: %{public}@", introspectionURL);
    os_log_debug(OS_LOG_DEFAULT, "Discovered token endpoint: %{public}@", tokenURL);
    completionHandler(YES, nil);
}

/**
 * Handles the results of exchanging the authorization grant for a FHIR token.
 * @param response          The response from the server.
 * @param data              The response data from the server.
 * @param error             The error associated with the operation.
 * @param completionHandler Block to execute once processing is complete.
 */
- (void)handleGrantExchangeResponse:(NSURLResponse *)response withData:(nullable NSData *)data error:(nullable NSError *)error completionHandler:(FHIRAuthenticatorAuthorizationGrantExchangeCompletionHandler)completionHandler
{
    if (![response isKindOfClass:[NSHTTPURLResponse class]]) {
        completionHandler(NO, nil, nil, nil, [NSError errorWithDomain:FHIRAuthenticatorErrorDomain code:FHIRAuthenticatorErrorInvalidResponse userInfo:nil]);
        return;
    }
    
    if (error) {
        os_log_error(OS_LOG_DEFAULT, "Encountered an error during authorization grant exchange: %{public}@", error.debugDescription);
        completionHandler(NO, nil, nil, nil, [NSError errorWithDomain:FHIRAuthenticatorErrorDomain code:FHIRAuthenticatorErrorRequestFailed userInfo:@{NSUnderlyingErrorKey : error}]);
        return;
    }
    
    NSHTTPURLResponse *httpResponse = (NSHTTPURLResponse *)response;
    NSInteger httpStatus = httpResponse.statusCode;
    if (httpStatus != 200) {
        os_log_error(OS_LOG_DEFAULT, "Authorization grant exchange did not complete successfully: %ld", httpStatus);
        completionHandler(NO, nil, nil, nil, [NSError errorWithDomain:FHIRAuthenticatorErrorDomain code:FHIRAuthenticatorErrorRequestFailed userInfo:@{@"status" : @(httpStatus)}]);
        return;
    }
    
    if (!data) {
        completionHandler(NO, nil, nil, nil, [NSError errorWithDomain:FHIRAuthenticatorErrorDomain code:FHIRAuthenticatorErrorEmptyResponse userInfo:nil]);
        return;
    }
    
    NSError *jsonError = nil;
    id payload = [NSJSONSerialization JSONObjectWithData:data options:kNilOptions error:&jsonError];
    
    if (!payload || jsonError) {
        os_log_error(OS_LOG_DEFAULT, "Encountered a conversion error during authorization grant exchange: %{public}@", jsonError.debugDescription);
        completionHandler(NO, nil, nil, nil, [NSError errorWithDomain:FHIRAuthenticatorErrorDomain code:FHIRAuthenticatorErrorUnparseableResponse userInfo:(jsonError ? @{NSUnderlyingErrorKey : jsonError} : nil)]);
        return;
    }
    
    if (![payload isKindOfClass:[NSDictionary class]]) {
        completionHandler(NO, nil, nil, nil, [NSError errorWithDomain:FHIRAuthenticatorErrorDomain code:FHIRAuthenticatorErrorIncorrectResponse userInfo:nil]);
        return;
    }
    
    NSDictionary *exchangePayload = (NSDictionary *)payload;
    NSString *exchangeError = exchangePayload[@"error_uri"];
    NSString *accessToken = exchangePayload[@"access_token"];
    NSString *scope = exchangePayload[@"scope"];
    NSString *tokenType = exchangePayload[@"token_type"];
    NSNumber *expiresIn = exchangePayload[@"expires_in"];
    NSString *patient = exchangePayload[@"patient"];
    NSString *encounter = exchangePayload[@"encounter"];
    NSString *refreshToken = exchangePayload[@"refresh_token"];
    NSString *openIDToken = exchangePayload[@"id_token"];
    
    if (exchangeError) {
        os_log_error(OS_LOG_DEFAULT, "Encountered error for authorization grant exchange: %{public}@", error);
        [self.webView loadRequest:[NSURLRequest requestWithURL:[NSURL URLWithString:exchangeError] cachePolicy:NSURLRequestReloadIgnoringLocalCacheData timeoutInterval:30]];
        completionHandler(NO, nil, nil, nil, [NSError errorWithDomain:FHIRAuthenticatorErrorDomain code:FHIRAuthenticatorErrorAuthorizationExchangeWorkflowFailed userInfo:@{@"error_uri" : error}]);
        return;
    }
    else if (!accessToken || !scope || !tokenType || expiresIn == nil) {
        completionHandler(NO, nil, nil, nil, [NSError errorWithDomain:FHIRAuthenticatorErrorDomain code:FHIRAuthenticatorErrorMissingRequiredResponseFields userInfo:@{@"error_uri" : error}]);
        return;
    }
    
    FHIRToken *token = [FHIRToken tokenWithAccessToken:accessToken scope:scope type:tokenType expiresIn:expiresIn.unsignedIntegerValue patient:patient encounter:encounter];
    
    os_log_debug(OS_LOG_DEFAULT, "Got access token: %{public}@", accessToken);
    os_log_debug(OS_LOG_DEFAULT, "Got scope: %{public}@", scope);
    os_log_debug(OS_LOG_DEFAULT, "Got token type: %{public}@", tokenType);
    os_log_debug(OS_LOG_DEFAULT, "Got expiry: %lu", expiresIn.unsignedIntegerValue);
    os_log_debug(OS_LOG_DEFAULT, "Got patient: %{public}@", patient);
    os_log_debug(OS_LOG_DEFAULT, "Got encounter: %{public}@", encounter);
    
    FHIRRefreshToken *rfrshToken = nil;
    if (refreshToken) {
        os_log_debug(OS_LOG_DEFAULT, "Got refresh token: %{public}@", refreshToken);
        rfrshToken = [FHIRRefreshToken tokenWithToken:refreshToken];
    }
    
    FHIRIDToken *idToken = nil;
    if (openIDToken) {
        os_log_debug(OS_LOG_DEFAULT, "Got ID token: %{public}@", openIDToken);
        idToken = [FHIRIDToken tokenWithToken:openIDToken];
    }
    
    completionHandler(YES, token, rfrshToken, idToken, nil);
}

/**
 * Handles the results of requesting a new FHIR token using a refresh token.
 * @param response          The response from the server.
 * @param data              The response data from the server.
 * @param error             The error associated with the operation.
 * @param completionHandler Block to execute once processing is complete.
 */
- (void)handleRefreshResponse:(NSURLResponse *)response withData:(nullable NSData *)data error:(nullable NSError *)error completionHandler:(FHIRAuthenticatorTokenRefreshCompletionHandler)completionHandler
{
    if (![response isKindOfClass:[NSHTTPURLResponse class]]) {
        completionHandler(NO, nil, nil, [NSError errorWithDomain:FHIRAuthenticatorErrorDomain code:FHIRAuthenticatorErrorInvalidResponse userInfo:nil]);
        return;
    }
    
    if (error) {
        os_log_error(OS_LOG_DEFAULT, "Encountered an error during token refresh: %{public}@", error.debugDescription);
        completionHandler(NO, nil, nil, [NSError errorWithDomain:FHIRAuthenticatorErrorDomain code:FHIRAuthenticatorErrorRequestFailed userInfo:@{NSUnderlyingErrorKey : error}]);
        return;
    }
    
    NSHTTPURLResponse *httpResponse = (NSHTTPURLResponse *)response;
    NSInteger httpStatus = httpResponse.statusCode;
    if (httpStatus != 200) {
        os_log_error(OS_LOG_DEFAULT, "Token refresh did not complete successfully: %ld", httpStatus);
        completionHandler(NO, nil, nil, [NSError errorWithDomain:FHIRAuthenticatorErrorDomain code:FHIRAuthenticatorErrorRequestFailed userInfo:@{@"status" : @(httpStatus)}]);
        return;
    }
    
    if (!data) {
        completionHandler(NO, nil, nil, [NSError errorWithDomain:FHIRAuthenticatorErrorDomain code:FHIRAuthenticatorErrorEmptyResponse userInfo:nil]);
        return;
    }
    
    NSError *jsonError = nil;
    id payload = [NSJSONSerialization JSONObjectWithData:data options:kNilOptions error:&jsonError];
    
    if (!payload || jsonError) {
        os_log_error(OS_LOG_DEFAULT, "Encountered a conversion error during token refresh: %{public}@", jsonError.debugDescription);
        completionHandler(NO, nil, nil, [NSError errorWithDomain:FHIRAuthenticatorErrorDomain code:FHIRAuthenticatorErrorUnparseableResponse userInfo:(jsonError ? @{NSUnderlyingErrorKey : jsonError} : nil)]);
        return;
    }
    
    if (![payload isKindOfClass:[NSDictionary class]]) {
        completionHandler(NO, nil, nil, [NSError errorWithDomain:FHIRAuthenticatorErrorDomain code:FHIRAuthenticatorErrorIncorrectResponse userInfo:nil]);
        return;
    }
    
    NSDictionary *exchangePayload = (NSDictionary *)payload;
    NSString *exchangeError = exchangePayload[@"error_uri"];
    NSString *accessToken = exchangePayload[@"access_token"];
    NSString *scope = exchangePayload[@"scope"];
    NSString *tokenType = exchangePayload[@"token_type"];
    NSNumber *expiresIn = exchangePayload[@"expires_in"];
    NSString *patient = exchangePayload[@"patient"];
    NSString *encounter = exchangePayload[@"encounter"];
    NSString *openIDToken = exchangePayload[@"id_token"];
    
    if (exchangeError) {
        os_log_error(OS_LOG_DEFAULT, "Encountered error for token refresh: %{public}@", error);
        completionHandler(NO, nil, nil, [NSError errorWithDomain:FHIRAuthenticatorErrorDomain code:FHIRAuthenticatorErrorTokenRefreshWorkflowFailed userInfo:@{@"error_uri" : error}]);
        return;
    }
    else if (!accessToken || !scope || !tokenType || expiresIn == nil) {
        completionHandler(NO, nil, nil, [NSError errorWithDomain:FHIRAuthenticatorErrorDomain code:FHIRAuthenticatorErrorMissingRequiredResponseFields userInfo:@{@"error_uri" : error}]);
        return;
    }
    
    FHIRToken *token = [FHIRToken tokenWithAccessToken:accessToken scope:scope type:tokenType expiresIn:expiresIn.unsignedIntegerValue patient:patient encounter:encounter];
    
    os_log_debug(OS_LOG_DEFAULT, "Got access token: %{public}@", accessToken);
    os_log_debug(OS_LOG_DEFAULT, "Got scope: %{public}@", scope);
    os_log_debug(OS_LOG_DEFAULT, "Got token type: %{public}@", tokenType);
    os_log_debug(OS_LOG_DEFAULT, "Got expiry: %lu", expiresIn.unsignedIntegerValue);
    os_log_debug(OS_LOG_DEFAULT, "Got patient: %{public}@", patient);
    os_log_debug(OS_LOG_DEFAULT, "Got encounter: %{public}@", encounter);
    
    FHIRIDToken *idToken = nil;
    if (openIDToken) {
        os_log_debug(OS_LOG_DEFAULT, "Got ID token: %{public}@", openIDToken);
        idToken = [FHIRIDToken tokenWithToken:openIDToken];
    }
    
    completionHandler(YES, token, idToken, nil);
}

/**
 * Determines if a authorization grant workflow callback is valid and can be used.
 * @param callbackURL The URL received and to be checked.
 */
- (void)validateAuthorizationGrantCallback:(NSURL *)callbackURL
{
    __block NSString *code = nil;
    __block NSString *state = nil;
    __block NSString *error = nil;
    NSURLComponents *components = [NSURLComponents componentsWithURL:callbackURL resolvingAgainstBaseURL:NO];
    [components.queryItems enumerateObjectsUsingBlock:^(NSURLQueryItem *_Nonnull obj, NSUInteger idx, BOOL *_Nonnull stop) {
        if ([obj.name isEqualToString:@"state"]) {
            state = obj.value;
        }
        else if ([obj.name isEqualToString:@"code"]) {
            code = obj.value;
        }
        else if ([obj.name isEqualToString:@"error_uri"]) {
            error = obj.value;
        }
    }];
    
    if (error && [self.currentState isEqualToString:state]) {
        os_log_error(OS_LOG_DEFAULT, "Encountered error for authorization grant callback: %{public}@", error);
        [self.webView loadRequest:[NSURLRequest requestWithURL:[NSURL URLWithString:error] cachePolicy:NSURLRequestReloadIgnoringLocalCacheData timeoutInterval:30]];
        if (self.authGrantCompletionHandler) {
            self.authGrantCompletionHandler(NO, nil, [NSError errorWithDomain:FHIRAuthenticatorErrorDomain code:FHIRAuthenticatorErrorAuthorizationGrantWorkflowFailed userInfo:@{@"error_uri" : error}]);
        }
    }
    else if (error) {
        if (self.authGrantCompletionHandler) {
            self.authGrantCompletionHandler(NO, nil, [NSError errorWithDomain:FHIRAuthenticatorErrorDomain code:FHIRAuthenticatorErrorAuthorizationGrantStateMismatch userInfo:@{@"error_uri" : error}]);
        }
    }
    else if (code && [self.currentState isEqualToString:state]) {
        os_log_debug(OS_LOG_DEFAULT, "Received an authorization grant: %{public}@", code);
        if (self.authGrantCompletionHandler) {
            self.authGrantCompletionHandler(YES, code, nil);
        }
    }
    else if (code) {
        if (self.authGrantCompletionHandler) {
            self.authGrantCompletionHandler(NO, nil, [NSError errorWithDomain:FHIRAuthenticatorErrorDomain code:FHIRAuthenticatorErrorAuthorizationGrantStateMismatch userInfo:nil]);
        }
    }
    else {
        if (self.authGrantCompletionHandler) {
            self.authGrantCompletionHandler(NO, nil, [NSError errorWithDomain:FHIRAuthenticatorErrorDomain code:FHIRAuthenticatorErrorAuthorizationGrantInvalidCallback userInfo:nil]);
        }
    }
    
    self.authGrantCompletionHandler = nil;
}

#pragma mark - NSObject

- (NSString *)debugDescription
{
    return [NSString stringWithFormat:@"<%@: %p> {\n\tFHIR Base URL: %@\n\tCallback Scheme: %@\n\tClient Identifier: %@\n\tAuthorization URL: %@\n\tIntrospection URL: %@\n\tToken URL: %@\n\tCurrent State: %@\n\tWeb View: %@\n}",
            NSStringFromClass([self class]),
            (void *)self,
            self.FHIRBaseURL.absoluteString,
            self.callbackScheme,
            self.clientIdentifier,
            self.authorizationURL.absoluteString,
            self.introspectionURL.absoluteString,
            self.tokenURL.absoluteString,
            self.currentState,
            self.webView.debugDescription];
}

@end
```

Now, SMART on FHIR does not define a standard for session based controls such as locking/unlocking or logout, so if you know more about how you are authenticating to the particular FHIR instance more fields will be included in the authorization grant exchange response (e.g. session lifetime). If that is the case, you will need to extend the authenticator to handle reauthentication and logout. Also, error handling is at the discretion of the application.

For refresh tokens, you can create a timer that should periodically go and get new tokens based off of the token's half-life. Once you get the new token, you should update your method of injecting the token into the headers for the transactions.

Lastly, since the OpenID Connect tokens are used primarily on the server, the token model object does not validate it nor decode it. However, if you have need to do this on the application-side, you will need to leverage the appropriate [JWK](https://tools.ietf.org/html/rfc7517) endpoint to validate it and then decode the JWT yourself. The decoded values can be added to the model object via a category/subclassing or by modifying the class directly.

---

If you want to play around in the browser, you can use the [demo app](https://launch.smarthealthit.org/) to test the workflows as well as get familiar with the payloads returned. You can also use the [JWT Debugger](https://jwt.io/#debugger-io) to inspect the tokens' contents.
