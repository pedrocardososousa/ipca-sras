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
```

---

## 🔌 Verify mod_proxy Is Enabled

```bash
grep "mod_proxy" /etc/httpd/conf.modules.d/00-proxy.conf
```

---

## 🔁 Configure Apache Reverse Proxy with Load Balancing

Create file:

```bash
vi /etc/httpd/conf.d/revers_proxy.conf
```

Insert:

```apache
<Proxy balancer://myset>
    BalancerMember http://192.168.27.211:80
    BalancerMember http://192.168.27.212:80
    ProxySet lbmethod=bytraffic
</Proxy>

ProxyPass "/" "balancer://myset/"
ProxyPassReverse "/" "balancer://myset/"
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