---
title: "YubiKey and GPG Agent Fuzziness — Continued, Part 2"
date: 2026-09-03T00:22:09+02:00
draft: false
description: "One more recovery step when pcscd hits systemd's start limit."
---

A tiny follow-up to [my earlier YubiKey and gpg-agent workaround](https://s.norrs.no/posts/yubikey-and-gpg-agent-fuzziness-and-chrome-stealing-its-exclusive-access/): this time, a soft USB replug was not enough.

`ssh-add -L` still listed my public key, but with `(none)` instead of the usual `cardno:…` comment. Meanwhile, `gpg --card-status` reported `Service is not running`.

My `scdaemon.conf` contains `disable-ccid`, so GPG relies on PC/SC to reach the YubiKey. `pcscd.service` had failed repeatedly, leaving its socket stuck in `service-start-limit-hit`. Restarting `gpg-agent` or replugging the USB device did not clear that state.

The recovery was:

```bash
sudo systemctl reset-failed pcscd.service pcscd.socket
sudo systemctl restart pcscd.socket
gpgconf --kill scdaemon
fix-gpgagent
```

After that, `gpg --card-status` could read the card again, and `ssh-add -L` showed the serial-number comment. I have not tracked down the original daemon failure yet.

I added a matching [`install_pcscd_reset_sudo`](https://github.com/norrs/dotfiles/blob/main/dot.tasks/install_pcscd_reset_sudo) installer. Run it once:

```bash
~/dotfiles/dot.tasks/install_pcscd_reset_sudo
```

It installs `/usr/local/sbin/pcscd-reset-helper` with a narrowly scoped, no-argument `NOPASSWD` sudo rule. [`fix-gpgagent`](https://github.com/norrs/dotfiles/blob/main/dot.local/bin/fix-gpgagent) now calls it when the PC/SC socket is inactive or the service has failed, before continuing its existing cleanup.

Same one-command workaround, one more layer of the stack to un-wedge.
