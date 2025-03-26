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
# Clone repository
git clone https://github.com/your-repo/gpu-kubernetes-ansible.git
cd gpu-kubernetes-ansible

# Install requirements
python3 -m pip install -r requirements.txt
ansible-galaxy install -r requirements.yml

# Deploy (see full options below)
ansible-playbook -i inventory.ini playbooks/site.yml

# GPU-specific deployment
ansible-playbook -i inventory.ini playbooks/site.yml --tags nvidia --limit gpu
```

## 🏗️ Architecture Overview

```bash
kubernetes/
├── 📜 ansible.cfg             # Ansible configuration
├── inventory.ini           # Host definitions
├── 📂 group_vars/             # Group-specific variables
│   ├── 🏷️ all.yml             # Common configurations
│   ├── 🎮 gpu.yml             # GPU node configurations
│   ├── a100-node.yml       # NVIDIA A100 GPU node configurations
│   ├── k8s-cluster.yml     # Kubernetes cluster network settings
│   ├── monitoring.yml      # Monitoring stack configs
│   └── os/                 # OS-specific variables
│        ├── rocky.yml      # Rocky Linux configurations
│        └── ubuntu.yml     # Ubuntu configurations
├── host_vars/              # Host-specific overrides
│   ├── gpu-nodes.yml       # A100-specific settings
│   └── high-density.yml    # High-density node tweaks
├── roles/                  # Ansible roles
│   ├── common/             # Core system configuration (kernel, repos, validation)
│   │      └── tasks/       # NEW: Configuration validation tasks
│   │           ├── validate.yml 
│   │           └──   ...   # Other common tasks
│   ├── docker/             # Container runtime setup
│   ├── kubernetes/         # Cluster orchestration components
│   │      └── templates/   # Docker daemon JSON template
│   │           └── config.yml.j2
│   └── nvidia/             # GPU acceleration stack
└── playbooks/
    └── site.yml            # Master deployment playbook
```  

## Validation Workflow

### When Validations Run
1. **Pre-Deployment**:
   - ✅ Hardware compatibility (GPU, RAM)
   - ✅ OS version checks
   - ✅ Network connectivity

2. **Post-Configuration**:
    ```yaml
    # roles/common/tasks/main.yml
    - include_tasks: repos.yml    # Package repositories
    - include_tasks: validate.yml # Config validation << NEW
    - include_tasks: nvidia.yml   # GPU setup
    ```

    ```bash
    # Run all validations
    ansible-playbook playbooks/site.yml --tags validation

    # Skip validations (not recommended)
    ansible-playbook playbooks/site.yml --skip-tags validation

    # Verify container runtime configs
    ansible all -m include_role -a name=common tasks_from=validate.yml
    ```

    **Best Practices to Avoid Reboots:**

    ✅ Run all non-reboot steps in a single playbook to minimize interruptions:

    ```bash
    ansible-playbook -i inventory.ini playbooks/site.yml \
      --tags common,docker,kubernetes \
      --limit reservations --check
    ```


3. **Post-Deployment**:

    ```bash
    # Cluster health check
    kubectl get nodes -o wide
    kubectl get pods -A
    ```

4. **Execution Order**:
- Estimated timing: Typical Deployment Timeline
    | Phase | Duration | Notes |
    |-------|----------|-------|
    | Preflight | 2min | Fast validation |
    | GPU Setup | 15-30min | Driver install may require reboot |
    | K8s Bootstrap | 5min | Network-dependent |

- Visual workflow diagram:
  ```mermaid
  graph TD
      A[Preflight Checks] --> B[Base System]
      B                   --> C[Container Runtime]
      C                   --> D[GPU Setup]
      D                   --> E[Kubernetes Bootstrap]
      E                   --> F[Master Init]
      F                   --> G[Worker Join]
  ```

5. **Customization Section**:
- Warning about required reboots:
  ```markdown
  ⚠️ 🔥 **Reboot Required**: GPU driver changes require node reboots. Schedule accordingly.
  ```

6. Troubleshooting
- Quick copy-paste diagnostics:

```bash
# Get all node conditions
kubectl get nodes -o=jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.conditions[?(@.status=="True")].type}{"\n"}{end}'

# Check GPU allocatable resources
kubectl get nodes -o json | jq '.items[].status.allocatable'
```

### 7. Resources - NVIDIA-specific troubleshooting:
- [NVIDIA Troubleshooting Guide](https://docs.nvidia.com/datacenter/tesla/troubleshooting-guide/index.html)
- [Kubernetes Debugging Docs](https://kubernetes.io/docs/tasks/debug/)

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

### Quick Fixes
| Symptom | Solution |
|---------|----------|
| GPU nodes not recognized | `ansible gpu -m reboot` then re-run playbook with `--tags nvidia` |
| Image pull errors | Verify `docker_proxy` settings in `group_vars/all.yml` |


## 📚 Resources

- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [NVIDIA Container Toolkit](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/overview.html)
- [Ansible Best Practices](https://docs.ansible.com/ansible/latest/user_guide/playbooks_best_practices.html)
- [Kubernetes GPU Scheduling](https://kubernetes.io/docs/tasks/manage-gpus/scheduling-gpus)

### Version-Specific Guides
- [NVIDIA Driver Matrix](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/platform-support.html)
- [Kubernetes Version Skew Policy](https://kubernetes.io/releases/version-skew-policy/)