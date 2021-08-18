# Ubuntu 下部署 FileCoin 集群挖矿

## 准备工作

运行 lotus 节点需要8-core CPU and 32 GiB RAM

参考配置：

https://docs.filecoin.io/get-started/lotus/installation/#minimal-requirements

如果是挖矿节点，参考配置：


准备三台机器，一台运行lotus节点，一台运行miner节点，一台运行wokrer

## 部署lotus

在第一台机器

### 安装

安装依赖

```bash 

sudo apt install mesa-opencl-icd ocl-icd-opencl-dev gcc git bzr jq pkg-config curl clang build-essential hwloc libhwloc-dev wget -y && sudo apt upgrade -y

```

然后建立软连接

```bash

ln -s /usr/lib/x86_64-linux-gnu/libhwloc.so.15 /usr/lib/x86_64-linux-gnu/libhwloc.so.5

```

然后把自己构建的二进制可执行文件，包括lotus、lotus-miner、lotus-worker　拷贝到 /usr/local/bin 目录，注意后面的两台机器都这么需要这个步骤。


再添加执行权限

```

sudo chmod +x /usr/local/bin/lotus*

```

### 定义环境变量

编辑用户目录的.bashrc文件，添加配置：

```bashrc

export LOTUS_SKIP_GENESIS_CHECK=_yes_
export IPFS_GATEWAY=https://proof-parameters.s3.cn-south-1.jdcloud-oss.com/ipfs/
export LOTUS_PATH=/data/data1/filecoin/devnet/lotus
export FIL_PROOFS_PARAMETER_CACHE=/data/data1/filecoin/devnet/parameter_cache
export FIL_PROOFS_PARENT_CACHE=/data/data1/filecoin/devnet/parent_cache

```

记得使配置文件生效

```bash
source ~/.bashrc
```

### 启动lotus

```bash

lotus daemon >> ~/log/lotus.log 2>&1 &

```

### 等待同步结果

```bash 

lotus sync wait

```

## 部署miner 

在第二台机器

### 定义环境变量

编辑用户目录的.bashrc文件，添加配置：

```bashrc

export LOTUS_SKIP_GENESIS_CHECK=_yes_
export IPFS_GATEWAY=https://proof-parameters.s3.cn-south-1.jdcloud-oss.com/ipfs/
export RUST_BACKTRACE=full
export RUST_LOG=Trace
export LOTUS_MINER_PATH=/data/data1/filecoin/devnet/miner
export BELLMAN_CPU_UTILIZATION=0.875
export FIL_PROOFS_MAXIMIZE_CACHING=1
export FIL_PROOFS_USE_GPU_COLUMN_BUILDER=1
export FIL_PROOFS_USE_GPU_TREE_BUILDER=1
export FIL_PROOFS_PARAMETER_CACHE=/data/data1/filecoin/devnet/parameter_cache
export FIL_PROOFS_PARENT_CACHE=/data/data1/filecoin/devnet/parent_cache
export TMPDIR=/data/data1/filecoin/devnet/tmpdir
export FIL_PROOFS_USE_MULTICORE_SDR=1

```
记得使配置文件生效

```bash
source ~/.bashrc
```

### 安装

拷贝lotus-miner到　/usr/local/bin，并且添加执行权限

### 准备钱包

```bash

lotus wallet new bls

```

给钱包发送代币

```bash

lotus wallet send xxxxxxxx

```

### 初始化

如果是开发网络，定义：--sector-size=2KiB，如果是mainnet或者calibnet，则--sector-size=３2GiB

```bash

lotus-miner init --sector-size=2KiB  --owner=t3qbvcjx52r7426qiqphxzjtpktnzi5f2v4kc6bptipbwyp3pok6stijtgi63huobvw6l2ugqzqprihrhef6xa  --worker=t3qbvcjx52r7426qiqphxzjtpktnzi5f2v4kc6bptipbwyp3pok6stijtgi63huobvw6l2ugqzqprihrhef6xa

```

### 编辑配置文件

```bash

vim $LOTUS_MINER_PATH/config.toml

```

编辑配置文件，保存

```toml
[API]
  ListenAddress = "/ip4/127.0.0.1/tcp/2345/http"
  RemoteListenAddress = "127.0.0.1:2345"


[Storage]
  AllowAddPiece = false
  AllowPreCommit1 = false
  AllowPreCommit2 = false
  AllowCommit = false
  AllowUnseal = false

```

如果定义为true，则会使用当前miner分配密封任务

127.0.0.1改成本机ip地址，如果是局域网则内网ip,如果是公网则公网ip。


### 启动矿工

```bash
lotus-miner run --nosync >> ~/log/miner.log 2>&1 &
```

## 部署worker

在另外一台机器或者lotus所在机器都可以启动worker，本例以另外一台机器为例，也就是第三台机器。

### 


### 定义环境变量

在　miner　所在机器获取　MINER_API_INFO　变量

```bash
lotus-miner auth api-info --perm admin
```

在worker机器，按照部署lotus的步骤，确保准备好了lotus-worker的运行环境。

配置好环境变量

```bashrc
export MINER_API_INFO=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJBbGxvdyI6WyJyZWFkIiwid3JpdGUiLCJzaWduIiwiYWRtaW4iXX0.ERQweeN9CPFQA8eHG9-MBK_dQtiuucuRIl-hDPnlrcA:/ip4/xx.xx.xx.xx/tcp/2345/http
export LOTUS_SKIP_GENESIS_CHECK=_yes_
export IPFS_GATEWAY=https://proof-parameters.s3.cn-south-1.jdcloud-oss.com/ipfs/
export RUST_BACKTRACE=full
export RUST_LOG=Trace
export LOTUS_WORKER_PATH=/data/data1/filecoin/devnet/worker
export BELLMAN_CPU_UTILIZATION=0.875
export FIL_PROOFS_MAXIMIZE_CACHING=1
export FIL_PROOFS_USE_GPU_COLUMN_BUILDER=1
export FIL_PROOFS_USE_GPU_TREE_BUILDER=1
export FIL_PROOFS_PARAMETER_CACHE=/data/data1/filecoin/devnet/parameter_cache
export FIL_PROOFS_PARENT_CACHE=/data/data1/filecoin/devnet/parent_cache
export TMPDIR=/data/data1/filecoin/devnet/tmpdir
export FIL_PROOFS_USE_MULTICORE_SDR=1

```

记得使配置文件生效

```bash
source ~/.bashrc
```

### 安装

拷贝lotus-worker到　/usr/local/bin，并且添加执行权限

### 启动worker

```bash
lotus-worker run --addpiece=true --precommit1=true --unseal=true --precommit2=true --commit=true  >> ~/log/worker.log 2>&1 &
```

### 质押扇区

回到miner所在机器，

```bash
lotus-miner sectors pledge
```

