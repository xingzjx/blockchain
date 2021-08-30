# NBZZ　部署教程

## 部署bee节点

### 官方部署教程

https://docs.ethswarm.org/docs/installation/install

### docker-compose

[docker-compose批量部署](./ubuntu_docker-compose_multi_swarm.md)

### k8s集群部署

[helm部署bee节点](https://github.com/ethersphere/helm/tree/master/charts/bee)

##　安装nbzz

### 单节点部署

```bash

sudo apt update
sudo apt upgrade -y

# Install Git
sudo apt install git -y

# Checkout the source and install
git clone https://github.com/LeetSquad/Swarm-nbzz.git nbzz
cd nbzz

sh install.sh

. ./activate

```

然后，初始化，

```bash

nbzz init

```

进入bee节点私钥目录，执行如下命令：

```bash

nbzz faucet -p you-bee-password

nbzz pledge -p you-bee-password 

nbzz start -p you-bee-password

```

注意：you-bee-password　是部署bee节点时设置的密码

### 批量化部署

执行命令：

```bash

nbzz_install.sh

```

改命令会安装nBZZ以及启动nBZZ节点。

工程脚本：https://gitee.com/xingzjx/nbzz_install