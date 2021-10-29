- Criação de três VMs com as configurações:
  - AlmaLinux 8.4 Minimal
  - 2 CPU
  - 2 GB RAM
  - Bridged Adapter
- Config:
  - `/etc/sysconfig/network-scripts/ifcfg-enp0s3`
    - k8s-main
      ```bash
      BOOTPROTO=static
      IPADDR=192.168.0.199
      NETMASK=255.255.255.0
      GATEWAY=192.168.0.1
      DNS1=192.168.0.1
      ```
    - k8s-node-01
      ```bash
      BOOTPROTO=static
      IPADDR=192.168.0.198
      NETMASK=255.255.255.0  
      GATEWAY=192.168.0.1
      DNS1=192.168.0.1
      ```
    - k8s-node-02
      ```bash
      BOOTPROTO=static
      IPADDR=192.168.0.197
      NETMASK=255.255.255.0
      GATEWAY=192.168.0.1
      DNS1=192.168.0.1
      ```
  - /etc/hostname
    - k8s-main
      ```bash
      k8s-main.localdomain
      ```
    - k8s-node-01
      ```bash
      k8s-node-01.localdomain
      ```
    - k8s-node-02
      ```bash
      k8s-node-02.localdomain
      ```
  - /etc/hosts
    ```bash
    192.16.0.199    k8s-main k8s-main.localdomain
    192.16.0.198    k8s-node-01 k8s-node-01.localdomain
    192.16.0.197    k8s-node-02 k8s-node-02.localdomain
    ```
- Instalação do Kubernetes
  - k8s-main, k8s-node-01, k8s-node-02
    ```bash
    sed -i --follow-symlinks 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/sysconfig/selinux
    systemctl disable firewalld

    dnf config-manager --add-repo=https://download.docker.com/linux/centos/docker-ce.repo
    dnf install -y containerd.io docker-ce
    systemctl enable --now docker
    cat <<EOF > /etc/docker/daemon.json
    {
      "exec-opts": ["native.cgroupdriver=systemd"]
    }
    EOF
    systemctl restart docker

    sed -i '/ swap / s/^/#/' /etc/fstab
    swapoff -a

    cat <<EOF > /etc/yum.repos.d/kubernetes.repo
    [kubernetes]
    name=Kubernetes
    baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
    enabled=1
    gpgcheck=1
    repo_gpgcheck=1
    gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
    EOF
    dnf install kubeadm iproute-tc -y
    systemctl enable kubelet

    reboot
    ```
  - k8s-main
    ```bash
    kubeadm init
    mkdir -p $HOME/.kube
    cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
    chown $(id -u):$(id -g) $HOME/.kube/config
    export kubever=$(kubectl version | base64 | tr -d '\n')
    kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$kubever"
    ```
  - k8s-node-01, k8s-node-02
    ```bash
    kubeadm join 192.168.0.199:6443 --token 11ofzn.c8ppstvgfeubnkcv \
    --discovery-token-ca-cert-hash sha256:a841c2b3ad31b6607a1f178ec9b0c0ee1f3b64685d0ad3801bb6b6ea2616b16b
    ```
  - k8s-main
    ```bash
    kubectl get nodes
    ```
