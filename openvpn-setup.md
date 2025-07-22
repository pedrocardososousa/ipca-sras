# OpenVPN + FreeIPA Server Setup with MFA (OTP)

## Prerequisites

Install necessary packages:

```bash
dnf install openvpn easy-rsa sssd sssd-ldap sssd-tools authselect -y
dnf install ipa-client -y
```

## Test FreeIPA Ports

```bash
dnf install nc
nc -zv ipa01.cluster.local 389
nc -zv ipa01.cluster.local 88
nc -zv ipa01.cluster.local 80
nc -zv ipa01.cluster.local 464
nc -zv ipa01.cluster.local 123
```

## FreeIPA Client Installation

```bash
ipa-client-install
```

## Setup Easy-RSA

```bash
dnf install easy-rsa -y
dnf install nano -y
mkdir -p /etc/openvpn/easy-rsa
cp -r /usr/share/easy-rsa/* /etc/openvpn/easy-rsa/
cd /etc/openvpn/easy-rsa
```

Edit `vars` file:

```bash
set_var EASYRSA_REQ_COUNTRY    "PT"
set_var EASYRSA_REQ_PROVINCE   "Minho"
set_var EASYRSA_REQ_CITY       "Barcelos"
set_var EASYRSA_REQ_ORG        "IPCA"
set_var EASYRSA_REQ_EMAIL      "admin@cluster.local"
set_var EASYRSA_REQ_OU         "IT"
```

### Initialize the PKI and Build Certificates

```bash
./easyrsa init-pki
./easyrsa build-ca nopass
./easyrsa gen-req server nopass
./easyrsa sign-req server server
./easyrsa gen-dh
openvpn --genkey --secret ta.key
```

### Move Files to OpenVPN Directory

```bash
cp pki/ca.crt /etc/openvpn/
cp pki/issued/server.crt /etc/openvpn/
cp pki/private/server.key /etc/openvpn/
cp pki/dh.pem /etc/openvpn/
cp ta.key /etc/openvpn/
```

## PAM Configuration for OpenVPN

Edit `/etc/pam.d/openvpn`:

```bash
auth	required	pam_sss.so forward_pass
account	required	pam_sss.so
```

## SSSD Configuration

Edit `/etc/sssd/sssd.conf`:

```ini
[sssd]
domains = cluster.local
services = nss, pam
config_file_version = 2

[domain/cluster.local]
id_provider = ipa
auth_provider = ipa
chpass_provider = ipa
ipa_domain = cluster.local
ipa_server = _srv_, ipa01.cluster.local
ipa_hostname = ovpn01.cluster.local
ipa_otp = true
krb5_use_kdcinfo = false
cache_credentials = true
krb5_use_fast = try
dns_discovery_domain = cluster.local
enumerate = false

[pam]
debug_level = 9
```

## OpenVPN Server Configuration

Edit `/etc/openvpn/server/server.conf`:

```conf
port 443
proto tcp
dev tun
float
tun-mtu 1450
mssfix 1410

ca /etc/openvpn/ca.crt
cert /etc/openvpn/server.crt
key /etc/openvpn/server.key
dh /etc/openvpn/dh.pem

server 10.8.0.0 255.255.255.0
ifconfig-pool-persist /var/log/openvpn/ipp.txt
push "route 172.16.24.0 255.255.255.0"

plugin /usr/lib64/openvpn/plugins/openvpn-plugin-auth-pam.so openvpn
verify-client-cert none
username-as-common-name

tls-crypt /etc/openvpn/ta.key 0
cipher AES-256-GCM
auth SHA256
data-ciphers AES-256-GCM:AES-128-GCM:AES-256-CBC:AES-128-CBC
tls-server

log-append /var/log/openvpn/server.log
verb 7
status /var/log/openvpn/openvpn-status.log

keepalive 10 120
persist-key
persist-tun
user nobody
group nobody

auth-retry interact
```

Set permissions:

```bash
chown root:root /etc/openvpn/server/server.conf
chmod 600 /etc/openvpn/server/server.conf
mkdir -p /var/log/openvpn
chown openvpn:openvpn /var/log/openvpn
```

## Firewall Configuration

```bash
firewall-cmd --permanent --add-port=1194/udp
firewall-cmd --permanent --add-port=443/udp
firewall-cmd --permanent --add-port=443/tcp
firewall-cmd --reload
```

## Enable OpenVPN Service

```bash
systemctl enable --now openvpn-server@server
```

## Keepalived Configuration (Optional HA)

```bash
dnf install keepalived -y
mv /etc/keepalived/keepalived.conf /etc/keepalived/keepalived.conf.ori
```

Edit `/etc/keepalived/keepalived.conf`:

```conf
vrrp_instance VI_DB {
    state MASTER
    interface ens33
    virtual_router_id 60
    priority 100
    advert_int 1

    authentication {
        auth_type PASS
        auth_pass Pass12345
    }

    virtual_ipaddress {
        172.16.24.1
    }
}
```

Start Keepalived:

```bash
systemctl enable --now keepalived
```

---

**Author:** António Machado, José Rosa e Pedro Sousa  
**Organization:** IPCA  
**Purpose:** VPN with FreeIPA + MFA integration