---
layout: single
title: "WordPress Docker Deployment with GlusterFS"
permalink: /wordpress-docker/
---

# 🐳 WordPress Deployment using Docker and GlusterFS

This guide covers setting up WordPress containers on Rocky Linux using Docker and mounting shared storage from a GlusterFS cluster.

---

## 📦 System Preparation

```bash
su
dnf update -y
dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
```

---

## 🐳 Docker Installation

```bash
dnf install -y docker-ce docker-ce-cli containerd.io
systemctl start docker
systemctl status docker
systemctl enable docker
```

---

## 🔥 Firewall Configuration

```bash
firewall-cmd --zone=public --add-port=80/tcp --permanent
firewall-cmd --reload
```

---

## 🔗 GlusterFS Client Setup

```bash
dnf install -y glusterfs-client
mkdir -p /mnt/wordpress-data
```

### 🧭 Update /etc/hosts

```bash
vi /etc/hosts
```

Add:

```
192.168.27.241 nfs01 nfs01.cluster.local
192.168.27.242 nfs02 nfs02.cluster.local
```

### 🔗 Mount GlusterFS Volume

```bash
mount.glusterfs nfs01.cluster.local:/volume1 /mnt/wordpress-data
```

### 📌 Make it Persistent

Edit `/etc/fstab`:

```fstab
192.168.27.241:/volume1 /mnt/wordpress-data glusterfs defaults,_netdev 0 0
```

---

## 🗄️ Create WordPress Database (On DB Server)

```sql
CREATE DATABASE IPCA;
GRANT ALL PRIVILEGES ON IPCA.* TO 'ipca'@'%' IDENTIFIED BY 'wordPress!ipca';
FLUSH PRIVILEGES;
```

---

## 📁 Docker Compose File
```bash
docker volume create --driver local --opt type=none --opt o=bind --opt device=/mnt/wordpress-data wp-content
```

Create `compose.yaml`:

```yaml
services:
  wordpress:
    image: wordpress
    restart: always
    ports:
      - 80:80
    environment:
      WORDPRESS_DB_HOST: 192.168.27.230
      WORDPRESS_DB_USER: ipca
      WORDPRESS_DB_PASSWORD: wordPress!ipca
      WORDPRESS_DB_NAME: IPCA
    volumes:
      - wp-content:/var/www/html

volumes:
  wp-content:
#    driver: local
#    driver_opts:
#      type: none
#      device: /mnt/wordpress-data
#      o: bind
```

---

## 🚀 Deploy WordPress

```bash
docker compose up -d
```

Check if volume is mounted:

```bash
ls /mnt/wordpress-data
```

---

> ✅ This setup ensures WordPress uses shared storage from a GlusterFS cluster for persistent file storage across multiple containers or nodes.