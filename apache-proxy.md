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

### 🛡️ Step 1: Install ClamAV on Rocky Linux

- Install ClamAV and related packages:
```bash
dnf install epel-release -y
dnf install clamav clamd clamav-update -y
```

- Create a dedicated service account for ClamAV:
```bash
groupadd clamav
useradd -g clamav -s /bin/false -c "Clam Antivirus" clamav
```

- Update the ClamAV virus database:
```bash
freshclam
```

- Enable ClamAV scanning permissions in SELinux:
```bash
setsebool -P antivirus_can_scan_system 1
```

- Start and enable the ClamAV daemon:
```bash
systemctl enable --now clamd@scan
```

- Edit the ClamAV config:
```bash
vi /etc/clamd.d/scan.conf
```

- Use a Unix Socket (recommended for local use)
Uncomment and configure the following:
```ini
LocalSocket /run/clamd.scan/clamd.sock
```

- Make sure the socket directory exists:
```bash
mkdir -p /run/clamd.scan
chown clamscan:clamscan /run/clamd.scan
```

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

## 🧪 Test Scenarios

### ✅ Scenario 1: Apache HA (Keepalived)
1. Shutdown one Apache server.
2. Ensure the virtual IP (192.168.27.220) is still available.
3. Confirm Proxy still routes traffic.

### ✅ Scenario 2: ModSecurity XSS Block
Run this comamand in the client
```bash
curl -H "User-Agent: <script>alert('xss')</script>" http://192.168.27.220/
```
Expected result: HTTP 403 blocked by ModSecurity.


```bash
cat /var/log/httpd/modsec_audit.log
```

Output:
```
[29/May/2025:09:46:03.938110 +0100] aDgeytMzrRSZmCAplzSpzgAAAJg 192.168.27.1 54802 192.168.27.220 80
--e1cd7d3c-B--
GET / HTTP/1.1
Host: 192.168.27.220
Accept: */*
User-Agent: <script>alert('xss')</script>

--e1cd7d3c-F--
HTTP/1.1 200 OK
X-Powered-By: PHP/8.2.28
Link: <http://demo.ipca.site/wp-json/>; rel="https://api.w.org/"
Vary: Accept-Encoding
Content-Type: text/html; charset=UTF-8
Transfer-Encoding: chunked

--e1cd7d3c-E--

--e1cd7d3c-H--
Message: Warning. Pattern match "(?:^([\\d.]+|\\[[\\da-f:]+\\]|[\\da-f:]+)(:[\\d]+)?$)" at REQUEST_HEADERS:Host. [file "/etc/httpd/conf/crs/rules/REQUEST-920-PROTOCOL-ENFORCEMENT.conf"] [line "730"] [id "920350"] [msg "Host header is a numeric IP address"] [data "192.168.27.220"] [severity "WARNING"] [ver "OWASP_CRS/4.14.0"] [tag "application-multi"] [tag "language-multi"] [tag "platform-multi"] [tag "attack-protocol"] [tag "paranoia-level/1"] [tag "OWASP_CRS"] [tag "OWASP_CRS/PROTOCOL-ENFORCEMENT"] [tag "capec/1000/210/272"] [tag "PCI/6.5.10"]
Message: Warning. detected XSS using libinjection. [file "/etc/httpd/conf/crs/rules/REQUEST-941-APPLICATION-ATTACK-XSS.conf"] [line "102"] [id "941100"] [msg "XSS Attack Detected via libinjection"] [data "Matched Data: XSS data found within REQUEST_HEADERS:User-Agent: <script>alert('xss')</script>"] [severity "CRITICAL"] [ver "OWASP_CRS/4.14.0"] [tag "application-multi"] [tag "language-multi"] [tag "platform-multi"] [tag "attack-xss"] [tag "xss-perf-disable"] [tag "paranoia-level/1"] [tag "OWASP_CRS"] [tag "OWASP_CRS/ATTACK-XSS"] [tag "capec/1000/152/242"]
Message: Warning. Pattern match "(?i)<script[^>]*>[\\s\\S]*?" at REQUEST_HEADERS:User-Agent. [file "/etc/httpd/conf/crs/rules/REQUEST-941-APPLICATION-ATTACK-XSS.conf"] [line "130"] [id "941110"] [msg "XSS Filter - Category 1: Script Tag Vector"] [data "Matched Data: <script> found within REQUEST_HEADERS:User-Agent: <script>alert('xss')</script>"] [severity "CRITICAL"] [ver "OWASP_CRS/4.14.0"] [tag "application-multi"] [tag "language-multi"] [tag "platform-multi"] [tag "attack-xss"] [tag "xss-perf-disable"] [tag "paranoia-level/1"] [tag "OWASP_CRS"] [tag "OWASP_CRS/ATTACK-XSS"] [tag "capec/1000/152/242"]
Message: Warning. Pattern match "(?i)<[^0-9<>A-Z_a-z]*(?:[^\\s\\x0b\"'<>]*:)?[^0-9<>A-Z_a-z]*[^0-9A-Z_a-z]*?(?:s[^0-9A-Z_a-z]*?(?:c[^0-9A-Z_a-z]*?r[^0-9A-Z_a-z]*?i[^0-9A-Z_a-z]*?p[^0-9A-Z_a-z]*?t|t[^0-9A-Z_a-z]*?y[^0-9A-Z_a-z]*?l[^0-9A-Z_a-z]*?e|v[^0-9A-Z_a-z]*?g|e[^0-9A-Z_a-z]*?t[^0- ..." at REQUEST_HEADERS:User-Agent. [file "/etc/httpd/conf/crs/rules/REQUEST-941-APPLICATION-ATTACK-XSS.conf"] [line "225"] [id "941160"] [msg "NoScript XSS InjectionChecker: HTML Injection"] [data "Matched Data: <script found within REQUEST_HEADERS:User-Agent: <script>alert('xss')</script>"] [severity "CRITICAL"] [ver "OWASP_CRS/4.14.0"] [tag "application-multi"] [tag "language-multi"] [tag "platform-multi"] [tag "attack-xss"] [tag "xss-perf-disable"] [tag "paranoia-level/1"] [tag "OWASP_CRS"] [tag "OWASP_CRS/ATTACK-XSS"] [tag "capec/1000/152/242"]
Message: Warning. Unconditional match in SecAction. [file "/etc/httpd/conf/crs/rules/RESPONSE-980-CORRELATION.conf"] [line "98"] [id "980170"] [msg "Anomaly Scores: (Inbound Scores: blocking=18, detection=18, per_pl=18-0-0-0, threshold=10000) - (Outbound Scores: blocking=0, detection=0, per_pl=0-0-0-0, threshold=10000) - (SQLI=0, XSS=15, RFI=0, LFI=0, RCE=0, PHPI=0, HTTP=0, SESS=0, COMBINED_SCORE=18)"] [ver "OWASP_CRS/4.14.0"] [tag "reporting"] [tag "OWASP_CRS"]
Apache-Error: [file "apache2_util.c"] [line 271] [level 3] [client 192.168.27.1] ModSecurity: Warning. Pattern match "(?:^([\\\\\\\\d.]+|\\\\\\\\[[\\\\\\\\da-f:]+\\\\\\\\]|[\\\\\\\\da-f:]+)(:[\\\\\\\\d]+)?$)" at REQUEST_HEADERS:Host. [file "/etc/httpd/conf/crs/rules/REQUEST-920-PROTOCOL-ENFORCEMENT.conf"] [line "730"] [id "920350"] [msg "Host header is a numeric IP address"] [data "192.168.27.220"] [severity "WARNING"] [ver "OWASP_CRS/4.14.0"] [tag "application-multi"] [tag "language-multi"] [tag "platform-multi"] [tag "attack-protocol"] [tag "paranoia-level/1"] [tag "OWASP_CRS"] [tag "OWASP_CRS/PROTOCOL-ENFORCEMENT"] [tag "capec/1000/210/272"] [tag "PCI/6.5.10"] [hostname "192.168.27.220"] [uri "/"] [unique_id "aDgeytMzrRSZmCAplzSpzgAAAJg"]
Apache-Error: [file "apache2_util.c"] [line 271] [level 3] [client 192.168.27.1] ModSecurity: Warning. detected XSS using libinjection. [file "/etc/httpd/conf/crs/rules/REQUEST-941-APPLICATION-ATTACK-XSS.conf"] [line "102"] [id "941100"] [msg "XSS Attack Detected via libinjection"] [data "Matched Data: XSS data found within REQUEST_HEADERS:User-Agent: <script>alert('xss')</script>"] [severity "CRITICAL"] [ver "OWASP_CRS/4.14.0"] [tag "application-multi"] [tag "language-multi"] [tag "platform-multi"] [tag "attack-xss"] [tag "xss-perf-disable"] [tag "paranoia-level/1"] [tag "OWASP_CRS"] [tag "OWASP_CRS/ATTACK-XSS"] [tag "capec/1000/152/242"] [hostname "192.168.27.220"] [uri "/"] [unique_id "aDgeytMzrRSZmCAplzSpzgAAAJg"]
Apache-Error: [file "apache2_util.c"] [line 271] [level 3] [client 192.168.27.1] ModSecurity: Warning. Pattern match "(?i)<script[^>]*>[\\\\\\\\s\\\\\\\\S]*?" at REQUEST_HEADERS:User-Agent. [file "/etc/httpd/conf/crs/rules/REQUEST-941-APPLICATION-ATTACK-XSS.conf"] [line "130"] [id "941110"] [msg "XSS Filter - Category 1: Script Tag Vector"] [data "Matched Data: <script> found within REQUEST_HEADERS:User-Agent: <script>alert('xss')</script>"] [severity "CRITICAL"] [ver "OWASP_CRS/4.14.0"] [tag "application-multi"] [tag "language-multi"] [tag "platform-multi"] [tag "attack-xss"] [tag "xss-perf-disable"] [tag "paranoia-level/1"] [tag "OWASP_CRS"] [tag "OWASP_CRS/ATTACK-XSS"] [tag "capec/1000/152/242"] [hostname "192.168.27.220"] [uri "/"] [unique_id "aDgeytMzrRSZmCAplzSpzgAAAJg"]
Apache-Error: [file "apache2_util.c"] [line 271] [level 3] [client 192.168.27.1] ModSecurity: Warning. Pattern match "(?i)<[^0-9<>A-Z_a-z]*(?:[^\\\\\\\\s\\\\\\\\x0b\\\\"'<>]*:)?[^0-9<>A-Z_a-z]*[^0-9A-Z_a-z]*?(?:s[^0-9A-Z_a-z]*?(?:c[^0-9A-Z_a-z]*?r[^0-9A-Z_a-z]*?i[^0-9A-Z_a-z]*?p[^0-9A-Z_a-z]*?t|t[^0-9A-Z_a-z]*?y[^0-9A-Z_a-z]*?l[^0-9A-Z_a-z]*?e|v[^0-9A-Z_a-z]*?g|e[^0-9A-Z_a-z]*?t[^0- ..." at REQUEST_HEADERS:User-Agent. [file "/etc/httpd/conf/crs/rules/REQUEST-941-APPLICATION-ATTACK-XSS.conf"] [line "225"] [id "941160"] [msg "NoScript XSS InjectionChecker: HTML Injection"] [data "Matched Data: <script found within REQUEST_HEADERS:User-Agent: <script>alert('xss')</script>"] [severity "CRITICAL"] [ver "OWASP_CRS/4.14.0"] [tag "application-multi"] [tag "language-multi"] [tag "platform-multi"] [tag "attack-xss"] [tag "xss-perf-disable"] [tag "paranoia-level/1"] [tag "OWASP_CRS"] [tag "OWASP_CRS/ATTACK-XSS"] [tag "capec/1000/152/242"] [hostname "192.168.27.220"] [uri "/"] [unique_id "aDgeytMzrRSZmCAplzSpzgAAAJg"]
Apache-Error: [file "apache2_util.c"] [line 271] [level 3] [client 192.168.27.1] ModSecurity: Warning. Unconditional match in SecAction. [file "/etc/httpd/conf/crs/rules/RESPONSE-980-CORRELATION.conf"] [line "98"] [id "980170"] [msg "Anomaly Scores: (Inbound Scores: blocking=18, detection=18, per_pl=18-0-0-0, threshold=10000) - (Outbound Scores: blocking=0, detection=0, per_pl=0-0-0-0, threshold=10000) - (SQLI=0, XSS=15, RFI=0, LFI=0, RCE=0, PHPI=0, HTTP=0, SESS=0, COMBINED_SCORE=18)"] [ver "OWASP_CRS/4.14.0"] [tag "reporting"] [tag "OWASP_CRS"] [hostname "192.168.27.220"] [uri "/"] [unique_id "aDgeytMzrRSZmCAplzSpzgAAAJg"]
Apache-Handler: proxy-server
Stopwatch: 1748508362657061 1281109 (- - -)
Stopwatch2: 1748508362657061 1281109; combined=39426, p1=34640, p2=3369, p3=182, p4=1053, p5=182, sr=0, sw=0, l=0, gc=0
Response-Body-Transformed: Dechunked
Producer: ModSecurity for Apache/2.9.6 (http://www.modsecurity.org/); OWASP_CRS/4.14.0.
Server: Apache/2.4.62 (Rocky Linux)
Engine-Mode: "ENABLED"

--e1cd7d3c-Z--
```


> ✅ This setup uses Apache as a reverse proxy load balancer, managed with Keepalived for high availability and protected with ModSecurity using the OWASP Core Rule Set (CRS).
