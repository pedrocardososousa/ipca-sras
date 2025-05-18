---
layout: single
title: "MariaDB Galera Cluster"
permalink: /galera-cluster/
---

# 🛢️ MariaDB Galera Cluster

This page documents the setup and configuration of a **3-node Galera cluster** running on Rocky Linux, with a shared **Virtual IP (VIP)** for high availability access.

---

## 🧱 Architecture

The Galera cluster provides **multi-master synchronous replication**, ensuring that all three nodes stay in sync and that one failure does not interrupt service.

```
                     +-------------------+
                     |  DB VIP           |
                     |  192.168.27.230   |
                     +--------+----------+
                              |
    +-------------------------+--------------------+
    |                         |                    |
+----------------+   +----------------+   +----------------+
| Node 1         |   | Node 2         |   | Node 3         |
| 192.168.27.231 |   | 192.168.27.232 |   | 192.168.27.233 |
| MariaDB        |   | MariaDB        |   | MariaDB        |
+----------------+   +----------------+   +----------------+
```

---

## 🧪 Requirements

- Rocky Linux 8 or 9
- MariaDB 10.5+ with Galera support
- Keepalived for VIP management
- SELinux enabled (adjust ports if needed)
- Passwordless SSH between nodes (optional for automation)

---

## ⚙️ Configuration Steps

### 0. Deploy from Template Server
```bash
sudo su
nmtui
hostnamectl set-hostname db01
dnf update -y
```

Open ports using firewalld
```bash
firewall-cmd --zone=public --add-service=mysql --permanent
firewall-cmd --zone=public --add-port=3306/tcp --permanent
firewall-cmd --zone=public --add-port=4567/tcp --permanent
firewall-cmd --zone=public --add-port=4568/tcp --permanent
firewall-cmd --zone=public --add-port=4444/tcp --permanent
firewall-cmd --zone=public --add-port=4006/tcp --permanent
firewall-cmd --zone=public --add-port=4008/tcp --permanent
firewall-cmd --zone=public --add-port=4567/udp --permanent
firewall-cmd --reload
```

Adjust SELinux to allow MariaDB and Galera to run
```bash
dnf install policycoreutils-python-utils
semanage port -a -t mysqld_port_t -p tcp 3306
semanage port -a -t mysqld_port_t -p tcp 4567
semanage port -a -t mysqld_port_t -p tcp 4568
semanage port -a -t mysqld_port_t -p tcp 4444
semanage port -a -t mysqld_port_t -p udp 4567
semanage permissive -a mysqld_t
```


### 1. Install MariaDB and Galera

```bash
sudo dnf install mariadb-server galera -y
sudo systemctl enable mariadb
```

Edit `/etc/my.cnf.d/galera.cnf` on all nodes:

```ini
[mysqld]
binlog_format=ROW 
default_storage_engine=innodb
innodb_autoinc_lock_mode=2
bind-address=0.0.0.0
max_connections=1000 # Adjust based on expected load
innodb_buffer_pool_size=1G  # Adjust based on available memory (recommended to set it to ~70-80% of available RAM)

# Galera Cluster settings
wsrep_on=ON # galera replication on
wsrep_provider=/usr/lib64/galera-4/libgalera_smm.so  #galera library file path. 
wsrep_cluster_name=cluster_poc 
wsrep_cluster_address="gcomm://192.168.27.231,192.168.27.232,192.168.27.233" #allserverdetails
wsrep_sst_method=rsync #data sync method.
wsrep_sst_auth="root:abcd@1234"
wsrep_node_address="192.168.27.231" # IP address of the node on which this configuration file resides.
wsrep_node_name="db01" # hostname of the node on which this configuration file resides
```

Update `wsrep_node_name` and `wsrep_node_address` on each node accordingly.

---

### 2. Initialize the Cluster

On the **first node only**:

```bash
galera_new_cluster
```

On the other nodes:

```bash
sudo systemctl start mariadb
```

Verify with:

```bash
mysql -e "SHOW STATUS LIKE 'wsrep_cluster_size';"
```

You should see `3`.

---

### 3. Configure Keepalived (for VIP)

On **each DB node**, install:

```bash
sudo dnf install keepalived -y
```

Example `/etc/keepalived/keepalived.conf` for Node 1 (MASTER):

```ini
vrrp_instance VI_DB {
    state MASTER
    interface eth0
    virtual_router_id 60
    priority 100
    advert_int 1

    authentication {
        auth_type PASS
        auth_pass secretpass
    }

    virtual_ipaddress {
        192.168.27.230
    }
}
```

Change `state` and `priority` for other nodes (`BACKUP`, `90`, `80`, etc.).

Enable Keepalived:

```bash
sudo systemctl enable --now keepalived
```

Test failover by stopping keepalived on one node:

```bash
sudo systemctl stop keepalived
```

Then ping `192.168.27.230` — it should still respond from another node.

---

## 🔐 Security Notes

- Use firewalld to allow port `3306`, and Galera ports `4567`, `4568`, `4444`
- Configure SELinux to allow MySQL port access:

```bash
sudo semanage port -a -t mysqld_port_t -p tcp 4567
```

---

## 🧭 Connection

Applications (e.g., WordPress containers) connect to:

```env
DB_HOST=192.168.27.230
DB_PORT=3306
```

No need to worry about which node is online — Keepalived handles the routing.

---

## ✅ Best Practices

- Use `rsync` for SST (simpler than xtrabackup)
- Monitor `wsrep_cluster_size` for health
- Backup individual nodes regularly
- Avoid large transactions; Galera replicates synchronously

---

> 💡 *MariaDB Galera Cluster ensures zero-downtime database replication with automatic failover — a perfect fit for high-availability WordPress hosting.*