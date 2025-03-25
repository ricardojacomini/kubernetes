# GPU-Optimized Kubernetes Cluster Automation

![Kubernetes](https://img.shields.io/badge/Kubernetes-1.28-blue) 
![Ansible](https://img.shields.io/badge/Ansible-2.12+-red) 
![GPU](https://img.shields.io/badge/NVIDIA-GPU%20Ready-green)
![OS](https://img.shields.io/badge/OS-Ubuntu%2024.04%20%7C%20Rocky%209-success)

Automated deployment of production-ready Kubernetes clusters with NVIDIA GPU acceleration using Ansible. Designed for AI/ML workloads and high-performance computing.

## Key Features
- **Multi-OS Support**: Ubuntu 24.04 & Rocky Linux 8/9
- **GPU Acceleration**: Automatic NVIDIA driver & container toolkit setup
- **Optimized Performance**: Preconfigured kernel settings and cgroupv2
- **Verified Components**: Version-pinned dependencies

## Table of Contents
- [Quick Start](#-quick-start)
- [Architecture Overview](#-architecture-overview)
- [Execution Order](#-execution-order)
- [Customization](#%EF%B8%8F-customization)
- [Verification](#-verification)
- [Troubleshooting](#-troubleshooting)
- [Resources](#-resources)

## 🚀 Quick Start
### Prerequisites
- Ansible 2.12+
- Python 3.8+
- Target nodes with:
  - Ubuntu 24.04 or Rocky Linux 9
  - Minimum 4GB RAM, 2 vCPUs per node
  - NVIDIA GPU (for GPU nodes)

### Deployment
```bash

# Option 1:
ml ansible-core

or

# Option 2 (if available):
git clone https://github.com/your-repo/gpu-kubernetes-ansible.git
cd gpu-kubernetes-ansible 

# Then

# First Install the required callback plugin
ansible-galaxy collection install ansible.posix

# run
ansible-playbook -i inventory.ini playbooks/site.yml
```

## 🏗️ Architecture Overview

```bash
kubernetes/
├── ansible.cfg              # Ansible configuration
├── inventory.ini            # Host definitions
├── group_vars/              # Group variables
│   ├── all.yml             # Common configurations
│   ├── ubuntu.yml          # Ubuntu-specific settings
│   ├── rocky.yml           # Rocky Linux settings
│   ├── gpu.yml             # GPU node configurations
│   └── monitoring.yml      # Monitoring stack configs
├── host_vars/              # Host-specific overrides
│   ├── a100-node.yml       # A100-specific settings
│   └── high-density.yml    # High-density node tweaks
├── roles/                  # Ansible roles
│   ├── common/             # Base system setup
│   ├── docker/             # Container runtime
│   ├── kubernetes/         # K8s components
│   └── nvidia/             # GPU optimization
└── playbooks/
    └── site.yml            # Master deployment playbook
```    
---

## 📋 Inventory Configuration

Ensure inventory.ini contains your correct hosts:

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

## 🔄 Execution Order

### 1. Preflight Checks

Verify system requirements and configurations:

```bash
ansible-playbook -i inventory.ini playbooks/site.yml --tags prerequisites --check
```

### 2. Base System Setup (All Nodes)
Configures kernel settings, repositories, and system dependencies:

```bash
ansible-playbook -i inventory.ini playbooks/site.yml --tags common --limit reservations
```

### 3. Container Runtime Setup (All Nodes)

Installs and configures Docker or Containerd:

```bash
ansible-playbook -i inventory.ini playbooks/site.yml --tags docker --limit reservations
```

### 4. GPU Setup (GPU Nodes Only)
Installs NVIDIA drivers and container toolkit:

```bash
ansible-playbook -i inventory.ini playbooks/site.yml --tags nvidia --limit gpu
```

### 5. Kubernetes Bootstrap (All Nodes)

Installs Kubernetes components (kubeadm, kubelet, kubectl):
```bash
ansible-playbook -i inventory.ini playbooks/site.yml --tags kubernetes --limit reservations
```

### 6. Master Initialization (First Run Only)
Initializes the control plane:

```bash
ansible-playbook -i inventory.ini playbooks/site.yml --tags master --limit master
```

### 7. Worker Join

On master node:
```bash
kubeadm token create --print-join-command
```

On each worker (run output from above):
```bash
kubeadm join <master-ip>:6443 --token <token> --discovery-token-ca-cert-hash <hash>
```

## ✔️ VERIFICATION

Cluster Health
```bash
ansible -i inventory.ini master -m command -a "kubectl get nodes -o wide"
```

GPU Status
```bash
ansible -i inventory.ini gpu -m command -a "nvidia-smi"
```

Detailed GPU Info
```bash
ansible -i inventory.ini gpu -m command -a "nvidia-smi --query-gpu=name,driver_version,cuda_version,memory.total --format=csv"
```

## 🎛️ Customization

### Core Configuration Variables

| Variable | File | Description | Default | Valid Options |
|----------|------|-------------|---------|---------------|
| `k8s_version` | `all.yml` | Kubernetes version | `1.28.0` | Any supported version |
| `container_runtime` | `all.yml` | Container runtime | `containerd` | `containerd`, `docker` |
| `pod_network` | `all.yml` | CNI plugin | `calico` | `calico`, `flannel`, `cilium` |
| `pod_network_cidr` | `all.yml` | Pod IP range | `10.244.0.0/16` | Valid CIDR range |

### GPU-Specific Variables

| Variable | File | Description | Default |
|----------|------|-------------|---------|
| `nvidia_runtime_class` | `roles/nvidia/vars/main.yml` | GPU pod scheduling class | `nvidia` |
| `nvidia_mig_enabled` | `gpu-nodes.yml` | Enable MIG partitioning | `false` |
| `nvidia_driver_version` | `gpu.yml` | Driver version | `535.86.05` |

Change CNI Plugin to Cilium:

```yaml
# group_vars/all.yml
pod_network: "cilium"
pod_network_cidr: "10.0.0.0/16"
cilium:
  kubeProxyReplacement: "strict"
  hubble:
    enabled: true
```

Configure Kubernetes Resource Reservations:

```yaml
# group_vars/all.yml
kubelet_reserved:
  cpu: "500m"
  memory: "1Gi"
  ephemeral-storage: "5Gi"
````

Enable MIG on A100/H100 GPUs:

for GPU-specific configurations

```yaml
# host_vars/gpu-node.yml 
nvidia_mig:
  enabled: true
  profiles:
    - "1g.5gb"  # 1 GPU instance with 5GB memory
    - "2g.10gb" # 2 GPU instances with 10GB memory each
  strategy: "mixed"  # Options: single, mixed
```

for cluster-wide networking

```yaml
# group_vars/all.yml 
pod_network: "cilium"
pod_network_cidr: "10.0.0.0/16"
cilium:
  kubeProxyReplacement: "strict"  # Options: disabled, probe, strict
  hubble:
    enabled: true
    metrics:
      enabled: true
  bandwidthManager:
    enabled: true
```

## OS-specific deployments:

```bash
### Ubuntu nodes
ansible-playbook playbooks/site.yml -l ubuntu_nodes

### Rocky nodes
ansible-playbook playbooks/site.yml -l rocky_nodes
````

## 🛠 Troubleshooting Guide

Common Issues
Nodes Not Joining Cluster

```bash
# Check kubelet logs
journalctl -u kubelet -n 100 --no-pager

# Verify certificates
openssl x509 -in /etc/kubernetes/pki/ca.crt -text -noout```
```

* Container runtime issues:

```bash
# Check containerd status
sudo ctr version

# Verify container images
sudo ctr images list
````

* GPU Problems:

```bash
# Verify NVIDIA driver and runtime status
ansible gpu_nodes -m command -a "nvidia-smi --query-gpu=name,driver_version,cuda_version,memory.total --format=csv"
ansible gpu_nodes -m command -a "nvidia-container-runtime --version"

# Additional diagnostic commands (run on GPU nodes directly):
# Verify driver loading
lsmod | grep nvidia

# Check Kubernetes GPU resources
kubectl describe node <gpu-node> | grep -A10 'Capacity:\|Allocatable:'

# Verify device plugin pods
kubectl get pods -n kube-system | grep nvidia-device-plugin
```

## 📚 Resources

- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [NVIDIA Container Toolkit](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/overview.html)
- [Ansible Best Practices](https://docs.ansible.com/ansible/latest/user_guide/playbooks_best_practices.html)
- [Kubernetes GPU Scheduling](https://kubernetes.io/docs/tasks/manage-gpus/scheduling-gpus)