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

