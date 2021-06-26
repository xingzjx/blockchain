<h1>FileCoin学习指南</h>

# 导读


参考：

[filecoin白皮书中文版](https://gitee.com/xingzjx/blockchain/blob/master/filecoin%20whitepaper%20cn.md)
[部署和开发官方文档](https://docs.filecoin.io/)

# 部署
## 云端部署

- DigitalOcean

- Amazon Web Services

## 本地部署

### 检查机器配置

运行 lotus 节点需要8-core CPU and 32 GiB RAM

参考配置：

https://docs.filecoin.io/get-started/lotus/installation/#minimal-requirements

如果是挖矿节点，参考配置：


### 安装

安装依赖

```bash 

sudo apt install mesa-opencl-icd ocl-icd-opencl-dev gcc git bzr jq pkg-config curl clang build-essential hwloc libhwloc-dev wget -y && sudo apt upgrade -y


```

然后建立软连接

```bash

ln -s /usr/lib/x86_64-linux-gnu/libhwloc.so.15 /usr/lib/x86_64-linux-gnu/libhwloc.so.5

```

进入：https://github.com/filecoin-project/lotus/releases 页面下载版本 lotus_v1.10.0_linux-amd64.tar.gz

解压缩到 /usr/local/bin 目录。

```bash 

sudo tar -xzf lotus_v1.10.0_linux-amd64.tar.gz -C /usr/local/bin/

```

### 启动lotus

```bash

lotus daemon

```

### 查看同步状态

```bash 

lotus sync status

```

[file coin 单节点挖矿](https://nad128668.medium.com/comprehensive-guide-to-install-filecoin-mining-rig-8c95cb9613dc)

# 挖矿和矿池

filpool 矿池

[Filecoin的矿池之路](https://www.jinse.com/news/blockchain/643989.html)

[关于Filecoin矿池的技术分享视频](https://www.bilibili.com/video/av840003306/)

# 前端交互（build）

# 源码解读

