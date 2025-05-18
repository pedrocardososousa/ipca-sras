---
layout: single
title: "High Availability NFS Cluster"
permalink: /nfs-ha-cluster/
---

# 📁 High Availability NFS Cluster with DRBD and Pacemaker

This guide outlines how to deploy a High Availability (HA) NFS cluster using DRBD and Pacemaker on Rocky Linux to provide shared storage for WordPress containers.

---

## 🧱 Architecture Overview

```
                +----------------------+
                |     Virtual IP       |
                |   192.168.27.240     |
                +----------+-----------+
                           |
               +-----------+------------+
               |                        |
+------------------------+   +------------------------+
| NFS Server 1           |   | NFS Server 2           |
| 192.168.27.241         |   | 192.168.27.242         |
| DRBD + NFS + Pacemaker |   | DRBD + NFS + Pacemaker |
+------------------------+   +------------------------+
                           |
                  Shared NFS Exported Volume
```

---

## 🧰 Prerequisites

Install on both NFS servers:

```bash
sudo dnf install -y nfs-utils drbd drbd-utils pacemaker pcs corosync lvm2
sudo systemctl enable --now pcsd
sudo passwd hacluster
```

---

## ⚙️ DRBD Setup

Create `/etc/drbd.d/wordpress.res`:

```ini
resource wordpress {
  on nfs-node1 {
    device    /dev/drbd0;
    disk      /dev/sdb;
    address   192.168.27.241:7788;
    meta-disk internal;
  }
  on nfs-node2 {
    device    /dev/drbd0;
    disk      /dev/sdb;
    address   192.168.27.242:7788;
    meta-disk internal;
  }
}
```

Run on both:

```bash
drbdadm create-md wordpress
drbdadm up wordpress
```

On node1 only:

```bash
drbdadm -- --overwrite-data-of-peer primary wordpress
mkfs.xfs /dev/drbd0
mkdir /mnt/wordpress-data
mount /dev/drbd0 /mnt/wordpress-data
```

---

## 📦 NFS Export

Add to `/etc/exports`:

```bash
/mnt/wordpress-data 192.168.27.0/24(rw,sync,no_root_squash,no_subtree_check)
```

---

## 🧠 Pacemaker Setup

On node1:

```bash
pcs cluster auth nfs-node1 nfs-node2 -u hacluster -p yourpassword
pcs cluster setup --name nfs-cluster nfs-node1 nfs-node2
pcs cluster start --all
pcs property set stonith-enabled=false
```

Create cluster resources:

```bash
pcs resource create drbd_nfs ocf:linbit:drbd drbd_resource=wordpress op monitor interval=30s
pcs resource master master_nfs drbd_nfs master-max=1 master-node-max=1 clone-max=2 clone-node-max=1 notify=true

pcs resource create fs_nfs Filesystem device="/dev/drbd0" directory="/mnt/wordpress-data" fstype="xfs"
pcs resource create nfs-server systemd:nfs-server op monitor interval=30s
pcs resource create exportfs_nfs exportfs clientspec="192.168.27.0/24" options="rw,sync,no_root_squash" directory="/mnt/wordpress-data" fsid=0

pcs resource create vip_nfs IPaddr2 ip=192.168.27.240 cidr_netmask=24 nic=eth0

pcs constraint order start master_nfs then fs_nfs
pcs constraint order start fs_nfs then nfs-server
pcs constraint order start nfs-server then exportfs_nfs
pcs constraint order start exportfs_nfs then vip_nfs

pcs constraint colocation add fs_nfs with master master_nfs
pcs constraint colocation add nfs-server with fs_nfs
pcs constraint colocation add exportfs_nfs with nfs-server
pcs constraint colocation add vip_nfs with exportfs_nfs
```

---

## 🐳 Docker Integration (WordPress)

Example volume for Docker Compose:

```yaml
volumes:
  wp-content:
    driver: local
    driver_opts:
      type: "nfs"
      o: "addr=192.168.27.240,nfsvers=4,rw"
      device: ":/mnt/wordpress-data"
```

---

> 💡 Your WordPress containers now use a highly available, failover-protected NFS share backed by DRBD replication and Pacemaker-managed services.