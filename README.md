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
1. Clone the repository
2. Configure your `inventory.ini` (see below)
3. Run the deployment:
```bash
ansible-playbook -i inventory.ini playbooks/site.yml
```

## 🏗️ Architecture Overview

```bash
kubernetes/
├── ansible.cfg              # Ansible configuration
├── inventory.ini            # Host definitions
├── group_vars/              # Variable definitions
│   ├── all.yml             # Common configurations
│   ├── ubuntu.yml          # Ubuntu-specific settings
│   ├── rocky.yml           # Rocky Linux settings
│   ├── gpu.yml             # GPU node configurations
│   └── monitoring.yml      # Monitoring stack configs
├── host_vars/
│   ├── a100-node.yml       # A100-specific overrides
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

```bash
# Check cluster status
ansible -i inventory.ini master -m command -a "kubectl get nodes"

# Verify GPU nodes
ansible -i inventory.ini gpu -m command -a "nvidia-smi"
```

## 🎛️ Customization

| Variable | File | Description | Default |
|----------|------|-------------| --------|
| `k8s_version` | `group_vars/all.yml` | Kubernetes version | 1.28 |
| `nvidia_runtime_class` | `roles/nvidia/vars/main.yml` | GPU pod scheduling | nvidia | 
| `pod_network` | `roles/kubernetes/vars/main.yml` | Calico/Flannel/Cilium | calico |

### Example: Enable MIG on A100/H100

```yaml
nvidia_mig:
  enabled: true
  profiles:
    - "1g.5gb"
    - "2g.10gb"
```

## OS-specific deployments:

```bash
### Ubuntu nodes
ansible-playbook playbooks/site.yml -l ubuntu_nodes

### Rocky nodes
ansible-playbook playbooks/site.yml -l rocky_nodes
````

## 🛠 Troubleshooting

* Nodes not joining cluster:
Check kubelet logs

```bash
journalctl -u kubelet
```

* Container runtime issues:

Verify Containerd config

```bash
sudo ctr version
```

* GPU node verification:

Run nvidia-smi and nvidia-container-runtime --version

```bash
ansible gpu_nodes -m command -a "nvidia-smi --query-gpu=driver_version,cuda_version --format=csv"   
nvidia-smi --query-gpu=driver_version,cuda_version --format=csv
nvidia-container-runtime --version
```

## 📚 Resources

- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [NVIDIA Container Toolkit](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/overview.html)
- [Ansible Best Practices](https://docs.ansible.com/ansible/latest/user_guide/playbooks_best_practices.html)