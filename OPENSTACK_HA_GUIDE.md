# OpenStack HA Kubernetes Deployment Guide

This guide provides step-by-step instructions for deploying a highly available Kubernetes cluster on OpenStack using Kubespray v2.26.0 with Kubernetes 1.30.8.

## Architecture Overview

### Cluster Topology
- **3 Master Nodes**: Control plane with stacked etcd
- **3 Worker Nodes**: Application workload nodes
- **Network Plugin**: Cilium with eBPF capabilities
- **Load Balancer**: External load balancer for API server access

### Node Configuration
```
Master Nodes (Control Plane):
├── master1 (10.0.0.10) - etcd1, kube-apiserver, kube-controller-manager, kube-scheduler
├── master2 (10.0.0.11) - etcd2, kube-apiserver, kube-controller-manager, kube-scheduler
└── master3 (10.0.0.12) - etcd3, kube-apiserver, kube-controller-manager, kube-scheduler

Worker Nodes:
├── worker1 (10.0.0.20) - kubelet, kube-proxy, cilium-agent
├── worker2 (10.0.0.21) - kubelet, kube-proxy, cilium-agent
└── worker3 (10.0.0.22) - kubelet, kube-proxy, cilium-agent
```

## Prerequisites

### OpenStack Infrastructure
1. **OpenStack Environment**: Working OpenStack deployment with:
   - Nova (Compute)
   - Neutron (Networking)
   - Cinder (Block Storage) - optional
   - Heat (Orchestration) - optional

2. **VM Instances**: 6 Ubuntu 20.04/22.04 instances with:
   - **Master nodes**: 4 vCPU, 8GB RAM, 50GB disk
   - **Worker nodes**: 2 vCPU, 4GB RAM, 50GB disk
   - SSH access configured
   - Security groups allowing required ports

3. **Network Configuration**:
   - Private network with subnet (e.g., 10.0.0.0/24)
   - Router with external gateway
   - Floating IPs for external access (optional)

### Required Ports
```
Master Nodes:
- 6443/tcp  - Kubernetes API server
- 2379-2380/tcp - etcd server client API
- 10250/tcp - Kubelet API
- 10251/tcp - kube-scheduler
- 10252/tcp - kube-controller-manager

Worker Nodes:
- 10250/tcp - Kubelet API
- 30000-32767/tcp - NodePort Services

All Nodes:
- 22/tcp - SSH
- 179/tcp - BGP (Cilium)
- 4789/udp - VXLAN (Cilium)
- 51871/udp - WireGuard (Cilium encryption)
```

### Local Environment
- **Ansible Control Node**: Linux/macOS machine with:
  - Python 3.8+
  - Ansible 2.14+
  - SSH access to all cluster nodes

## Step 1: Prepare OpenStack Instances

### Create Security Group
```bash
# Create security group for Kubernetes cluster
openstack security group create k8s-cluster

# Allow SSH
openstack security group rule create --protocol tcp --dst-port 22 k8s-cluster

# Allow Kubernetes API
openstack security group rule create --protocol tcp --dst-port 6443 k8s-cluster

# Allow etcd
openstack security group rule create --protocol tcp --dst-port 2379:2380 k8s-cluster

# Allow Kubelet
openstack security group rule create --protocol tcp --dst-port 10250 k8s-cluster

# Allow NodePorts
openstack security group rule create --protocol tcp --dst-port 30000:32767 k8s-cluster

# Allow Cilium BGP
openstack security group rule create --protocol tcp --dst-port 179 k8s-cluster

# Allow Cilium VXLAN
openstack security group rule create --protocol udp --dst-port 4789 k8s-cluster

# Allow WireGuard (Cilium encryption)
openstack security group rule create --protocol udp --dst-port 51871 k8s-cluster

# Allow internal communication
openstack security group rule create --protocol icmp k8s-cluster
openstack security group rule create --remote-group k8s-cluster k8s-cluster
```

### Launch Instances
```bash
# Create master nodes
for i in {1..3}; do
  openstack server create \
    --flavor m1.large \
    --image ubuntu-20.04 \
    --key-name your-keypair \
    --security-group k8s-cluster \
    --network private-network \
    master${i}
done

# Create worker nodes
for i in {1..3}; do
  openstack server create \
    --flavor m1.medium \
    --image ubuntu-20.04 \
    --key-name your-keypair \
    --security-group k8s-cluster \
    --network private-network \
    worker${i}
done
```

## Step 2: Prepare Kubespray

### Clone and Setup
```bash
# Clone this repository
git clone <repository-url>
cd kubespray

# Checkout Kubernetes 1.30 compatible version
git checkout v2.26.0

# Install Python dependencies
pip install -r requirements.txt
```

### Configure Inventory
```bash
# Copy the pre-configured OpenStack HA inventory
cp -r inventory/openstack-ha inventory/my-openstack-cluster

# Update with your actual IP addresses
vim inventory/my-openstack-cluster/inventory.ini
```

Update the IP addresses in the inventory file:
```ini
[all]
master1 ansible_host=<MASTER1_IP> ip=<MASTER1_IP> etcd_member_name=etcd1
master2 ansible_host=<MASTER2_IP> ip=<MASTER2_IP> etcd_member_name=etcd2
master3 ansible_host=<MASTER3_IP> ip=<MASTER3_IP> etcd_member_name=etcd3
worker1 ansible_host=<WORKER1_IP> ip=<WORKER1_IP>
worker2 ansible_host=<WORKER2_IP> ip=<WORKER2_IP>
worker3 ansible_host=<WORKER3_IP> ip=<WORKER3_IP>
```

## Step 3: Deploy Kubernetes Cluster

### Pre-deployment Checks
```bash
# Test connectivity to all nodes
ansible -i inventory/my-openstack-cluster/inventory.ini all -m ping

# Check available disk space (ensure >50GB on each node)
ansible -i inventory/my-openstack-cluster/inventory.ini all -m shell -a "df -h /"
```

### Deploy Cluster
```bash
# Deploy the cluster (this takes 15-30 minutes)
ansible-playbook -i inventory/my-openstack-cluster/inventory.ini \
  --become --become-user=root \
  cluster.yml
```

### Verify Deployment
```bash
# SSH to first master node
ssh ubuntu@<MASTER1_IP>

# Check cluster status
sudo kubectl get nodes
sudo kubectl get pods --all-namespaces

# Verify Cilium is running
sudo kubectl get pods -n kube-system -l k8s-app=cilium
```

## Step 4: Access the Cluster

### Setup kubectl Access
```bash
# Copy kubeconfig from master node
scp ubuntu@<MASTER1_IP>:~/.kube/config ~/.kube/config-openstack-ha

# Set KUBECONFIG environment variable
export KUBECONFIG=~/.kube/config-openstack-ha

# Test access
kubectl get nodes
kubectl cluster-info
```

### Setup Load Balancer (Optional)
For production use, configure an external load balancer in front of the API servers:

```bash
# Example using OpenStack Octavia
openstack loadbalancer create --name k8s-api-lb --vip-subnet-id <subnet-id>
openstack loadbalancer listener create --name k8s-api-listener --protocol TCP --protocol-port 6443 k8s-api-lb
openstack loadbalancer pool create --name k8s-api-pool --lb-algorithm ROUND_ROBIN --listener k8s-api-listener --protocol TCP
openstack loadbalancer member create --subnet-id <subnet-id> --address <MASTER1_IP> --protocol-port 6443 k8s-api-pool
openstack loadbalancer member create --subnet-id <subnet-id> --address <MASTER2_IP> --protocol-port 6443 k8s-api-pool
openstack loadbalancer member create --subnet-id <subnet-id> --address <MASTER3_IP> --protocol-port 6443 k8s-api-pool
```

## Troubleshooting

### Common Issues

1. **SSH Connection Failures**
   ```bash
   # Check security groups and key pairs
   openstack server show <instance-name>
   
   # Verify SSH key is correct
   ssh-keygen -l -f ~/.ssh/id_rsa.pub
   ```

2. **Disk Space Issues**
   ```bash
   # Check disk usage on all nodes
   ansible -i inventory/my-openstack-cluster/inventory.ini all -m shell -a "df -h"
   
   # Clean up if needed
   ansible -i inventory/my-openstack-cluster/inventory.ini all -m shell -a "sudo apt autoremove -y"
   ```

3. **Network Connectivity Issues**
   ```bash
   # Test internal connectivity between nodes
   ansible -i inventory/my-openstack-cluster/inventory.ini all -m shell -a "ping -c 3 <MASTER1_IP>"
   
   # Check Cilium status
   kubectl exec -n kube-system ds/cilium -- cilium status
   ```

4. **etcd Issues**
   ```bash
   # Check etcd cluster health on master nodes
   sudo etcdctl --endpoints=https://127.0.0.1:2379 \
     --cacert=/etc/ssl/etcd/ssl/ca.pem \
     --cert=/etc/ssl/etcd/ssl/admin.pem \
     --key=/etc/ssl/etcd/ssl/admin-key.pem \
     endpoint health
   ```

### Reset Cluster
If you need to start over:
```bash
# Reset the cluster (WARNING: This destroys everything!)
ansible-playbook -i inventory/my-openstack-cluster/inventory.ini \
  --become --become-user=root \
  reset.yml

# Clean up
ansible -i inventory/my-openstack-cluster/inventory.ini all -m shell -a "sudo rm -rf /etc/kubernetes /var/lib/kubelet /var/lib/etcd"
```

## Maintenance

### Adding Nodes
```bash
# Add node to inventory file first, then run
ansible-playbook -i inventory/my-openstack-cluster/inventory.ini \
  --become --become-user=root \
  scale.yml
```

### Upgrading Cluster
```bash
# Update Kubernetes version in group_vars and run
ansible-playbook -i inventory/my-openstack-cluster/inventory.ini \
  --become --become-user=root \
  upgrade-cluster.yml
```

### Backup etcd
```bash
# Create etcd backup
ansible-playbook -i inventory/my-openstack-cluster/inventory.ini \
  --become --become-user=root \
  -e etcd_backup_location=/opt/etcd-backup \
  playbooks/backup-etcd.yml
```

## Configuration Details

- **Kubespray Version**: v2.26.0
- **Kubernetes Version**: v1.30.8
- **Container Runtime**: containerd
- **Network Plugin**: Cilium v1.15.4
- **DNS**: CoreDNS
- **Ingress**: Optional (nginx-ingress available)
- **Storage**: Local storage (Cinder CSI plugin available)

For advanced configuration options, see the files in `inventory/openstack-ha/group_vars/`.