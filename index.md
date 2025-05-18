# High Availability Web Hosting Infrastructure with Rocky Linux

Welcome to the documentation of my self-hosted infrastructure built for high availability, scalability, and reliability. This environment hosts WordPress websites using a layered architecture based on open-source technologies.

---

## 🔧 Architecture Overview

This setup consists of:

- **2 Load Balancer Servers**  
  Running Apache HTTP Server with `mod_proxy_balancer` and `keepalived` for failover support.

- **2 Application Servers (WordPress)**  
  Running Dockerized WordPress containers with persistent storage and isolated configurations.

- **3 Database Servers (MariaDB Cluster)**  
  Configured in a Galera Cluster for data redundancy and horizontal scaling.

- **1 Template Server**  
  A pre-hardened Rocky Linux base image used to quickly deploy new servers with consistent security and network settings.

---

## 🖧 Network Topology

      +----------------+
      | Virtual IP     |
      | 192.168.27.220 |
      +-------+--------+
              |
    +---------+---------+
    |                   |
+---------------+ +---------------+
| Proxy Server 1| | Proxy Server 2|
| 192.168.27.221| | 192.168.27.222|
+---------------+ +---------------+
| |
+---------+---------+
|
+---------+---------+
| |
+---------------+ +---------------+
| WordPress 1 | | WordPress 2 |
| 192.168.27.211| | 192.168.27.212|
+---------------+ +---------------+
| |
+---------+---------+
|
+---------+---------+---------+
| | | |
+---------------+ +---------------+ +---------------+
| MariaDB Node 1| | MariaDB Node 2| | MariaDB Node 3|
| 192.168.27.231| | 192.168.27.232| | 192.168.27.233|
+---------------+ +---------------+ +---------------+



---

## 📦 Components

### 🔹 Load Balancer Layer
- Apache HTTP Server
- `mod_proxy`, `mod_proxy_balancer`, `mod_status`
- Keepalived for VRRP-based failover

### 🔹 Application Layer
- WordPress inside Docker containers
- Docker volumes for persistent content
- Environment isolation for scaling

### 🔹 Database Layer
- MariaDB with Galera cluster replication
- Synchronous multi-master setup
- Automatic failover and data redundancy

### 🔹 Base Template Server
- Rocky Linux base image
- Pre-configured with:
  - SELinux and firewall rules
  - SSH hardening
  - Automatic package updates
  - System baseline tools

---

## 📚 Documentation

Check the sidebar for:

- [Load Balancer Setup](./load-balancer.html)
- [Docker WordPress Deployment](./wordpress-docker.html)
- [Galera Cluster Configuration](./galera-cluster.html)
- [Template Server & Security](./template-server.html)
- [Monitoring & Maintenance](./monitoring.html)

---

## 🧭 Goals

- High availability through redundancy
- Scalable application and database tiers
- Ease of deployment and management
- Open-source and secure stack

---

> 📘 *This site is a live documentation project. Contributions, suggestions, and feedback are welcome!*

