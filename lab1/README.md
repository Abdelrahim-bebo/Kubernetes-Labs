# Kubernetes Control Plane Setup (Red Hat / CentOS / Rocky / AlmaLinux)

This guide provides step-by-step instructions to initialize a single-node Kubernetes Control Plane (Master Node) on the Red Hat family of Linux distributions using `kubeadm` and `containerd`.

---

## Prerequisites

Run all commands as a user with `sudo` privileges.

---

### 1. Disable Swap

Kubernetes requires swap to be disabled to ensure stable memory management.

```bash
# Disable swap for the current session
sudo swapoff -a

# Disable swap permanently (survives reboots)
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```

---

### 2. Configure SELinux

Set SELinux to permissive mode so containers can access the host filesystem.

```bash
sudo setenforce 0
sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
```

---

### 3. Open Firewall Ports

Allow Kubernetes components and the CNI (Flannel) to communicate securely.

```bash
sudo firewall-cmd --permanent --add-port=6443/tcp      # API Server
sudo firewall-cmd --permanent --add-port=10250/tcp     # Kubelet
sudo firewall-cmd --permanent --add-port=2379-2380/tcp # etcd
sudo firewall-cmd --permanent --add-port=10257/tcp     # Controller Manager
sudo firewall-cmd --permanent --add-port=10259/tcp     # Scheduler
sudo firewall-cmd --permanent --add-port=8472/udp      # Flannel VXLAN
sudo firewall-cmd --reload
```

---

## System Configuration

### 4. Load Kernel Modules

```bash
sudo tee /etc/modules-load.d/k8s.conf <<EOF
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter
```

---

### 5. Configure Network Routing (IP Forwarding)

```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system
```

---

## Install Container Runtime (containerd)

### 6. Add Docker Repository and Install

```bash
sudo dnf install -y dnf-plugins-core
sudo dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
sudo dnf install -y containerd.io
```

---

### 7. Configure containerd (Critical for Red Hat)

Generate the default config and set `SystemdCgroup = true` to match Red Hat's init system.

```bash
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/SystemdCgroup \= false/SystemdCgroup \= true/g' /etc/containerd/config.toml

sudo systemctl enable --now containerd
```

---

## Install Kubernetes Components

### 8. Add Kubernetes Repository

> **Note:** This guide targets Kubernetes v1.34.

```bash
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.34/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.34/rpm/repodata/repomd.xml.key
EOF
```

---

### 9. Install kubeadm, kubelet, and kubectl

```bash
sudo dnf install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
sudo systemctl enable --now kubelet
```

---

## Initialize the Cluster

### 10. Pull Images and Initialize

```bash
# Verify container runtime is working
sudo kubeadm config images pull

# Initialize the cluster (assuming 10.244.0.0/16 for Flannel CNI)
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
```

---

### 11. Configure Local User Access

Run this as your regular user (or root) to use `kubectl` commands.

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

---

### 12. Install Network Plugin (Flannel)

```bash
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```

---

## Optional: Single-Node Cluster Setup

If you want to run applications directly on this Master node (for a single-node lab), remove the control-plane taint:

```bash
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
```

---

## Verification

Wait about 60 seconds after installing Flannel, then check your cluster status:

```bash
kubectl get nodes
kubectl get pods -A
```
