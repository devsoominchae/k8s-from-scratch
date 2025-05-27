## This repo contains the files and explanations to setup Kubernetes cluster for SAS Viya from scratch.

## Required Steps
1. Environment setup
2. Kubernetes control node setup

## 1. Environment setup
- RACE Image ID: 1439134
- 1 Control Node, 5 Worker Nodes on VirtualBox with Oracle 8.10
- 24 GB RAM, 8 CPUs, 75 GB Disk each
- Each node needs static IP configured
- Use personal NAT Network and forward inbound traffic to each VM (Host IP: Blank, Host Port: 2220, Guest IP 10.0.2.10, Guest Port: 22)

## 2. Kubernetes control node setup

Steps mentioned here must be done only on the conrol node.

### Disable swap memory.
Kubernetes is still undergoing process to stablize the usage of swap. [Read this K8s blog](https://kubernetes.io/blog/2025/03/25/swap-linux-improvements/)
######
    sudo swapoff -a
    sudo sed -i '/ swap / s/^/#/' /etc/fstab

### Enable containerd repository
######
    sudo yum install -y yum-utils
    sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo

### Install containerd
######
    sudo yum install -y containerd.io

### Configure containerd
Create an initial configuration for containerd. Then enable systemd cgroup driver. [Wonder why?](https://kubernetes.io/docs/setup/production-environment/container-runtimes/)
######
    sudo mkdir -p /etc/containerd
    containerd config default | sudo tee /etc/containerd/config.toml > /dev/null
    sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml

### Enable Kubernetes repository
######
    sudo tee /etc/yum.repos.d/kubernetes.repo << EOF
    [kubernetes]
    name=Kubernetes
    baseurl=https://pkgs.k8s.io/core:/stable:/v1.27/rpm/
    enabled=1
    gpgcheck=1
    gpgkey=https://pkgs.k8s.io/core:/stable:/v1.27/rpm/repodata/repomd.xml.key
    exclude=kubelet kubeadm kubectl cri-tools kubernetes-cni
    EOF
    
### Install Kubernetes binaries
######
    sudo yum install --disableexcludes=kubernetes kubectl kubeadm kubelet -y

