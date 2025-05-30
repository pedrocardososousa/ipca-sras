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

---

## ⚙️ Configuration Steps

### 0. Deploy from Template Server
Set interface and hostname
```bash
sudo su
nmtui
```
Basic information:
Interface settings: `Manual`
IP Address: `192.168.27.231/24`
Gateway: `192.168.27.2`
DNS: `192.168.27.2`
IPV6: `Disable`
Hostname: `db01`

Update system
```bash
dnf update -y
```

### 0. 🔐 Security Options
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
```

```bash
semanage port -a -t mysqld_port_t -p tcp 3306
semanage port -a -t mysqld_port_t -p tcp 4567
semanage port -a -t mysqld_port_t -p tcp 4568
semanage port -a -t mysqld_port_t -p tcp 4444
semanage port -a -t mysqld_port_t -p udp 4567
semanage permissive -a mysqld_t
```

```bash
/usr/sbin/setsebool mysql_connect_any 1
/usr/sbin/setsebool mysql_connect_http 1
/usr/sbin/setsebool selinuxuser_mysql_connect_enabled 1
```
# Check SELinux status
```bash
sestatus
```

### 1. Install MariaDB and Galera

```bash
#dnf install mariadb-server galera -y
#systemctl enable mariadb

curl -LsS https://r.mariadb.com/downloads/mariadb_repo_setup | bash

#import rpm keys
rpm --import https://supplychain.mariadb.com/MariaDB-Server-GPG-KEY

#You can verify that the repositories have been added to the server by running:
dnf repolist

dnf install MariaDB-server MariaDB-client galera -y

rpm -qa | grep -i mariadb
rpm -qa | grep -i galera

systemctl start mariadb
systemctl status mariadb

#Setup secure installation for MariaDB
mariadb-secure-installation

#Switch to UNIX socket authentication: 
#If prompted, choosing 'yes' will use the Unix authentication plugin for connecting to the database server. 
#However, the recommended authentication plugin is mysql_native_password. 
#Therefore, you should enter 'n' when asked.

#Change the root password? [Y/n]: Enter Y to set a strong password for the database root user.
#Remove anonymous users? [Y/n]: Enter Y to remove anonymous access to the database server.
#Disallow root login remotely? [Y/n]: Enter Y to disallow root login remotely, which enhances security.
#Remove test database and access to it? [Y/n]: Enter Y to delete any unwanted test databases from the server.
#Reload privilege tables now? [Y/n]: Enter Y to reload the privilege tables and apply the changes immediately to the database.
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

On the all nodes:

```bash
systemctl stop mariadb
```

On the **first node only**:

```bash
galera_new_cluster
```

On the all nodes:

```bash
systemctl start mariadb
```

Verify with:

```bash
mariadb -e "SHOW STATUS LIKE 'wsrep_cluster_size';"
mariadb -e "SHOW STATUS LIKE 'wsrep_incoming_addresses';"
mariadb -e "SHOW STATUS LIKE 'wsrep_cluster_status';"
mariadb -e "SHOW STATUS LIKE 'wsrep_cluster_size';"
```

You should see `3`.

---

### 3. Configure Keepalived (for VIP)

On **each DB node**, install:

```bash
dnf install keepalived -y
```

```bash
mv /etc/keepalived/keepalived.conf /etc/keepalived/keepalived.conf.ori
vi /etc/keepalived/keepalived.conf
```
Example `/etc/keepalived/keepalived.conf` for Node 1 (MASTER):

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
        192.168.27.230
    }
}
```

Change `state` and `priority` for other nodes (`BACKUP`, `90`, `80`, etc.).

Enable Keepalived:

```bash
systemctl enable --now keepalived
```

Test failover by stopping keepalived on one node:

```bash
systemctl stop keepalived
```

Then ping `192.168.27.230` — it should still respond from another node.

---

## 🧭 Connection

Applications (e.g., WordPress containers) connect to:

```env
DB_HOST=192.168.27.230
DB_PORT=3306
```

No need to worry about which node is online — Keepalived handles the routing.

---

## 🧭 Safe Bootstrap

```bash
cd /var/lib/mysql/
vi grastate.dat
```

```ini
 GALERA saved state
version: 2.1
uuid:    9e847de8-3ba8-11f0-811f-8709d50fadfd
seqno:   -1
safe_to_bootstrap: 1
```

## 🔍 Test Categories
### 1. ✅ Basic Cluster Health Tests

```bash
mariadb -e "SHOW STATUS LIKE 'wsrep_cluster_size';"
mariadb -e "SHOW STATUS LIKE 'wsrep_incoming_addresses';"
mariadb -e "SHOW STATUS LIKE 'wsrep_cluster_status';"
mariadb -e "SHOW STATUS LIKE 'wsrep_cluster_size';"
```

### 2. ✅ Read/Write Consistency Tests
A. Write to one node, read from others
On Node 1:
```bash
mariadb
```

```sql
CREATE DATABASE galera_test;
CREATE TABLE galera_test.users (id INT PRIMARY KEY, name VARCHAR(50));
INSERT INTO galera_test.users VALUES (1, 'Alice');
```

On Node 2 or 2:
```bash
mariadb
```

```sql
SELECT * FROM galera_test.users;
```

Opcional
```sql
DROP DATABASE galera_test;
```

✅ Expected: Data is visible immediately on all nodes.

### 3. 🔄 Network Partition / Node Failure
A. Hard Shutdown One Node
```bash
shutdown -h now  # or power off one node forcibly
```

```bash
mariadb -e "SHOW STATUS LIKE 'wsrep_cluster_size';"
mariadb -e "SHOW STATUS LIKE 'wsrep_incoming_addresses';"
mariadb -e "SHOW STATUS LIKE 'wsrep_cluster_status';"
mariadb -e "SHOW STATUS LIKE 'wsrep_cluster_size';"
```

✅ Expected:
 - Cluster size decreases.
 - Remaining nodes still accept reads/writes.

B. Restart the node

```bash
mariadb -e "SHOW STATUS LIKE 'wsrep_cluster_size';"
mariadb -e "SHOW STATUS LIKE 'wsrep_incoming_addresses';"
mariadb -e "SHOW STATUS LIKE 'wsrep_cluster_status';"
mariadb -e "SHOW STATUS LIKE 'wsrep_cluster_size';"
```

✅ Expected:
 - Node rejoins cluster automatically.
 - wsrep_cluster_size increases.

### 4. MariaDB running user

Check running user
```bash
ps -eo user,comm | grep mariadb
```

Output example:
```
mysql    mariadbd
```

> 💡 *MariaDB Galera Cluster ensures zero-downtime database replication with automatic failover — a perfect fit for high-availability WordPress hosting.*