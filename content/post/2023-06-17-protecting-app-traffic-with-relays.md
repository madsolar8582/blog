---
title: "Protecting App Traffic with Relays"
subtitle: ""
date: 2023-06-17T08:55:00-05:00
tags: ["masque", "ohttp", "relay"]
---

With iOS 15 and macOS 12, Apple introduced [iCloud Private Relay](https://support.apple.com/en-us/HT212614). This allows traffic to be protected from a degree of introspection and anonymizes your connection with the target host via [MASQUE](https://ietf-wg-masque.github.io/draft-ietf-masque-ip-proxy-reqs/draft-ietf-masque-ip-proxy-reqs.html) and [Oblivious HTTP](https://ietf-wg-ohai.github.io/oblivious-http/draft-ietf-ohai-ohttp.html) (OHTTP). If you combine this with encrypted DNS, you will have a much greater sense of privacy and security regarding your network traffic. Of course, some network operators take steps to disable these features to ensure that they can introspect and shape traffic on their network.

Regardless, starting in iOS 17 and macOS 14, you can configure the network stack to use your own MASQUE proxies and/or Oblivious HTTP endpoints. This process is pretty straightforward:

```obj-c
+ (void)setupRelays
{
  // Create the endpoint.
  nw_endpoint_t relay = nw_endpoint_create_url("https://relay.example.com");

  // Setup TLS.
  nw_protocol_options_t tlsConfig = nw_tls_create_options();
  // Configure the hop. You can have a second endpoint for the HTTP/2 fallback (the second parameter) if you desire.
  nw_relay_hop_t hop = nw_relay_hop_create(relay, relay, tlsConfig);

  // Configure the proxy. It allows for up to two distinct hops.
  nw_proxy_config_t relayConfig = nw_proxy_config_create_relay(hop, NULL);

  // Configure the username and password if required.
  nw_proxy_config_set_username_and_password(relayConfig, "username", "password");

  // Either allow or disallow traffic over unproxied connections if the connection to the proxy cannot be established.
  nw_proxy_config_set_failover_allowed(relayConfig, false);

  // Apply the configuration to the appropriate privacy context.
  nw_privacy_context_add_proxy(NW_DEFAULT_PRIVACY_CONTEXT, relayConfig);
}

+ (void)clearRelays
{
  // Remove all proxies from the appropriate privacy context.
  nw_privacy_context_clear_proxies(NW_DEFAULT_PRIVACY_CONTEXT);
}
```
Of course, if you want to use Oblivous HTTP instead of MASQUE, you need to use [`nw_proxy_config_create_oblivious_http`](https://developer.apple.com/documentation/network/4172961-nw_proxy_config_create_oblivious) instead.

Also, if you are using `NSURLSession` or `WKWebView`, similar APIs exist so that you can limit the scope of the traffic impacted by a relay:

```obj-c
+ (NSURLSession *)setupURLSession
{
  nw_endpoint_t relay = nw_endpoint_create_url("https://relay.example.com");
  nw_protocol_options_t tlsConfig = nw_tls_create_options();
  nw_relay_hop_t hop = nw_relay_hop_create(relay, relay, tlsConfig);
  nw_proxy_config_t relayConfig = nw_proxy_config_create_relay(hop, NULL);
  
  NSURLSessionConfiguration *sessionConfig = NSURLSessionConfiguration.defaultSessionConfiguration;
  sessionConfig.proxyConfigurations = @[relayConfig];
  
  return NSURLSession *session = [NSURLSession sessionWithConfiguration:sessionConfig];
}

+ (WKWebViewConfiguration *)setupWebViewConfiguration
{
  nw_endpoint_t relay = nw_endpoint_create_url("https://relay.example.com");
  nw_protocol_options_t tlsConfig = nw_tls_create_options();
  nw_relay_hop_t hop = nw_relay_hop_create(relay, relay, tlsConfig);
  nw_proxy_config_t relayConfig = nw_proxy_config_create_relay(hop, NULL);
  
  WKWebViewConfiguration *webConfig = [WKWebViewConfiguration new];
  webConfig.websiteDataStore = [WKWebsiteDataStore nonPersistentDataStore];
  webConfig.websiteDataStore.proxyConfigurations = @[relayConfig];

  return webConfig;
}
```

---

It is important to note that not every server or TCP/UDP protocol proxied over QUIC supports this kind of connection obfuscation, so you need to ensure that the endpoints you are connecting to fully support these modern protocols.
