# Kubernetes HA Load Balancer Setup Documentation

## 1️⃣ Overview

This document describes the configuration of a **highly available (HA) load balancer** in front of the Kubernetes control plane, using **HAProxy + Keepalived**. The purpose is to provide a **single, highly available API endpoint** for the cluster, with automatic failover between LB nodes.

* **Cluster Name:** `brd-hq-cluster`
* **Cluster Domain:** `brd-hq-cluster.brd.rw`
* **Load Balancer VIP:** `192.168.10.177`
* **HAProxy Nodes:** `node1` (MASTER), `node2` (BACKUP)
* **Control Plane Nodes:** `node1`, `node2`, `node3`
* **Worker Nodes:** `node4`, `node5`, `node6`
* **Inventory Path:** `inventory/brd-hq-cluster/hosts.yaml`
* **Group Vars Path:** `inventory/brd-hq-cluster/group_vars/k8s_cluster/k8s-cluster.yml`

---

## 2️⃣ Components

### 2.1 HAProxy

* Acts as the **reverse proxy** and load balancer for the Kubernetes API servers.
* Balances traffic between all control plane nodes (`node1`, `node2`, `node3`).
* Listens on port `6443` on the **VIP**.

### 2.2 Keepalived

* Provides **high availability** for the VIP.
* MASTER node owns the VIP, BACKUP monitors it.
* If MASTER fails, BACKUP takes over the VIP automatically.
* Communicates between nodes using **VRRP** (Virtual Router Redundancy Protocol).
* Secured with a password (`haproxy_keepalived_password`).

---

## 3️⃣ Inventory Configuration

### 3.1 Hosts

```yaml
lb:
  hosts:
    node1:   # MASTER
    node2:   # BACKUP

kube_control_plane:
  hosts:
    node1:
    node2:
    node3:

kube_node:
  hosts:
    node4:
    node5:
    node6:
```

---

### 3.2 Group Vars (`k8s-cluster.yml`)

```yaml
# Cluster domain
cluster_name: brd-hq-cluster.brd.rw
dns_domain: "{{ cluster_name }}"

# HAProxy settings
apiserver_loadbalancer_domain_name: brd-hq-cluster.brd.rw
apiserver_loadbalancer_port: 6443
kube_apiserver_loadbalancer: haproxy

# VIP for HAProxy
haproxy_apiserver_vip: 192.168.10.177

# Keepalived settings
haproxy_keepalived_enabled: true
haproxy_keepalived_interface: ens160    # network interface used for VIP
haproxy_keepalived_priority_master: 101
haproxy_keepalived_priority_backup: 100
haproxy_keepalived_password: "yourpassword"
```

* Replace `ens160` with the actual network interface used by your LB nodes.
* Use a secure, shared password for Keepalived authentication.

---

## 4️⃣ VIP Behavior

* The VIP (`192.168.10.177`) is **virtual**. It is not manually configured on nodes.
* Keepalived assigns it dynamically to the MASTER LB node.
* Failover: If MASTER fails, BACKUP automatically takes over.

---

## 5️⃣ Running the Deployment

From Kubespray root directory:

```bash
ansible-playbook -i inventory/brd-hq-cluster/hosts.yaml cluster.yml -b -v
```

* `-b` → run tasks with sudo
* `-v` → verbose output

### Verification

```bash
# Verify nodes and cluster
kubectl get nodes
kubectl cluster-info

# Check VIP assignment
ip addr show ens160 | grep 192.168.10.177

# Test API access via VIP
curl -k https://192.168.10.177:6443/version
```

---

## 6️⃣ Key Points

* **HA API endpoint**: all clients should use `https://192.168.10.177:6443` or `https://brd-hq-cluster.brd.rw:6443`.
* **MASTER node priority**: higher number = preferred VIP owner.
* **Backup node**: lower number, takes over if MASTER fails.
* **Interface selection**: VIP must be on the same subnet as cluster nodes.

---

## 7️⃣ Notes & Best Practices

1. Ensure the **VIP is reserved** in your network to avoid IP conflicts.
2. Keep **all HAProxy nodes in the same subnet** as the VIP.
3. Keep the **Keepalived password secret**.
4. For production, monitor HAProxy and Keepalived logs for failover events.
5. All cluster API traffic should go through the VIP to ensure HA.

<img width="1536" height="1024" alt="image" src="https://github.com/user-attachments/assets/aa4740b6-bdef-43a2-a07a-dfb022365e62" />

