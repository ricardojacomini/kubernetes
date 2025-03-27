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

# 📁 Kubernetes Cluster Deployment with Ansible

# ![Kubernetes Logo](https://upload.wikimedia.org/wikipedia/commons/thumb/3/39/Kubernetes_logo_without_workmark.svg/1200px-Kubernetes_logo_without_workmark.svg.png)

##  🏗️ Project Structure

```text
├── 📜 ansible.cfg            # Ansible configuration
├── 📂 group_vars/            # Group-specific variables
│   ├── 🎮 k8s-cluster.yml    # Cluster-wide Kubernetes configuration
│   ├── 🏷️ main.yml           # Common variables across all hosts
│   ├── 📊 monitoring.yml     # Monitoring stack configuration
│   ├── 🪨 rocky.yml          # Rocky Linux specific settings
│   └── 🐧 ubuntu.yml         # Ubuntu specific settings
├── 📂 inventory/             # Inventory management
│   └── 🏭 production.ini     # Production inventory file
├── 📂 playbooks/             # Deployment playbooks
│   └── ▶️ site.yml           # Main deployment playbook
└── 📂 roles/                 # Ansible roles
    ├── ⚙️ common/            # Base system configuration
    ├── 🐳 docker/            # Docker installation and config
    ├── ☸️ kubernetes/        # Kubernetes cluster deployment
    └── 🎮 nvidia/            # NVIDIA GPU configuration

## Validation Workflow

### When Validations Run
1. **Pre-Deployment**:
   - ✅ Hardware compatibility (GPU, RAM)
   - ✅ OS version checks
   - ✅ Network connectivity

2. **Pre-Configuration**:

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

- Check Ansible's path resolution with:

```bash
ansible -i inventory.ini reservations -m debug -a "msg={{ lookup('file', 'roles/common/tasks/rocky.yml') }}"
````

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

Test the configuration with:

```bash
# Check containerd config
ansible gpu_nodes -m shell -a "containerd config dump | grep -A10 nvidia"

# Verify runtime
ansible gpu_nodes -m shell -a "nvidia-container-cli --version"
```

## 🎛️ Customization

### Core Configuration Variables

| Variable | File | Description | Default | Valid Options |
|----------|------|-------------|---------|---------------|
| `k8s_version` | `group_vars/main.ymll` | Kubernetes version | `1.28.0` | Any supported version |
| `container_runtime` | `group_vars/main.yml` | Container runtime | `containerd` | `containerd`, `docker` |
| `pod_network` | `group_vars/main.yml` | CNI plugin | `calico` | `calico`, `flannel`, `cilium` |
| `pod_network_cidr` | `group_vars/main.yml` | Pod IP range | `192.168.1.0/24` | Valid CIDR range |

### GPU-Specific Variables

| Variable | File | Description | Default |
|----------|------|-------------|---------|
| `nvidia_runtime_class` | `roles/nvidia/vars/main.yml` | GPU pod scheduling class | `nvidia` |
| `nvidia_mig_enabled` | `group_vars/main.yml` | Enable MIG partitioning | `false` |
| `nvidia_driver_version` | group_vars/main.ymll` | Driver version | `535.86.05` |

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
ansible-playbook playbooks/site.yml -l ubuntu

### Rocky nodes
ansible-playbook playbooks/site.yml -l rocky
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

## Tested Environments

✅ **Extensively validated** on:  
- Rocky Linux 8.x/9.x  
- NVIDIA GPUs (A100/H100) with:
  - Driver v550+
  - Containerd v1.7+
  - Kubernetes v1.28+

⚠️ **Requires additional validation** on:  
- Ubuntu 20.04/22.04  
  - Particularly with:
    - SecureBoot configurations
    - Alternative display managers (Wayland/X11)
    - Non-NVIDIA GPU setups

### Verification Matrix
| Component       | Rocky Linux | Ubuntu |
|-----------------|-------------|--------|
| NVIDIA Drivers  | ✅          | ⚠️     |
| Containerd      | ✅          | ⚠️     |
| Kubernetes      | ✅          | ⚠️     |
| MIG Partitioning| ✅          | N/A    |

**Contributions welcome** for Ubuntu-specific testing and improvements!