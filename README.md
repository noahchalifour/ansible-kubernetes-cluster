# Homelab Kubernetes Cluster Deployment with Ansible

This project automates the deployment and configuration of a Kubernetes cluster in a homelab environment using Ansible and Kubespray. It's specifically designed to work with Proxmox VE virtualization platform.

## Overview

This Ansible project leverages [Kubespray](https://github.com/kubernetes-sigs/kubespray) to deploy a production-ready Kubernetes cluster with the following features:

- **Container Network Interface (CNI)**: Cilium for advanced networking and security
- **Ingress Controller**: NGINX for HTTP/HTTPS traffic routing
- **Load Balancer**: MetalLB for bare-metal load balancing
- **Certificate Management**: cert-manager for automatic TLS certificate provisioning
- **Dynamic Inventory**: Proxmox integration for automatic node discovery

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Proxmox VE Cluster                       │
├─────────────────────────────────────────────────────────────┤
│  ┌─────────────────┐  ┌─────────────────┐  ┌──────────────┐ │
│  │  Control Plane  │  │   Worker Node   │  │ Worker Node  │ │
│  │   (Master)      │  │                 │  │              │ │
│  │                 │  │                 │  │              │ │
│  │ - API Server    │  │ - Kubelet       │  │ - Kubelet    │ │
│  │ - etcd          │  │ - Cilium        │  │ - Cilium     │ │
│  │ - Scheduler     │  │ - MetalLB       │  │ - MetalLB    │ │
│  │ - Controller    │  │                 │  │              │ │
│  └─────────────────┘  └─────────────────┘  └──────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

## Prerequisites

### Infrastructure Requirements

1. **Proxmox VE cluster** with:
   - Virtual machines tagged appropriately (see [Node Tagging](#node-tagging))
   - Network connectivity between all nodes
   - SSH access to all nodes

2. **VM Requirements** (per Kubernetes node):
   - **Minimum**: 2 CPU cores, 4GB RAM, 20GB storage
   - **Recommended**: 4+ CPU cores, 8GB+ RAM, 50GB+ storage
   - Ubuntu 22.04 LTS or similar supported OS
   - Cloud-init configured for automatic provisioning

### Software Dependencies

- Python 3.8+
- Ansible 2.17+
- Go Task (Taskfile) - optional but recommended
- Access to Proxmox API

## Quick Start

### 1. Clone and Setup

```bash
git clone &lt;repository-url&gt;
cd kubernetes-ansible-homelab
```

### 2. Install Dependencies

Using Task (recommended):
```bash
task install
```

Or manually:
```bash
pip install -r requirements/pip.txt
ansible-galaxy install -r requirements/ansible.yml
```

### 3. Configure Environment

Create vault password file:
```bash
echo "your-vault-password" > .vault-pass
chmod 600 .vault-pass
```

### 4. Deploy Cluster

```bash
task deploy
```

Or with custom options:
```bash
task deploy INVENTORY=proxmox VERBOSITY=vv
```

## Configuration

### Node Tagging

The Proxmox dynamic inventory uses VM tags to determine node roles. Tag your VMs with:

- `k8s-control-plane`: For master/control plane nodes
- `k8s-worker`: For worker nodes  
- `etcd`: For dedicated etcd nodes (optional, can run on control plane)

### Environment Variables

Create a `.env` file with your Proxmox configuration:

```bash
# Proxmox API Configuration
PROXMOX_URL=https://your-proxmox.local:8006/api2/json
PROXMOX_USER=ansible@pve
PROXMOX_PASSWORD=your-proxmox-password
PROXMOX_NODE=proxmox-node-name

# SSH Configuration (optional)
SSH_PRIVATE_KEY=/path/to/your/ssh/key
```

### Inventory Configuration

The project uses a dynamic inventory (`inventories/proxmox.yml`) that automatically discovers nodes from Proxmox based on tags:

```yaml
plugin: community.proxmox.proxmox
want_facts: true
exclude_nodes: true
keyed_groups:
  - key: proxmox_tags_parsed
    separator: ""
    prefix: group
groups:
  kube_control_plane: "'k8s-control-plane' in (proxmox_tags_parsed | list)"
  kube_node: "'k8s-worker' in (proxmox_tags_parsed | list)" 
  etcd: "'etcd' in (proxmox_tags_parsed | list)"
```

### Cluster Configuration

#### Network Plugin (CNI)
Located in `group_vars/k8s_cluster/cni.yml`:
```yaml
# Use Cilium network plugin
kube_network_plugin: cilium
kube_owner: root
```

#### Add-ons Configuration
Located in `group_vars/k8s_cluster/addons.yml`:
```yaml
# Enable NGINX ingress controller
ingress_nginx_enabled: true

# Enable Cert-manager
cert_manager_enabled: true

# Enable MetalLB load balancer
metallb_enabled: true
metallb_speaker_enabled: true
metallb_namespace: metallb
kube_proxy_strict_arp: true
```

## Task Commands

This project uses [Go Task](https://taskfile.dev/) for automation:

| Command | Description |
|---------|-------------|
| `task install` | Install all dependencies (pip + ansible-galaxy) |
| `task validate` | Validate playbook syntax and run ansible-lint |
| `task prepare` | Prepare environment (create symlinks, etc.) |
| `task deploy` | Deploy Kubernetes cluster |

### Deploy with Options

```bash
# Use specific inventory
task deploy INVENTORY=custom-inventory

# Enable verbose output
task deploy VERBOSITY=vv

# Use custom SSH key
task deploy SSH_PRIVATE_KEY=/path/to/key

# Use custom vault password file
task deploy VAULT_PASSWORD=/path/to/vault-pass
```

## Post-Deployment

### Access Your Cluster

1. **Copy kubeconfig**:
   ```bash
   # From the first control plane node
   scp user@control-plane-ip:~/.kube/config ~/.kube/config
   ```

2. **Verify cluster**:
   ```bash
   kubectl get nodes
   kubectl get pods --all-namespaces
   ```

### Configure MetalLB

After deployment, configure MetalLB with your IP address pool:

```bash
kubectl apply -f - &lt;&lt;EOF
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: default-pool
  namespace: metallb
spec:
  addresses:
  - 192.168.1.100-192.168.1.110  # Adjust to your network
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: default
  namespace: metallb
spec:
  ipAddressPools:
  - default-pool
EOF
```

### Configure Ingress

Create ingress resources for your applications:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: example-ingress
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  tls:
  - hosts:
    - example.your-domain.com
    secretName: example-tls
  rules:
  - host: example.your-domain.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: example-service
            port:
              number: 80
```

## Troubleshooting

### Common Issues

1. **SSH Connection Failures**:
   - Verify SSH keys are properly configured
   - Check that `ansible_user` matches your VM user
   - Ensure SSH access without password prompts

2. **Proxmox API Issues**:
   - Verify API credentials in `.env` file
   - Check Proxmox API permissions for the user
   - Confirm Proxmox VE API is accessible

3. **Node Discovery Issues**:
   - Verify VM tags in Proxmox match expected values
   - Check that VMs have IP addresses assigned
   - Ensure cloud-init is properly configured

4. **Deployment Failures**:
   - Check logs with verbose output: `task deploy VERBOSITY=vvv`
   - Verify system requirements on all nodes
   - Ensure network connectivity between nodes

### Debug Commands

```bash
# Test inventory
ansible-inventory -i inventories/proxmox.yml --list

# Test connectivity
ansible all -i inventories/proxmox.yml -m ping

# Check node resources
ansible all -i inventories/proxmox.yml -m shell -a "free -h && df -h"
```

## Security Considerations

- Store sensitive variables in Ansible Vault
- Use SSH key authentication instead of passwords
- Regularly update the Kubespray version for security patches
- Configure proper firewall rules on Proxmox hosts
- Enable RBAC in your Kubernetes cluster
- Use cert-manager for automatic TLS certificate management

## Customization

### Adding Custom Roles

To add custom Ansible roles:

1. Create role directory: `mkdir -p roles/custom-role`
2. Add role to requirements: `requirements/ansible.yml`
3. Include in main playbook or create new playbooks

### Modifying Kubespray Configuration

Override Kubespray variables in `group_vars/k8s_cluster/` or `group_vars/all.yml`:

```yaml
# Example: Change Kubernetes version
kube_version: v1.29.0

# Example: Modify pod subnet
kube_pods_subnet: 10.233.64.0/18
```

## Contributing

1. Fork the repository
2. Create a feature branch
3. Test your changes thoroughly
4. Submit a pull request with detailed description

## License

This project is licensed under the MIT License - see the LICENSE file for details.

## Resources

- [Kubespray Documentation](https://kubespray.io/)
- [Proxmox VE Documentation](https://www.proxmox.com/en/proxmox-ve)
- [Ansible Documentation](https://docs.ansible.com/)
- [Kubernetes Documentation](https://kubernetes.io/docs/)