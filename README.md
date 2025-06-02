## This repo contains the files and explanations to setup Kubernetes cluster for SAS Viya from scratch.

## Required Steps
1. Environment setup
2. Kubernetes control plane node setup
3. Kubernetes worker node setup
4. Deploy SAS Viya from control plane node

## 1. Environment setup
- RACE Image ID: 1439134
- 1 Control Node, 5 Worker Nodes on VirtualBox with Oracle 8.10
- 24 GB RAM, 8 CPUs, 75 GB Disk each
- Each node needs static IP configured
- Use personal NAT Network and forward inbound traffic to each VM (Host IP: Blank, Host Port: 2220, Guest IP 10.0.2.10, Guest Port: 22)

## 2. Kubernetes control node setup

### Add the hostname and IP address of all control nodes and worker nodes
######
    sudo tee -a /etc/hosts << EOF
    10.0.2.10 control control.example.com
    10.0.2.11 worker1 worker1.example.com
    10.0.2.12 worker2 worker1.example.com
    10.0.2.13 worker3 worker1.example.com
    10.0.2.14 worker4 worker1.example.com
    10.0.2.15 worker5 worker1.example.com
    EOF

### Disable swap memory
Kubernetes is still undergoing process to stablize the usage of swap. [Read this K8s blog](https://kubernetes.io/blog/2025/03/25/swap-linux-improvements/)
######
    sudo swapoff -a
    sudo sed -i '/ swap / s/^/#/' /etc/fstab

### Add a namespace server if there is none
######
    sudo tee /etc/resolv.conf << EOF
    search race.sas.com example.com
    nameserver 10.19.1.23
    nameserver 10.36.1.53
    nameserver 10.19.1.24
    namespace 8.8.8.8
    namespace 8.8.4.4
    EOF


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

### Get the join command to join all worker nodes
######
    kubeadm token create --print-join-command

### After joining all worker nodes, confirm with the following command
######
    kubectl get nodes

Expected output
######
    [control@control ~]$ kubectl get nodes
    NAME                  STATUS   ROLES           AGE     VERSION
    control.example.com   Ready    control-plane   30h     v1.27.16
    worker1.example.com   Ready    <none>          18s     v1.27.16
    worker2.example.com   Ready    <none>          2m13s   v1.27.16
    worker3.example.com   Ready    <none>          2m1s    v1.27.16
    worker4.example.com   Ready    <none>          107s    v1.27.16
    worker5.exmaple.com   Ready    <none>          103s    v1.27.16

### Enable autofill for kubectl commands
    sudo yum install bash-completion -y
    echo 'source <(kubectl completion bash)' >> ~/.bashrc
    source ~/.bashrc

### Install K9s for monitoring
    curl -Lo k9s.tar.gz https://github.com/derailed/k9s/releases/latest/download/k9s_Linux_amd64.tar.gz
    tar -xzf k9s.tar.gz
    sudo mv k9s /usr/local/bin/

### Install metrics server
    kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

    kubectl patch deployment metrics-server -n kube-system \
      --type='json' \
      -p='[{"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--kubelet-insecure-tls"},{"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--kubelet-preferred-address-types=InternalIP,Hostname"}]'
    
    kubectl rollout restart deployment metrics-server -n kube-system


### Configure by applying the MetalLB yaml file
######
    kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.14.8/config/manifests/metallb-native.yaml

### Check if the pods are running
######
    kubectl get pods -n metallb-system

### Create yaml files to configure L2 advertisement IP pool
    cat <<EOF> ippool.yaml
    apiVersion: metallb.io/v1beta1
    kind: IPAddressPool
    metadata:
      name: first-pool
      namespace: metallb-system
    spec:
      addresses:
      - 10.0.2.20-10.0.2.30
      autoAssign: true
    EOF
    
    cat <<EOF> l2.yaml
    ---
    apiVersion: metallb.io/v1beta1
    kind: L2Advertisement
    metadata:
      name: lb-pool
      namespace: metallb-system
    spec:
      ipAddressPools:
      - first-pool
    EOF

### Apply the yaml files 
    kubectl apply -f ippool.yaml
    kubectl apply -f l2.yaml

### Change the config map and allow strictARP
    kubectl get configmap kube-proxy -n kube-system -o yaml | \
    sed -e "s/strictARP: false/strictARP: true/" | \
    kubectl apply -f - -n kube-system

### Configure ingress-nginx with a yaml file
    kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.10.1/deploy/static/provider/cloud/deploy.yaml

### Configure nfs
    sudo yum install nfs-utils -y
    sudo systemctl enable --now nfs-server
    sudo mkdir -p /nfs/

### Edit the following files
    sudo nano /etc/exports
######
    sudo nano /etc/exports.d/kubernetes.exports

Include this line
######
    /nfs *(rw,sync,no_root_squash,insecure,no_subtree_check,nohide)

### Enable nfs
    sudo systemctl enable --now nfs-server
    sudo exportfs -avrs
    showmount -e control.example.com

### Configure nfs on Kubernetes
    curl -skSL https://raw.githubusercontent.com/kubernetes-csi/csi-driver-nfs/v4.8.0/deploy/install-driver.sh | bash -s v4.8.0 --
    kubectl -n kube-system get pod -o wide -l app=csi-nfs-controller
    kubectl -n kube-system get pod -o wide -l app=csi-nfs-node

### Configure a storage class
    cat <<EOF> storageclass-configure.yaml
    apiVersion: storage.k8s.io/v1
    kind: StorageClass
    metadata:
      name: default
      annotations:
        storageclass.kubernetes.io/is-default-class: "true"
    provisioner: nfs.csi.k8s.io
    parameters:
      server: control.example.com
      share: /nfs
    reclaimPolicy: Delete
    volumeBindingMode: Immediate
    EOF
######
    kubectl apply -f storageclass-configure.yaml
    kubectl get sc

## 3. Kubernetes worker node setup
First, include the IP address and hostname of the control plane and the worker node.
######
    sudo tee -a /etc/hosts << EOF
    $(hostname -I) $(hostname -s) $(hostname -f)
    10.0.2.10 control control.example.com
    EOF


Then execute same command from "Disable swap memory" to "Enable Kubernetes repository"
### Install Kubernetes binaries except kubectl
######
    sudo yum install --disableexcludes=kubernetes kubeadm kubelet -y
    sudo systemctl enable --now kubelet

### Join the worker node to the control node
    sudo kubeadm join command here

### Install nfs
    sudo yum install nfs-utils -y

Full code for worker node setup
######
    sudo swapoff -a
    sudo sed -i '/ swap / s/^/#/' /etc/fstab

    sudo tee -a /etc/hosts << EOF
    10.0.2.10 control control.example.com
    10.0.2.11 worker1 worker1.example.com
    10.0.2.12 worker2 worker1.example.com
    10.0.2.13 worker3 worker1.example.com
    10.0.2.14 worker4 worker1.example.com
    10.0.2.15 worker5 worker1.example.com
    EOF

    sudo tee /etc/resolv.conf << EOF
    search race.sas.com example.com
    nameserver 10.19.1.23
    nameserver 10.36.1.53
    nameserver 10.19.1.24
    namespace 8.8.8.8
    namespace 8.8.4.4
    EOF

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

    sudo yum install -y yum-utils
    sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo

    sudo yum install -y containerd.io

    sudo mkdir -p /etc/containerd
    containerd config default | sudo tee /etc/containerd/config.toml > /dev/null
    sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
    sudo systemctl enable --now containerd

    sudo tee /etc/yum.repos.d/kubernetes.repo << EOF
    [kubernetes]
    name=Kubernetes
    baseurl=https://pkgs.k8s.io/core:/stable:/v1.27/rpm/
    enabled=1
    gpgcheck=1
    gpgkey=https://pkgs.k8s.io/core:/stable:/v1.27/rpm/repodata/repomd.xml.key
    exclude=kubelet kubeadm kubectl cri-tools kubernetes-cni
    EOF

    sudo yum install --disableexcludes=kubernetes kubeadm kubelet -y
    sudo systemctl enable --now kubelet

## 4. Deploy SAS Viya from control plane node
First, get the order files from my.sas.com and unzip it. 
######
Then create the namespace to deploy SAS Viya
######
    kubectl create namespace viya

Untaint the control node so that it can deploy resources as well
######
    kubectl taint nodes control.example.com node-role.kubernetes.io/control-plane-

Enable nginx.ingress.kubernetes.io/configuration-snippet
######
    kubectl get configmap ingress-nginx-controller -n ingress-nginx -o yaml | \
    sed 's/allow-snippet-annotations: "false"/allow-snippet-annotations: "true"/' | \
    kubectl apply -f - -n ingress-nginx

### Install kustomize
    curl -s "https://api.github.com/repos/kubernetes-sigs/kustomize/releases/latest" \
    | grep browser_download_url \
    | grep linux_amd64 \
    | cut -d '"' -f 4 \
    | xargs curl -LO
    
    tar -xzf kustomize_v*_linux_amd64.tar.gz
    sudo mv kustomize /usr/local/bin/
    
    kustomize version

### Create initial kustomization.yaml
######
    sudo tee kustomization.yaml << EOF
    namespace: viya
    resources:
    - sas-bases/base
    - sas-bases/overlays/network/networking.k8s.io
    - site-config/security/openssl-generated-ingress-certificate.yaml
    - sas-bases/overlays/cas-server
    - sas-bases/overlays/crunchydata/postgres-operator
    - sas-bases/overlays/postgres/platform-postgres
    # If your deployment contains SAS Viya Programming, comment out the next line
    - sas-bases/overlays/internal-elasticsearch
    - sas-bases/overlays/update-checker
    - sas-bases/overlays/cas-server/auto-resources
    configurations:
    - sas-bases/overlays/required/kustomizeconfig.yaml
    transformers:
    # If your deployment does not support privileged containers or if your deployment
    # contains SAS Viya Programming, comment out the next line
    - sas-bases/overlays/internal-elasticsearch/sysctl-transformer.yaml
    - sas-bases/overlays/required/transformers.yaml
    - sas-bases/overlays/cas-server/auto-resources/remove-resources.yaml
    # If your deployment contains SAS Viya Programming, comment out the next line
    - sas-bases/overlays/internal-elasticsearch/internal-elasticsearch-transformer.yaml
    # Mount information
    # - site-config/{{ DIRECTORY-PATH }}/cas-add-host-mount.yaml
    components:
    - sas-bases/components/crunchydata/internal-platform-postgres
    - sas-bases/components/security/core/base/full-stack-tls
    - sas-bases/components/security/network/networking.k8s.io/ingress/nginx.ingress.kubernetes.io/full-stack-tls
    patches:
    - path: site-config/storageclass.yaml
      target:
        kind: PersistentVolumeClaim
        annotationSelector: sas.com/component-name in (sas-backup-job,sas-data-quality-services,sas-commonfiles,sas-cas-operator,sas-pyconfig,sas-risk-cirrus-search,sas-risk-modeling-core,sas-event-stream-processing-studio-app)
    # License information
    # secretGenerator:
    # - name: sas-license
    #   type: sas.com/license
    #   behavior: merge
    #   files:
    #   - SAS_LICENSE=license.jwt
    configMapGenerator:
    - name: ingress-input
      behavior: merge
      literals:
      - INGRESS_HOST=trck1051917.trc.sas.com
    - name: sas-shared-config
      behavior: merge
      literals:
      - SAS_SERVICES_URL=https://trck1051917.trc.sas.com
      # - SAS_URL_EXTERNAL_VIYA={{ EXTERNAL-PROXY-URL }}
    secretGenerator:
    - name: sas-consul-config            ## This injects content into consul. You can add, but not replace
      behavior: merge
      files:
        - SITEDEFAULT_CONF=site-config/sitedefault.yaml
    EOF

### Build site.yaml file
    kustomize build -o site.yaml

### Apply cluster-api resources to the cluster. As an administrator with cluster permissions, run
    kubectl apply --selector="sas.com/admin=cluster-api" --server-side --force-conflicts -f site.yaml
    kubectl wait --for condition=established --timeout=60s -l "sas.com/admin=cluster-api" crd

### As an administrator with cluster permissions, run
    kubectl apply --selector="sas.com/admin=cluster-wide" -f site.yaml

### As an administrator with local cluster permissions, run
    kubectl apply --selector="sas.com/admin=cluster-local" -f site.yaml --prune

### As an administrator with namespace permissions, run
    kubectl apply --selector="sas.com/admin=namespace" -f site.yaml --prune
