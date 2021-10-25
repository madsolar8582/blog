---
title: "Fixing Sudo for Remote Users"
date: 2021-10-25T07:00:00-05:00
tags: ["macOS", "sudo"]
---

Starting with a recent-ish update to Big Sur, sudo commands would fail for users that are remote users (i.e. Active Directory/LDAP bound). The interesting part is that the sudoers file itself is fine and that some commands prefixed with sudo work and then eventually the rejected commands also start working. This seems to be due to the fact that the OS cannot successfully map the remote User ID and Group to the local admin group and local user account.

Luckily, there is a workaround: using /etc/sudoers.d with explicit users. By creating individual files with the username, those sudoers additions seem to not fall victim to the mapping problem. Unfortunately, sudo needs to be working to kick this process off. To create the files (1 by 1), you need to use visudo, `sudo visudo -f /etc/sudoers.d/<username>`, and then configure your sudo options:

```
<username> ALL = (ALL) ALL
```

Once that is done, sudo should no longer break.

---

Currently, I do not know if it is fixed in Monterey, but I don't want to risk removing my workaround. Honestly, I'm surprised this has been broken for this long, but I guess it is to be expected with an OS plagued by unaddressed Core Rot.
