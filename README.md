## This repo contains the files and explanations to setup Kubernetes cluster for SAS Viya from scratch.

## Required Steps
1. Environment setup
2. Kubernetes control plane node setup
3. Kubernetes worker node setup

## 1. Environment setup
- RACE Image ID: 1439134
- 1 Control Node, 5 Worker Nodes on VirtualBox with Oracle 8.10
- 24 GB RAM, 8 CPUs, 75 GB Disk each
- Each node needs static IP configured
- Use personal NAT Network and forward inbound traffic to each VM (Host IP: Blank, Host Port: 2220, Guest IP 10.0.2.10, Guest Port: 22)

## 2. Kubernetes control node setup

Steps mentioned here must be done only on the conrol node.

### Disable swap memory
Kubernetes is still undergoing process to stablize the usage of swap. [Read this K8s blog](https://kubernetes.io/blog/2025/03/25/swap-linux-improvements/)
######
    sudo swapoff -a
    sudo sed -i '/ swap / s/^/#/' /etc/fstab

### Enable kernel modules and sysctl
######
    cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
    br_netfilter
    overlay
    EOF
    
    sudo modprobe br_netfilter
    sudo modprobe overlay
    
    cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
    net.bridge.bridge-nf-call-ip6tables = 1
    net.bridge.bridge-nf-call-iptables  = 1
    net.ipv4.ip_forward                 = 1
    EOF
    
    sudo sysctl --system


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
    sudo systemctl enable --now containerd

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
    sudo systemctl enable --now kubelet

### Initialize control plane
######
    sudo kubeadm init --pod-network-cidr=192.168.0.0/16

### Setup kubectl for the user
######
    mkdir -p $HOME/.kube
    sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config
    sudo chown $(id -u):$(id -g) $HOME/.kube/config

### Install CNI (Calico)
######
    kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/calico.yaml


## 3. Kubernetes worker node setup
First, include the IP address and hostname of the control plane and the worker node
Then execute same command from "Disable swap memory" to Enable Kubernetes repository
######
    sudo tee -a /etc/hosts << EOF
    10.0.2.10 control control.example.com
