# OpenVPN Installation and Configuration Guide for RHEL-based Systems

## Install OpenVPN

### Step 1: Enable EPEL Repository
```bash
dnf install epel-release -y
```

### Step 2: Install OpenVPN
```bash
dnf install openvpn -y
```

---

## Set up Certificate Authority

### Step 1: Install easy-rsa
```bash
dnf install easy-rsa -y
```

### Step 2: Prepare easy-rsa Directory
```bash
mkdir /etc/openvpn/easy-rsa
ln -s /usr/share/easy-rsa /etc/openvpn/easy-rsa
cd /etc/openvpn/easy-rsa
```

### Step 3: Initialize the PKI
```bash
./easy-rsa/3/easyrsa init-pki
```

### Step 4: Build the CA (No Password)
```bash
./easy-rsa/3/easyrsa build-ca nopass
```

---

## Create Certificates

### Step 1: Generate Server Certificate
```bash
./easy-rsa/3/easyrsa gen-req server nopass
./easy-rsa/3/easyrsa sign-req server server
```

### Step 2: Generate Client Certificate (Repeat as Needed)
```bash
./easy-rsa/3/easyrsa gen-req client1 nopass
./easy-rsa/3/easyrsa sign-req client client1
```

### Step 3: Generate Diffie-Hellman Parameters
```bash
./easy-rsa/3/easyrsa gen-dh
```

---

## Configure OpenVPN

### Step 1: Copy Sample Config
```bash
cp /usr/share/doc/openvpn/sample/sample-config-files/server.conf /etc/openvpn
```

### Step 2: Edit Configuration
```bash
vi /etc/openvpn/server.conf
```

#### Lines to Modify:
```ini
ca /etc/openvpn/easy-rsa/pki/ca.crt
cert /etc/openvpn/easy-rsa/pki/issued/server.crt
key /etc/openvpn/easy-rsa/pki/private/server.key  # This file should be kept secret
dh /etc/openvpn/easy-rsa/pki/dh.pem
```

#### Optional TLS-auth (SSL used by default, disable if not used):
Comment out the following line:
```ini
#tls-auth ta.key 0 # This file is secret
```

Save and exit the editor.

---

## Configure Firewall

### Step 1: Open OpenVPN Port
```bash
firewall-cmd --add-service=openvpn --permanent
firewall-cmd --add-masquerade --permanent
firewall-cmd --reload
```

---

## Configure Routing

### Enable IP Forwarding
```bash
sysctl -w net.ipv4.ip_forward=1
```

---

## Start OpenVPN Server

### Recommended for Initial Testing
```bash
openvpn /etc/openvpn/server.conf
```

> OpenVPN should now be running and ready to accept client connections.
