# Ansible Configuration and Kubernetes Setup Analysis

This setup provides a production-ready Kubernetes cluster with:

- Systemd cgroup driver
- GPU support where available
- Proper kernel tuning
- Verified component versions

---

1. Prepare Your Environment

## Prerequisites:

    -  Ansible installed on your control node (where you'll run the playbooks)
    -  SSH access to all target nodes (with key-based authentication)
    -  Python 3 on all target nodes
    -  Inventory file (inventory.ini) properly configured with your hosts

**Directory Structure:**

   ```bash
    kubernetes-setup/
    ├── inventory.ini          # Host definitions
    ├── kube-dependencies.yml  # Kubernetes base setup
    ├── container.yaml         # Docker/Containerd setup
    ├── kubectl.yaml           # kubectl installation
    └── containerd-config.toml # Containerd config
   ```
---

2. Verify Inventory

Check inventory.ini contains your correct hosts and credentials:

 ```ini
[reservations]
c267
c276
icgpu10

[master]
c267

[workers]
c276
icgpu10

[gpu]
icgpu10
```
---

3. Execution Order

Run playbooks in this sequence:

1. Setup Container Runtime (Docker/Containerd)


```bash
ansible-playbook -i inventory.ini container.yaml
```

    - Installs Docker and configures Containerd
    - Auto-detects GPU nodes and configures NVIDIA runtime
    - Validates OS compatibility (Ubuntu 24.04/Rocky Linux 8)

2. Install Kubernetes Dependencies
```bash
ansible-playbook -i inventory.ini kube-dependencies.yml
```

- Disables swap
- Configures kernel modules
- Installs:
    * Containerd (with systemd cgroup driver)
    * kubelet, kubeadm
    * kubectl (on master node)

3. (Optional) Install Latest kubectl

```bash
ansible-playbook -i inventory.ini kubectl.yaml
```

- Downloads and verifies the latest stable kubectl
- Useful if you want a newer version than what's in repositories

4. Initialize Kubernetes Cluster
On Master Node (c267):

```bash
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
```

Follow the post-init instructions to:

a. Set up kubeconfig for your user:

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

b. Install a CNI plugin (e.g., Flannel):

```bash
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
````

** Join Worker Nodes:**
Run the kubeadm join command generated during master initialization on all worker nodes (c276, icgpu10).

---

5. Verify Installation
```bash
kubectl get nodes  # Should show all nodes as 'Ready'
kubectl get pods -A  # Check system pods are running
````
--- 
**Key Customization Points**

1. Kubernetes Version:

    * Edit kube-dependencies.yml to change 1.32.* to your desired version

2. GPU Support:

    * The playbook auto-detects NVIDIA GPUs via lspci
    * For manual override, set has_nvidia: true/false in container.yaml vars

3. Containerd Config:

    * The containerd-config.toml is applied automatically
    * Customize paths if not using /home/rdesouz4/.kube/etc/containerd/

---

**Troubleshooting Tips**

1. Dry Runs:

```bash
ansible-playbook -i inventory.ini container.yaml --check
```

2. Specific Hosts:

```bash
ansible-playbook -i inventory.ini container.yaml --limit icgpu10
```

3. Common Issues:

    * If nodes don't join: Check kubelet logs (journalctl -u kubelet)
    * If containers don't start: Verify Containerd config (sudo ctr version)
    * GPU nodes: Run nvidia-smi and nvidia-container-runtime --version