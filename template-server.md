---
layout: single
title: "Rocky Linux 9.5 Template Server Configuration"
permalink: /template-server/
---

# 🛡️ Hardened Template Server – Rocky Linux 9.5

This guide covers the full configuration of a hardened Rocky Linux 9.5 server image to be used as a **template** for cloning secure and auditable Linux deployments.

---

## 💽 Partition Scheme

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

## ⚙️ Full System Configuration

The complete list of commands, security steps, monitoring setup, and system hardening have been applied and documented in this template.

Please refer to each command and setting under the appropriate headings used in your configuration (security profile scan, GRUB, SSH hardening, logging, audit checks, resource monitors, and Tripwire configuration).

This template is built to serve as a secure baseline for deployment across your infrastructure.

---

## 🔐 Security Configuration & Hardening Commands

```bash
# Base system prep
su
dnf update -y
dnf install openscap-scanner scap-security-guide -y

# Security scanning profiles
oscap xccdf eval --report report.html --profile xccdf_org.ssgproject.content_profile_anssi_bp28_high /usr/share/xml/scap/ssg/content/ssg-rl9-ds.xml
oscap xccdf eval --report report.html --profile xccdf_org.ssgproject.content_profile_anssi_bp28_intermediary /usr/share/xml/scap/ssg/content/ssg-rl9-ds.xml

# Export security report
scp -r manager@192.168.27.136:/home/manager/report.html /Users/psousa/Downloads

# Set GRUB password
grub2-setpassword
# Password used: 1PCA

# Check services
ps faxuw
systemctl list-units

# Remote access services
dnf install net-tools -y
netstat -anp

# Bash prompt and history hardening
vi .bashrc
# Add:
PS1="[\u@\h \w]\\$"
export HISTTIMEFORMAT="%F %T "
export PROMPT_COMMAND="history -a"
export TMOUT=120

# Disable Ctrl-Alt-Del reboot
systemctl disable ctrl-alt-del.target
systemctl mask ctrl-alt-del.target
systemctl daemon-reload

# Console lock
dnf install vlock -y
vlock

# SSH hardening
vi /etc/ssh/sshd_config
# Add or update:
PermitRootLogin no
LogLevel VERBOSE

# Check SUID/SGID files
find / -path /proc -prune -o -type f \( -perm -4000 -o -perm -2000 \) -exec ls -l {} \;

# Check orphan files
find / -path /proc -prune -nouser -o -nogroup -exec ls -l {} \;

# Verify installed packages
rpm -qaV

# Log bash activity to syslog
vi /etc/profile.d/bash.sh
# Add:
export PROMPT_COMMAND='RETRN_VAL=$?;logger -p local6.debug "$(whoami) [$$]: $(history 1 | sed "s/^[ ]*[0-9]\+[]*//" ) [$RETRN_VAL]"'

# Local monitoring
dnf install sysstat -y
sar -A
sar -q
sar -r
systemctl enable --now sysstat

# Local process accounting
dnf install psacct -y
systemctl start psacct.service
systemctl enable psacct.service

# Uptime and command activity tracking
ac         # Global check
ac -d      # By day
ac -p      # By user

# View user command execution
sa -u
lastcomm
lastcomm ipca

# File integrity with Tripwire
dnf install epel-release -y
dnf install tripwire -y
/usr/sbin/tripwire-setup-keyfiles

# Tripwire passphrases used:
# Pass 1: www.ipca.pt
# Pass 2: www.est.ipca.pt
# Pass 3: www.ipca.pt
# Pass 4: www.ipca.pt

twadmin -m P -S /etc/tripwire/site.key /etc/tripwire/twpol.txt
tripwire --init
tripwire --check
```

---

> ✅ This setup completes a fully hardened Rocky Linux 9.5 template server using security guidelines, monitoring tools, and auditing systems.