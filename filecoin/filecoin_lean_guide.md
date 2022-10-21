# Filecoin 学习指南和资料整理

- [Filecoin 学习指南和资料整理](#filecoin-学习指南和资料整理)
  - [官方资料](#官方资料)
    - [白皮书](#白皮书)
    - [技术规格书](#技术规格书)
    - [开发文档](#开发文档)
  - [中文社区整理](#中文社区整理)
  - [其它资料](#其它资料)
  - [关于挖矿和矿池](#关于挖矿和矿池)
  - [源码解读](#源码解读)


首先申明，学习任何技术，都应该先以官方资料为主，其它资料作为辅助。还有看源码之前，先研究其设计文档，官方文档。

## 官方资料

学习思路：建议先看白皮书，然后技术规格书，然后看官方文档，最后再研究源码。

### 白皮书

Filecoin白皮书：https://gitee.com/xingzjx/blockchain/blob/master/filecoin/whitebook/filecoin_whitepaper_cn.md

白皮书共８个章节，其中，第三章介绍了复制证明和时空证明算法以及第五章的市场，是Filecoin的重要概念。


### 技术规格书

技术规格书：https://spec.filecoin.io/

技术规格书讲到了 Filecoin　的设计架构，以及一些核心概念和术语等，它共有８个章节。

- 第一章：简介，包括　Filecoin 设计以及子系统的概念
- 第二章：系统，讲解了节点、文件和数据、虚拟机、区块链、Token、存储挖矿和市场，７个子系统
- 第三章：核心库，介绍了Filecoin用到的一些库，比如libp2p,ipld等等
- 第四章：数据结构和算法，包括存储证明、共识机制、加密机制等等
- 第五章：术语表，包括订单、文件、创世区块等术语
- 第六章：附录
- 第七章：实现版本，包括四种实现：Lotus、Venus（原go-filecoin项目）、Forest、Fuhon(cpp-filecoin)
- 第七章：发布，规格书的版本号

### 开发文档

Lotus开发文档：https://docs.filecoin.io/

已迁移到：https://lotus.filecoin.io/lotus/get-started/what-is-lotus/

目前实现最完善的是Lotus，所以开发文档看Lotus的文档。

该文档包括七个章节，分别是:

- About Filecoin：　Filecoin 简介,包括章节等
- Get Started：包括 Lotus　节点的部署以及使用等
- Store：Filecoin存储部分
- Mine：挖矿，矿工必看的部分
- Netwokrs：Filecoin网络简介，包括Mainnet，Calibnet，Nerpanet
- Build：构建
- Reference：资料引用

## 中文社区整理

https://github.com/filecoin-project/community-china

[本地搭建 2K 测试网入门教程](https://github.com/filecoin-project/community-china/blob/master/documents/tutorial/local_2k_dev_tutorial/local_2k_dev_tutorial.md)

## 其它资料

包括网上个人以及第三方的学习笔记和实践经验

[石榴矿池运维开源方案](https://github.com/shannon-6block/lotus-miner)

[Filecoin运维系列(小一辈无产阶级码农)](https://www.r9it.com/)

[原语云lotus部署文档](https://docs.yycloud.pro/)

[StarLi_2020源码解读系列](https://blog.csdn.net/starli2020/category_9758801.html?spm=1001.2014.3001.5482)

[Filecoin — What’s Window PoST?-StarLi](https://starli.medium.com/filecoin-whats-window-post-7361bfbad755)

[石榴矿池李白youtube视频](https://www.youtube.com/channel/UCDPDdczcoVNNM-vWFxNT-9w/featured)

[Filecoin博客(weixin_46596227)](https://blog.csdn.net/weixin_46596227/category_11170000.html)

[如何搭建大规模Filecoin集群](https://mp.weixin.qq.com/s/koJ5QjOURT7ZS-WO60m-Dw)

[集群最大化利用资源增长算力方案](https://www.yuque.com/docs/share/8eae8a0b-e161-4a57-bc4c-11d7b2f13369)

[mixboot lotus 经验分享](https://blog.csdn.net/u010953692/category_9573142.html?spm=1001.2014.3001.5482)

## 关于挖矿和矿池

filpool 矿池

[Filecoin的矿池之路](https://www.jinse.com/news/blockchain/643989.html)

[关于Filecoin矿池的技术分享视频](https://www.bilibili.com/video/av840003306/)

[filecoin 单节点挖矿](https://nad128668.medium.com/comprehensive-guide-to-install-filecoin-mining-rig-8c95cb9613dc)

[币圈李白 如何做好Filecoin联合挖矿 20210921](https://youtu.be/jcJnwQcj1XQ)

## 源码解读

[lotus-4 代码详解之miner调度及work处理](https://blog.csdn.net/qingu/article/details/106693387)

[Filecoin任务调度算法详解](https://zhuanlan.zhihu.com/p/158701820)

[Filecoin源码分析(2)-- Filecoin的共识和出块原理解析](https://juejin.cn/post/6864831562882646024)