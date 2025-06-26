# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Kubespray is an Ansible-based tool for deploying production-ready Kubernetes clusters. It uses Ansible playbooks to install and configure Kubernetes components across multiple cloud providers and bare metal environments.

## Core Commands

### Cluster Management
- `ansible-playbook -i inventory/sample/inventory.ini cluster.yml` - Deploy a new Kubernetes cluster
- `ansible-playbook -i inventory/sample/inventory.ini scale.yml` - Scale cluster (add/remove nodes)
- `ansible-playbook -i inventory/sample/inventory.ini upgrade-cluster.yml` - Upgrade cluster components
- `ansible-playbook -i inventory/sample/inventory.ini reset.yml` - Reset and clean up cluster
- `ansible-playbook -i inventory/sample/inventory.ini remove-node.yml -e node=nodename` - Remove specific node
- `ansible-playbook -i inventory/sample/inventory.ini recover-control-plane.yml` - Recover control plane

### Development and Testing
- `cd tests && make create-vagrant` - Create Vagrant test environment
- `cd tests && make delete-vagrant` - Destroy Vagrant test environment
- `molecule test` - Run molecule tests (from individual role directories)
- `ansible-playbook tests/cloud_playbooks/create-kubevirt.yml` - Create cloud test environment

### Dependencies
- Install Python requirements: `pip install -r requirements.txt`
- Install test requirements: `pip install -r tests/requirements.txt`

## Architecture

### Main Playbooks Structure
- `cluster.yml` - Main entry point that orchestrates the full cluster deployment
- `playbooks/cluster.yml` - Core deployment playbook with these phases:
  1. **Preparation** - OS bootstrap, container engine setup, downloads
  2. **etcd Installation** - Deploy etcd cluster for Kubernetes state storage
  3. **Kubernetes Installation** - Deploy control plane and worker nodes
  4. **CNI Installation** - Install Container Network Interface
  5. **Applications** - Deploy cluster applications and add-ons

### Key Role Categories
- **bootstrap_os/** - OS-specific bootstrapping (Ubuntu, CentOS, etc.)
- **container-engine/** - Container runtime setup (containerd, Docker, CRI-O)
- **kubernetes/** - Core Kubernetes components (control-plane, node, kubeadm)
- **network_plugin/** - CNI implementations (Calico, Cilium, Flannel, etc.)
- **kubernetes-apps/** - Cluster applications (DNS, ingress, storage, monitoring)
- **etcd/** - etcd cluster management and configuration

### Inventory and Configuration
- `inventory/sample/` - Template inventory and group variables
- `inventory/sample/group_vars/all/` - Global configuration variables
- `inventory/sample/group_vars/k8s_cluster/` - Kubernetes-specific settings
- Network plugin configs: `k8s-net-*.yml` files for different CNI options

### Supported Platforms
- **Container Runtimes**: containerd (default), Docker, CRI-O
- **Network Plugins**: Calico (default), Cilium, Flannel, Kube-OVN, Kube-Router, Macvlan
- **Cloud Providers**: AWS, GCP, Azure, OpenStack, vSphere, Hetzner, OCI
- **Operating Systems**: Ubuntu, Debian, CentOS/RHEL, Fedora, openSUSE, Flatcar

## Development Workflow

1. **Configuration**: Start by copying `inventory/sample` and customizing variables
2. **Testing**: Use Vagrant or cloud providers for testing deployments
3. **Roles**: Individual components are organized as Ansible roles with molecule tests
4. **Variables**: Configuration is heavily variable-driven - check `group_vars/` for options
5. **Validation**: The `validate_inventory` role checks configuration before deployment

## Important Notes

- This is an Ansible-based infrastructure-as-code project, not a traditional software application
- Most "testing" involves deploying actual infrastructure rather than unit tests
- Configuration is managed through Ansible inventory files and group variables
- The project supports many deployment scenarios through variable customization
- Each role can be tested independently using Molecule framework