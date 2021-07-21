# Filecoin 源码导读

## 前言

Filecoin 源码导读主要从源码的目录设计，以及项目中的架构等，从整体设计的角度，然后，分解成每一个点，从而分析其中的源码。也就是从整体到局部的分析思路。

## 准备工作

在源码学习之前，建议先看官方的白皮书，官方文档，设计架构以及dapp应用等。

- 白皮书：https://gitee.com/xingzjx/blockchain/blob/master/filecoin/filecoin%20whitepaper%20cn.md

- Filecoin Spec：https://spec.filecoin.io/

设计文档，从设计的角度，分别介绍数据结构，挖矿机制，共识机制，支付方式，虚拟机执行，状态机，存储角色等等。

- CODEWALK：　https://github.com/filecoin-project/go-filecoin/blob/master/CODEWALK.md

CODEWALK讲述了Filecoin的代码历史，框架，以及各个模块的功能。

- Filecoin 节点部署：https://gitee.com/xingzjx/blockchain/blob/master/filecoin/filecoin%20learn%20guide.md

- Filecoin 编译：https://gitee.com/xingzjx/blockchain/blob/master/filecoin/mac_compile_deploy_filecoin.md

## 代码结构

在工程目录执行命令

```bash

tree -L 1

```

### 一级目录结构

```
├── api 包括json-rpc接口，以及cli也会调用到该接口
├── build　
├── chain　实现与链的交互功能
├── CHANGELOG.md
├── cli　命令行工具，在部署全节点的linux系统执行
├── cmd　程序主函数的入口，包括lotus，miner和worker等
├── conformance
├── documentation
├── extern　外部库，里面filecoin-ffi是rust语言实现的共识证明
├── gen
├── genesis
├── go.mod
├── go.sum
├── journal
├── lib　实现lotus项目各模块公用的函数
├── LICENSE-APACHE
├── LICENSE-MIT
├── lotuspond
├── Makefile
├── markets
├── metrics
├── miner　定义产出区块逻辑，与全节点通过API进行交互
├── node　定义了lotus节点相关的struct和interface等，包括hello协议以及节点实现
├── paychmgr
├── README.md
├── scripts
├── SECURITY.md
├── storage　定义存储矿工逻辑，用于实现"lotus-storage-miner"
├── sumary_struct.sh
├── testplans
└── tools
```

### cmd　模块

在 go　语言的工程目录中，cmd　目录一般作为程序的入口

```
├── chain-noise
├── lotus 守护进程，负责公链上数据的同步，与公链进行交互等，是lotus的主要进程之一
├── lotus-bench
├── lotus-chainwatch
├── lotus-fountain
├── lotus-gateway
├── lotus-health
├── lotus-keygen
├── lotus-offline-wallet
├── lotus-pcr
├── lotus-seal-worker　密封数据进程，密封数据是挖矿过程中必不可少的一环，本进程即实现此功能
├── lotus-seed
├── lotus-shed
├── lotus-stats
├── lotus-storage-miner　挖矿进程，打包信息到区块，存储矿工
├── lotus-townhall
├── lotus-wallet
└── tvx

```
### extern 模块

这个目录主要是外部依赖，和主工程lotus的代码不在一个工程

```

├── blst
├── filecoin-ffi 用cgo和rust实现的证明算法
├── sector-storage
├── serialization-vectors
├── storage-sealing
└── test-vectors

```

### node 模块

```
├── builder.go
├── config
├── fxlog.go
├── hello　hello协议实现
├── impl
├── modules 子模块实现，包括p2p实现
├── node_test.go
├── options.go
├── repo
├── test
└── testopts.go

```

## 核心技术

### libp2p

### zksnark

### 共识算法


参考：

[filecoin技术架构分析之四：filecoin源码顶层架构分析](https://blog.csdn.net/qq_21393091/article/details/88073352)

[Filecoin逻辑梳理及源代码导读](https://mp.weixin.qq.com/s?__biz=MzU5MzMxNTk2Nw==&mid=2247484751&idx=1&sn=54c2239d9f59c539a1a3c569362da288&chksm=fe13145fc9649d4950225d85153e68f8d08383a20d836a8edf25cc3411b224d736375288bf26&scene=21#wechat_redirect)

[FileCoin 挖矿教程 （七）Lotus 源码大致的结构目录](https://blog.csdn.net/weixin_46596227/article/details/118345766)