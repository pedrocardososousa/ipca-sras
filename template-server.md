---
layout: single
title: "Template Server Configuration (Rocky Linux 9.5)"
permalink: /template-server/
---

# 🛡️ Rocky Linux 9.5 Template Server – Final Configuration

This guide documents the setup of a hardened Rocky Linux 9.5 template server using security profiles, GRUB protections, system auditing tools, and filesystem structure for reliable and secure cloning.

---

## 💽 Partition Scheme

Below is the detailed `/etc/fstab` layout used on the template server:

```fstab
/dev/mapper/rl-root     /                       xfs     defaults        0 0
UUID=11de96b3-e602-4e72-a81d-d8e72e9a57fb /boot                   xfs     defaults,noexec,nosuid,nodev 0 0
UUID=80C6-B51E          /boot/efi               vfat    umask=0077,shortname=winnt,nodev 0 2
/dev/mapper/rl-home     /home                   xfs     defaults,noexec,nosuid,nodev 0 0
/dev/mapper/rl-opt      /opt                    xfs     defaults,nosuid,nodev 0 0
/dev/mapper/rl-srv      /srv                    xfs     defaults,nosuid,nodev 0 0
/dev/mapper/rl-tmp      /tmp                    xfs     defaults,noexec,nosuid,nodev 0 0
/dev/mapper/rl-var      /var                    xfs     defaults,noexec,nosuid,nodev 0 0
/dev/mapper/rl-var_log  /var/log                xfs     defaults,noexec,nosuid,nodev 0 0
/dev/mapper/rl-var_tmp  /var/tmp                xfs     defaults,noexec,nosuid,nodev 0 0
/dev/mapper/rl-swap     none                    swap    defaults        0 0
```

---

## ✅ Final Server Hardening & Configuration

[...] (omitted for brevity, identical to previous content)