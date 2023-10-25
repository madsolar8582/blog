---
title: "SSH and Yubikeys"
subtitle: ""
date: 2023-10-25T18:15:00-05:00
tags: ["ssh", "git", "yubikey"]
---

[Yubikeys](https://www.yubico.com/products/yubikey-fips/) are a popular hardware security token that can be leveraged for WebAuthn/FIDO2, OTP/TOTP, and Smart Card/PIV purposes. Since SSH supports FIDO security keys and Git leverages SSH for operations, you can use them for Git over SSH as well as commit signing via SSH keys. However, there are some prerequisites. 

First, the Yubikey must be on [firmware](https://docs.yubico.com/hardware/yubikey/yk-tech-manual/) version 5.2.3 or newer for `ed25519-sk` key pairs. Firmware prior to this only supports `ecdsa-sk` and ECDSA is [not recommended](https://blog.trailofbits.com/2020/06/11/ecdsa-handle-with-care/) by the cryptographic community. Unfortunately, but for legitimate security reasons, Yubikey firmware is not upgradable. You can validate what firmware version the Yubikey is on using [Yubikey Manager](https://www.yubico.com/support/download/yubikey-manager/) (this also allows you to set the FIDO PIN which is required).

Second, the client and server should be using OpenSSH 8.3 or newer for full compatibility.

Third, if not using public GitHub, you need to be using GitHub Enterprise 3.7 or newer.

Lastly, you need to be using Git 2.34 or newer.

Assuming all of those requirements are met, you then need to decide between a Discoverable Credential (Resident) or a Non-Discoverable Credential (Non-Resident). A discoverable credential will reside on the Yubikey itself, which makes it portable whereas a non-discoverable credential will still create a key pair on disk tied to the Yubikey. In order for that key pair to be used across multiple machines, the key pair will need to be copied. Therefore, this decision is based off of the tradeoff between convenience and security.

For Discoverable Credentials, you would generate the key pair as such:
```bash
ssh-keygen -t ed25519-sk -a 100 -C "email@example.com" -O resident -O application=ssh:MyGitHubKey -O verify-required
```

Now, if you don't want to enter the PIN and touch the Yubikey, you can pass `-O no-touch-required` instead. Likewise, you can pass `-O verify-required -O no-touch-required` to only require the PIN. Lastly, if you only want to touch the Yubikey, omit the `-O verify-required` flag.

For Non-Discoverable Credentials, you would generate the key pair as such:
```bash
ssh-keygen -t ed25519-sk -a 100 -C "email@example.com"
```

Regardless of which method was used, you now have a public key (default name of `id_ed25519_sk.pub`) that needs to be added to GitHub as an Authentication Key. Once added, you can verify connectivity:
```bash
ssh -T git@github.com
```

For commit and tag signing, the public key needs to be added to GitHub a second time as a Signing Key. Once that is done, you need to configure Git to use the SSH key for signing commits and tags:
```bash
git config --global gpg.format ssh
git config --global user.signingkey ~/.ssh/id_ed25519_sk.pub
git config --global gpg.ssh.allowedSignersFile ~/.ssh/allowed_signers
git config --global commit.gpgsign true
git config --global tag.gpgsign true
cat ~/.ssh/id_ed25519_sk.pub >> ~/.ssh/allowed_signers
```

With this configuration, commits and tags made by you will be signed. To verify tags, do `git tag -v <tag>` and to verify commits, do `git log --show-signature`. Merging and pulling, however, have a few options. Using `--verify-signatures non-verify` will abort if invalid signatures/unsigned commits are detected. Use `--verify-signatures -S  signed-branch` to validate and sign the resulting commit assuming all commits have valid signatures.

---

For more information about `ssh-keygen`, please review the [man page](http://manpages.ubuntu.com/manpages/precise/en/man1/ssh-keygen.html).
