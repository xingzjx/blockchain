# 使用 kubeadm 搭建 kubernetes 本地集群

<!-- TOC -->

- [使用 kubeadm 搭建 kubernetes 本地集群](#使用-kubeadm-搭建-kubernetes-本地集群)
  - [安装环境](#安装环境)
  - [准备工作](#准备工作)
    - [更改 hostname](#更改-hostname)
    - [关闭防火墙](#关闭防火墙)
    - [关闭交互分区](#关闭交互分区)
    - [让 iptables 查看每个节点的桥接流量](#让-iptables-查看每个节点的桥接流量)
  - [配置 docker](#配置-docker)
  - [安装kubeadm, kubelet, kubectl](#安装kubeadm-kubelet-kubectl)
  - [配置 kubeconfig](#配置-kubeconfig)
  - [部署 calico](#部署-calico)
  - [节点加入集群](#节点加入集群)
  - [metrics-server](#metrics-server)
  - [dashboard](#dashboard)
  - [MetalLb](#metallb)

<!-- /TOC -->

## 安装环境

若干台 ubuntu 20.04 16G 32 核

## 准备工作

### 更改 hostname

```bash

hostnamectl set-hostname k8s-master

```

### 关闭防火墙

```bash

sudo ufw disable

```

### 关闭交互分区

```bash

sudo swapoff -a; sudo sed -i '/swap/d' /etc/fstab

```

### 让 iptables 查看每个节点的桥接流量

加载 br_netfilter

```bash

sudo modprobe br_netfilter

lsmod | grep br_netfilter

cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
> br_netfilter
> EOF

```

设置 net.bridge.bridge-nf-call-iptables

```bash

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
> net.bridge.bridge-nf-call-ip6tables = 1
> net.bridge.bridge-nf-call-iptables = 1
> EOF

```

```bash

sudo sysctl --system

```

## 配置 docker

```bash

sudo mkdir /etc/docker

cat <<EOF | sudo tee /etc/docker/daemon.json
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
EOF

```

重启加载配置

```bash

sudo systemctl enable docker
sudo systemctl daemon-reload
sudo systemctl restart docker

```

## 安装kubeadm, kubelet, kubectl

安装依赖包

```bash

sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl


```

添加 Kubernetes Apt 仓库

```bash

sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg


echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list

```

安装软件包

```bash

sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl

```

## 配置 kubeconfig

```bash

mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

```

## 部署 calico 

```bash

wget https://docs.projectcalico.org/v3.12/manifests/calico.yaml

```

添加 IP_AUTODETECTION_METHOD 字段

```yaml

    - name: CLUSTER_TYPE
        value: "k8s,bgp"
    # 新增部分
    - name: IP_AUTODETECTION_METHOD
        value: "interface=eno1"
    # value: "interface=eth.*"，　interface　是自己机器网卡的名字
    # value: "interface=can-reach=www.baidu.com"
    # 新增部分结束
    - name: IP
        value: "autodetect"
    - name: CALICO_IPV4POOL_IPIP
        value: "Always"

```

```bash

sudo kubectl apply -f calico.yaml

```

## 节点加入集群

在 master 节点获取 token

```bash

kubeadm token create --print-join-command

```

在 node 节点执行

```bash

kubeadm join xx.xx.xx.xx:6443 --token o4op56.sn6gkif90o54b8bb     --discovery-token-ca-cert-hash sha256:b48ac11cc495f4f0f78069a44e8674497b9d818f20338961f3c0502c6032db92

```

注意：在 node 节点加入集群之前，确保已经做好准备工作以及安装好 kubeadm 等工具。

以下使用阿里云的 add_node.sh 脚本：

```bash

#!/bin/bash

#关闭SWAP分区
sudo swapoff -a 

sudo sed -ri 's/.*swap.*/#&/' /etc/fstab

#关闭防火墙
sudo systemctl stop ufw

sudo systemctl disable ufw

#让 iptables 查看每个节点的桥接流量

# 加载 br_netfilter
sudo modprobe br_netfilter

lsmod | grep br_netfilter

cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
	br_netfilter
EOF

# 设置 net.bridge.bridge-nf-call-iptables

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
	net.bridge.bridge-nf-call-ip6tables = 1
	net.bridge.bridge-nf-call-iptables = 1
EOF

sudo sysctl --system

# 时间同步
sudo apt-get -y install ntpdate

sudo ntpdate time.windows.com

#++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++

# 全节点安装docker
sudo apt-get update

sudo apt-get -y install apt-transport-https ca-certificates curl software-properties-common  gnupg lsb-release

curl -fsSL http://mirrors.aliyun.com/docker-ce/linux/ubuntu/gpg | sudo apt-key add -

sudo add-apt-repository "deb [arch=amd64] http://mirrors.aliyun.com/docker-ce/linux/ubuntu $(lsb_release -cs) stable"

sudo apt-get update

sudo apt-get -y install docker-ce

sudo docker rmi -f $(sudo docker images -qa)

sudo docker rm -f $(sudo docker ps -qa)

# 配置docker

cat <<EOF | sudo tee /etc/docker/daemon.json
	{
	"exec-opts": ["native.cgroupdriver=systemd"],
	"registry-mirrors": ["https://b9pmyelo.mirror.aliyuncs.com"],
	"log-driver": "json-file",
	"default-address-pools":[{"base": "192.168.0.0/16", "size": 24}],
	"log-opts": {
		"max-size": "100m"
	},
	"storage-driver": "overlay2"
	}
EOF

sudo systemctl daemon-reload

sudo systemctl restart docker

sudo systemctl enable docker

sudo apt-get update

sudo apt-get -y install apt-transport-https ca-certificates curl

curl https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | sudo apt-key add -

cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
	deb https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main
EOF

sudo apt-get update

sudo apt-get install -y kubelet=1.20.0-00 && sudo apt-get install -y kubeadm=1.20.0-00 && sudo apt-get install -y kubectl=1.20.0-00

sudo systemctl start kubelet

sudo apt-mark hold kubelet kubeadm kubectl

```

## metrics-server 

安装

```bash

kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

```

配置

```bash

kubectl edit deploy -n kube-system metrics-server


```

```yaml

      containers:
      - args:
        - --cert-dir=/tmp
        - --secure-port=4443
        # Add this line
        - --kubelet-insecure-tls 
        - --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
        - --kubelet-use-node-status-port
        image: k8s.gcr.io/metrics-server/metrics-server:v0.4.2
```

```bash

kubectl get pod -n kube-system -w

kubectl top node

```

## dashboard

```bash

kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.2.0/aio/deploy/recommended.yaml

```

```bash

kubectl proxy

```

访问：http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/ 

创建用户

```bash

cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kubernetes-dashboard
EOF

cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kubernetes-dashboard
EOF

```

获取 token

```bash

kubectl -n kubernetes-dashboard describe secret $(kubectl -n kubernetes-dashboard get secret | grep admin-user | awk '{print $1}')

```

## MetalLb

安装

```bash

kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.9.6/manifests/namespace.yaml
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.9.6/manifests/metallb.yaml
# On first install only
kubectl create secret generic -n metallb-system memberlist --from-literal=secretkey="$(openssl rand -base64 128)"

```

配置节点 ip , 作为负载均衡的地址

```bash

kubectl apply -f - <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: metallb-system
  name: config
data:
  config: |
    address-pools:
    - name: default
      protocol: layer2
      addresses:
      - 162.211.182.45-162.211.182.48 # Update this with your Nodes IP range
EOF

```