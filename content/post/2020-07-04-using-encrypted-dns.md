---
title: "Using Encrypted DNS"
date: 2020-07-04T09:00:00-05:00
tags: ["dot", "doh", "encrypted dns"]
---

Starting with this year's releases, the OSes now support system-wide DNS resolvers that support [DNS over TLS](https://en.wikipedia.org/wiki/DNS_over_TLS) (DoT) and [DNS over HTTPS](https://en.wikipedia.org/wiki/DNS_over_HTTPS) (DoH). This is an important privacy and security feature for users as this allows for these devices to be less susceptible to traffic hijacking. That being said, resolvers that implement these features are rare, so APIs were added so that apps can use fallback resolvers in the event that the system resolver does not support encrypted DNS.

The way to enable fallback encrypted DNS is a bit weird, you must enable the resolver on the `nw_privacy_context_t` used by transactions. By using the default one, the configuration applies to all C name lookups, all NSURLSession name lookups, and Network.framework lookups using that context. However, if you are using Network.framework for transactions, you can supply various privacy contexts with different settings rather than use the default one.

Regardless of which context you use, the code is pretty straightforward:

### DoH

```objective-c
- (void)setupDoH
{
  nw_endpoint_t dohEndpoint = nw_endpoint_create_url("https://cloudflare-dns.com/dns-query");
  nw_resolver_config_t dohResolver = nw_resolver_config_create_https(dohEndpoint);
  nw_privacy_context_require_encrypted_name_resolution(NW_DEFAULT_PRIVACY_CONTEXT, true, dohResolver);
}
```

### DoT

```objective-c
- (void)setupDoT
{
  nw_endpoint_t dotEndpoint = nw_endpoint_create_host("cloudflare-dns.com", "853");
  nw_resolver_config_t = dotResolver = nw_resolver_config_create_tls(dotEndpoint);
  nw_privacy_context_require_encrypted_name_resolution(NW_DEFAULT_PRIVACY_CONTEXT, true, dotResolver);
}
```

---

There are some caveats though: You can only have one encrypted DNS resolver set as the fallback (can't use multiple for resiliency) and DNS over HTTPS is preferred as it masquerades as normal HTTPS traffic where some networks will block DNS over TLS as it can be identified by its port. Nevertheless, if you can, you should enable support for this feature as it will make your applications more secure and protect your users.
