---
layout: single
title: "Apache Load Balancer with Keepalived"
permalink: /apache-keepalived/
---

# 🌐 Apache Load Balancer with Keepalived on Rocky Linux

This guide walks you through configuring Apache as a reverse proxy with load balancing, combined with Keepalived for high availability using a Virtual IP (VIP).

---

## 🧰 Prerequisites

- Rocky Linux server
- Two Frontend apache servers:
  - `192.168.27.221`
  - `192.168.27.222`
- Virtual IP
  - `192.168.27.220`
- Two backend WordPress nodes:
  - `192.168.27.211`
  - `192.168.27.212`

---

## ⚙️ Apache HTTP Server Installation

```bash
dnf update -y
dnf install httpd -y
```

### 🔥 Open Firewall Port

```bash
firewall-cmd --zone=public --add-port=80/tcp --permanent
firewall-cmd --reload
```

---

## 🚀 Start and Enable Apache

```bash
systemctl start httpd
systemctl status httpd
systemctl enable --now httpd
```

---
Source: https://httpd.apache.org/docs/2.4/howto/reverse_proxy.html

## 🔌 Verify mod_proxy Is Enabled

```bash
grep "mod_proxy" /etc/httpd/conf.modules.d/00-proxy.conf
```

---

## 🔁 Configure Apache Reverse Proxy with Load Balancing

Create file:

```bash
vi /etc/httpd/conf.d/reverse_proxy.conf
```

Insert:
Simple approach

```apache
<Proxy balancer://myset>
    BalancerMember http://192.168.27.211:80
    BalancerMember http://192.168.27.212:80
    ProxySet lbmethod=bytraffic
</Proxy>

ProxyPass "/" "balancer://myset/"
ProxyPassReverse "/" "balancer://myset/"
```

Virtual Host Approach (selected)
```apache
<VirtualHost *:80>
    ServerName demo.ipca.site

    # Enable ModSecurity
    <IfModule security2_module>
        SecRuleEngine On
    </IfModule>

    # Define the reverse proxy
    <Proxy "balancer://myset">
        BalancerMember http://192.168.27.211:80
        BalancerMember http://192.168.27.212:80
        ProxySet lbmethod=bytraffic
    </Proxy>

    ProxyPreserveHost On
    ProxyPass "/" "balancer://myset/"
    ProxyPassReverse "/" "balancer://myset/"

    ErrorLog /var/log/httpd/proxy_error.log
    CustomLog /var/log/httpd/proxy_access.log combined
</VirtualHost>
```

Restart Apache:

```bash
systemctl restart httpd
```

---

## 🛡️ Keepalived Installation

```bash
dnf install keepalived -y
```

Backup and configure:

```bash
mv /etc/keepalived/keepalived.conf /etc/keepalived/keepalived.conf.ori
vi /etc/keepalived/keepalived.conf
```

Example configuration:

```ini
vrrp_instance VI_DB {
    state MASTER
    interface ens160
    virtual_router_id 60
    priority 100
    advert_int 1

    authentication {
        auth_type PASS
        auth_pass Pass12345
    }

    virtual_ipaddress {
        192.168.27.220
    }
}
```
Change `state` and `priority` for other nodes (`BACKUP`, `90`, `80`, etc.).

Start and enable Keepalived:

```bash
systemctl enable --now keepalived
```
---

> ✅ This configuration sets up an Apache-based reverse proxy load balancer with Keepalived providing failover using a virtual IP address (`192.168.27.220`).

---

## 🔒 Install and Configure ModSecurity with CRS

### 1. Install ModSecurity

```bash
dnf install mod_security wget -y
```

### 2. Download and Configure OWASP Core Rule Set (CRS)

```bash
cd /etc/httpd/conf/
wget https://github.com/coreruleset/coreruleset/archive/refs/tags/v4.14.0.tar.gz
dnf install tar -y
tar xzvf v4.14.0.tar.gz
ln -s coreruleset-4.14.0/ /etc/httpd/conf/crs
rm -f v4.14.0.tar.gz
cp crs/crs-setup.conf.example crs/crs-setup.conf
```

### 3. Include CRS in ModSecurity Configuration
Source: https://docs.rockylinux.org/guides/web/apache_hardened_webserver/modsecurity/
Edit or create:

```bash
vi /etc/httpd/conf.d/mod_security.conf
```

Add this to end of the file:
```apache
Include    /etc/httpd/conf/crs/crs-setup.conf

SecAction "id:900110,phase:1,pass,nolog,\
    setvar:tx.inbound_anomaly_score_threshold=10000,\
    setvar:tx.outbound_anomaly_score_threshold=10000"

SecAction "id:900000,phase:1,pass,nolog,\
      setvar:tx.paranoia_level=1"


# === ModSec Core Rule Set: Runtime Exclusion Rules (ids: 10000-49999)

# ...


# === ModSecurity Core Rule Set Inclusion

Include    /etc/httpd/conf/crs/rules/*.conf


# === ModSec Core Rule Set: Startup Time Rules Exclusions
```

### 4. Restart Apache

```bash
systemctl restart httpd
systemctl status httpd
```

### 5. Check SELinux status

```bash
vi /etc/selinux/config
```

```ini
SELINUX=enforcing
```

```bash
/usr/sbin/setsebool -P httpd_can_network_connect 1
```
---


## Check Apache Settings

### Apache running user

Check running user
```bash
ps -eo user,comm | grep httpd
```

Output example:
```
root     httpd
apache   httpd
apache   httpd
apache   httpd
apache   httpd
```

Check apache configuration
```bash
grep -i "^User\|^Group" /etc/httpd/conf/httpd.conf
```

Output example:
```
User apache
Group apache
```

Check service status
```bash
systemctl status httpd
```

Output example (check main PID)
```
● httpd.service - The Apache HTTP Server
     Loaded: loaded (/usr/lib/systemd/system/httpd.service; enabled; preset: disabled)
     Active: active (running) since Thu 2025-05-29 09:28:37 WEST; 8min ago
       Docs: man:httpd.service(8)
   Main PID: 949 (httpd)
     Status: "Total requests: 0; Idle/Busy workers 100/0;Requests/sec: 0; Bytes served/sec:   0 B/sec"
      Tasks: 177 (limit: 10242)
     Memory: 79.7M
        CPU: 1.208s
     CGroup: /system.slice/httpd.service
             ├─ 949 /usr/sbin/httpd -DFOREGROUND
             ├─ 971 /usr/sbin/httpd -DFOREGROUND
             ├─ 972 /usr/sbin/httpd -DFOREGROUND
             ├─ 999 /usr/sbin/httpd -DFOREGROUND
             └─1000 /usr/sbin/httpd -DFOREGROUND
```

### Check file structure

```
/etc/httpd/conf.d/
├── mod_security.conf
├── reverse_proxy.conf
├── crs/
│   ├── crs-setup.conf
│   └── rules/
```
> ✅ This setup uses Apache as a reverse proxy load balancer, managed with Keepalived for high availability and protected with ModSecurity using the OWASP Core Rule Set (CRS).
