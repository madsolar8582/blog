---
title: "Goodbye macOS Server"
subtitle: ""
date: 2022-04-28T07:45:00-05:00
tags: ["server", "apple"]
---

The time has finally come. After years of removing features and reducing the overall value of the product, Apple has finally [discontinued macOS Server](https://support.apple.com/en-us/HT208312). Going forward, it appears that the Server app will not function with the next release of macOS and users leveraging the last two features (Profile Manager and Open Directory) will need to move to other implementations.

For Profile Manager, you'll need to invest in a MDM vendor for Zero Touch configuration. Otherwise, you can continue to use [Apple Configurator](https://support.apple.com/apple-configurator) to manage devices (except Macs).

For Directory Manager, you can use [OpenLDAP](https://www.openldap.org/), or, use [Active Directory](https://docs.microsoft.com/en-us/windows-server/identity/ad-ds/active-directory-domain-services) via Windows Server.

It's disappointing, but not unexpected as Apple has been moving out of the Prosumer / Enterprise-lite business for a while with the launches of [Apple Business Manager](https://business.apple.com/) and [Apple School Manager](https://school.apple.com/) (including also shutting down [Fleetsmith](https://support.apple.com/en-us/HT213238)).

---

As an aside, I'm hoping that this year Apple talks more publicly about how Xcode Cloud works as macOS virtualization is still a hot topic and one highly sought by enterprises and developers alike. 
