# kubespray-setup

## **Step 1: Prepare your admin machine**

1. **Install git and Python dependencies**

```bash
sudo apt update
sudo apt install -y git python3-pip python3-venv
```

2. **Clone the Kubespray repo**

```bash
git clone https://github.com/kubernetes-sigs/kubespray.git
cd kubespray
```

3. **Install required Python packages**

```bash
pip3 install -r requirements.txt
```

4. **Check your shell** (usually bash or zsh):

   ```bash
   echo $SHELL
   ```

5. **Add `~/.local/bin` to your PATH**:
   For **bash**, run:

   ```bash
   echo 'export PATH=$HOME/.local/bin:$PATH' >> ~/.bashrc
   source ~/.bashrc
   ```

   For **zsh**, run:

   ```bash
   echo 'export PATH=$HOME/.local/bin:$PATH' >> ~/.zshrc
   source ~/.zshrc
   ```

6. **Verify** it worked:

   ```bash
   echo $PATH
   which ansible
   ```

**Explanation:**

* `git clone` gets Kubespray code.
* `requirements.txt` installs Python packages that Ansible needs to run playbooks.

---

## **Step 2: Create inventory for your cluster**

Kubespray comes with a **sample inventory**. We copy it and then edit it for your 6 VMs.

1. **Copy the sample inventory**

```bash
cp -rfp inventory/sample inventory/brd-hq-cluster
```

* This creates a folder `inventory/brd-hq-cluster` which will hold your cluster configuration.

2. **List your VM IPs**
   For example, if your VMs have these IPs:

* Control planes: `xxx.xxx.xxx.xxx`, `xxx.xxx.xxx.xxx`, `xxx.xxx.xxx.xxx`
* Workers: `yyy.yyy.yyy.yyy`, `yyy.yyy.yyy.yyy`, `yyy.yyy.yyy.yyy`

3. **Build the inventory**

```bash
  nano inventory/brd-hq/hosts.yaml
```
```bash
  all:
    hosts:
      node1:
        ansible_host: 192.168.1.11
        ip: 192.168.1.11
        access_ip: 192.168.1.11
      node2:
        ansible_host: 192.168.1.12
        ip: 192.168.1.12
        access_ip: 192.168.1.12
      node3:
        ansible_host: 192.168.1.13
        ip: 192.168.1.13
        access_ip: 192.168.1.13
      node4:
        ansible_host: 192.168.1.14
        ip: 192.168.1.14
        access_ip: 192.168.1.14
      node5:
        ansible_host: 192.168.1.15
        ip: 192.168.1.15
        access_ip: 192.168.1.15
      node6:
        ansible_host: 192.168.1.16
        ip: 192.168.1.16
        access_ip: 192.168.1.16
  
    children:
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
      etcd:
        hosts:
          node1:
          node2:
          node3:
      k8s_cluster:
        children:
          kube_control_plane:
          kube_node:
      calico_rr:
        hosts: {}
```
---

Perfect! ✅ Your inventory is ready and SSH connectivity is confirmed.

Now we can move to **Step 4: Deploy the cluster using Kubespray**.

---

### **Step 4: Deploy Kubernetes**

1. **Run the Ansible playbook** from the Kubespray root folder:

```bash
ansible-playbook -i inventory/brd-hq/hosts.yaml cluster.yml -b -v
```

**Explanation:**

* `-i inventory/brd-hq/hosts.yaml` → tells Ansible which nodes to use
* `cluster.yml` → the main Kubespray playbook that installs Kubernetes
* `-b` → run with sudo
* `-v` → verbose output (useful for debugging)

2. **What happens during deployment**:

* Installs Docker/containerd on all nodes
* Sets up kubeadm and Kubernetes components
* Configures etcd on control planes
* Deploys networking (Calico by default)
* Marks worker nodes ready to run pods

3. **Time required:**

* For 6 nodes, expect **15–30 minutes**, depending on your VM specs and network.

---

### **Step 5: Verify the cluster**

After the playbook finishes:

Copy kubeconfig to your user

```bash
mkdir -p ~/.kube
sudo cp /etc/kubernetes/admin.conf ~/.kube/config
sudo chown $(id -u):$(id -g) ~/.kube/config
```

```bash
kubectl get nodes
kubectl get pods -A
```

```bash
kubectl label node node4 node-role.kubernetes.io/worker=worker
kubectl label node node5 node-role.kubernetes.io/worker=worker
kubectl label node node6 node-role.kubernetes.io/worker=worker
```

You should see:

* 3 control plane nodes `Ready`
* 3 worker nodes `Ready`
* System pods running in `kube-system` namespace

---

💡 **Tip:**

* Keep an eye on Ansible output for any failed tasks.
* If something fails, you can re-run the playbook; it’s mostly **idempotent**.

---


