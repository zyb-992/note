# 使用Kubeadm

## 什么是Kubeadm

1. `Kubernetes`是由很多模块构成的，而实现核心功能的组件，如apiserver、etcd、controller-manager、scheduler，其本质都是可执行文件，我们可以使用Shell脚本等将所有组件安装好打包到服务器上配置运行，但这些组件之间的配置关系十分复杂，搭建集群的过程也很麻烦，为了简化kubernetes的部署工作，我们便可以使用**`kubeadm`**来在集群中安装`Kubernetes`
2. `kubeadm`的原理也和`minikube`类似，也是使用容器和镜像来封装`Kubernetes`的上述各种组件，它的目标不是单机部署，而是能够轻松地在集群环境里部署Kubernetes，并让集群接近甚至达到生产级质量

## 安装Kubeadm 

```shell
#!/bin/bash

cat <<EOF | sudo tee /etc/docker/daemon.json
{
  "registry-mirrors": ["https://9q1nmamk.mirror.aliyuncs.com"]
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
EOF

sudo systemctl enable docker
sudo systemctl daemon-reload
sudo systemctl restart docker


# modify the config of iptables
# open the modules of br_netfilter

cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
br_netfilter
EOF

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward=1 # better than modify /etc/sysctl.conf
EOF

sudo sysctl --system


# close the swap of linux
sudo swapoff -a
sudo sed -ri '/\sswap\s/s/^#?/#/' /etc/fstab



# install the kubeadm

sudo apt install -y apt-transport-https ca-certificates curl

curl https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | sudo apt-key add -

cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main
EOF

sudo apt update


# install the kubeadm kubelet kubectl
sudo apt install -y kubeadm=1.23.3-00 kubelet=1.23.3-00 kubectl=1.23.3-00

# lock the version of these exe
sudo apt-mark hold kubeadm kubelet kubectl
```

```shell
# 查看kubeadm、kubectl是否成功安装
kubeadm version 
kubectl version --short
```



## 使用Kubeadm搭建Kubernetes集群

### 

```shell
# init
sudo kubeadm init --apiserver-advertise-address=192.168.137.149 --image-repository registry.aliyuncs.com/google_containers --kubernetes-version v1.23.3 --pod-network-cidr=10.10.0.0/16

# run the following as a regular user

# 
```

<img src="D:\Program Files\电子书\go\md\图片\image-20230214173125684.png" style="zoom: 67%;" />

```shell
kubeadm join 192.168.137.149:6443 --token 0ekp4w.bqm2yqaupj67gjtw \
--discovery-token-ca-cert-hash sha256:7d776753d571fb7eb4eda38dc570e996655882ed6526eb266c7415c3d3b42119 
```

