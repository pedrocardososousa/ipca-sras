---
layout: single
title: "GlusterFS Cluster for WordPress"
permalink: /glusterfs-cluster/
---

# 🗂️ GlusterFS Cluster for WordPress Shared Storage

This guide outlines the configuration of a two-node GlusterFS cluster to provide highly available shared storage for WordPress containers.

---

## 🧱 Cluster Overview

- **GlusterFS Node 1**: `192.168.27.241`
- **GlusterFS Node 2**: `192.168.27.242`
- **GlusterFS Node 3**: `192.168.27.243`
- **Volume Name**: `volume1`
- **Replica Count**: 3
- **Mount Point on Clients**: `/mnt/wordpress-data`

---

## 📌 Architecture Diagram

```
                      +---------------------+
                       |   WordPress Server  |
                       |   192.168.27.211    |
                       +---------------------+
                                  |      
                 Mount /mnt/wordpress-data via GlusterFS
                                  |
    +-------------------+-------------------+-------------------+
    |   Gluster Node 1  |   Gluster Node 2  |   Gluster Node 3  |
    |   192.168.27.241  |   192.168.27.242  |   192.168.27.243  |
    +-------------------+-------------------+-------------------+
              |                   |                   |        
            +------------ Replicated Volume ------------+
```

---

## ⚙️ Installation Steps (on Both GlusterFS Nodes)

## 🧰 Prerequisites

We will create a new LVM logical volume that will be mounted on /data/glusterfs/vol0 on both of the cluster's servers:
```bash
dnf install lvm2 -y
```
```bash
pvcreate /dev/nvme0n2
vgcreate vg_data /dev/nvme0n2
lvcreate -l 100%FREE -n lv_data vg_data
mkfs.xfs /dev/vg_data/lv_data
mkdir -p /data/glusterfs/volume1
```

We can now add that logical volume to the ``/etc/fstab`` file:
```ini
/dev/mapper/vg_data-lv_data /data/glusterfs/volume1        xfs     defaults        1 2
```

And mount it:
```bash
mount -a
```

As the data is stored in a sub-volume called brick, we can create a directory in this new data space dedicated to it:
For node01
```bash
mkdir /data/glusterfs/volume1/brick1
```

For node02
```bash
mkdir /data/glusterfs/volume1/brick2
```

For node03
```bash
mkdir /data/glusterfs/volume1/brick3
```

### 1. Install GlusterFS

```bash
dnf install centos-release-gluster9 -y
```
```bash
dnf install glusterfs glusterfs-libs glusterfs-server -y
```

```bash
firewall-cmd --zone=public --add-service=glusterfs --permanent
firewall-cmd --reload
```

vi /etc/hosts
```ini
192.168.27.241 nfs01 nfs01.cluster.local
192.168.27.242 nfs02 nfs02.cluster.local
192.168.27.243 nfs03 nfs03.cluster.local
```
# Check SELinux
```bash
getsebool -a | grep -i gluster
/usr/sbin/setsebool gluster_anon_write 1
/usr/sbin/setsebool gluster_export_all_ro 1
/usr/sbin/setsebool gluster_export_all_rw 1
/usr/sbin/setsebool gluster_use_execmem 1
sestatus
```

```bash
systemctl enable glusterfsd.service glusterd.service
systemctl start glusterfsd.service glusterd.service
```

---

### 2. Peer the Nodes

On one node:
```bash
gluster peer probe nfs02.cluster.local
gluster peer probe nfs03.cluster.local
gluster peer status
```

---

### 3. Create the Volume

```bash
gluster volume create volume1 replica 3 nfs01.cluster.local:/data/glusterfs/volume1/brick1/ nfs02.cluster.local:/data/glusterfs/volume1/brick2/ nfs03.cluster.local:/data/glusterfs/volume1/brick3/ force
```
Replica 2 volumes are prone to split-brain. Use Arbiter or Replica 3 to avoid this. See: https://docs.gluster.org/en/latest/Administrator-Guide/Split-brain-and-ways-to-deal-with-it/.
Do you still want to continue?
 (y/n) y
volume create: volume1: success: please start the volume to access data

```bash
gluster volume start volume1
gluster volume status
gluster volume info
```

```bash
gluster volume set volume1 auth.allow 192.168.27.211,192.168.27.212
gluster volume get volume1 auth.allow
```

---

## 📦 Mount Gluster Volume on WordPress Servers

### Install Gluster Client

```bash
dnf install glusterfs-client
```

### Mount the Volume

```bash
mkdir -p /mnt/wordpress-data
mount.glusterfs nfs01.cluster.local:/volume1 /mnt/wordpress-data
touch /mnt/wordpress-data/test
```

### Add to `/etc/fstab` for persistence

```fstab
192.168.27.241:/volume1 /mnt/wordpress-data glusterfs defaults,_netdev 0 0
```

---

## 🐳 Docker Compose Integration (Bind Mount)

Use GlusterFS volume as bind mount:

```yaml
volumes:
  wp-content:
    driver: local
    driver_opts:
      type: "none"
      o: "bind"
      device: "/mnt/wordpress-data"
```

---

## 🚀 Extra commands

```bash
gluster volume set volume1 cluster.quorum-reads false
gluster volume set volume1 cluster.quorum-count 1
gluster volume heal volume1 enable
gluster volume status
```
---

## ✅ Notes

- Consider using a VIP with Keepalived for better mount availability
- You can scale the volume later using more bricks
- Monitor Gluster with: `gluster volume status` and `gluster volume info`

> 💡 This setup removes the NFS layer and simplifies management while retaining redundancy and scalability.