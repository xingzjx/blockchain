<h1>挖矿学习指南</h1>

# 挖矿原理

&emsp;&emsp;所谓的挖矿，其实就是利用计算机的资源比如CPU、GPU、FPGA、ASIC等，计算哈希值，然后验证获取奖励的过程。常见的挖矿哈希算法有Sha256(BTC)，Ethash，Equihash(ZEC)，Scrypt（LTC）等等。

&emsp;&emsp;注意：计算哈希值是其中一种挖矿方式，还有基于poc的存储证明挖矿，比如filecoin和chia等。本文只讨论哈希算法挖矿相关内容。

参考：

[哈希算法常见参考](https://github.com/Lolliedieb/lolMiner-releases)

[精通比特币：挖矿和共识](https://github.com/xingzjx/MasterBitcoin2CN/blob/master/ch10.md)

[CUDA与OpenCL架构](https://www.cnblogs.com/huliangwen/articles/5003504.html)

[CPU、GPU、FPGA、ASIC，区块链挖矿技术哪家强](https://cloud.tencent.com/developer/article/1559977)

[深入理解CPU和异构计算芯片GPU/FPGA/ASIC](https://zhuanlan.zhihu.com/p/25996962)

# 挖矿工具

&emsp;&emsp;unmineable是一个图形界面挖矿工具，支持windows和Linux，它需要配和其它命令行工具，比如PhoenixMiner，lolMiner，XMRIG等等。该工具会根据网络情况自动选择矿池服务器的地址。
&emsp;&emsp;挖矿工具要做的事情，就是采用不同的挖矿算法，然后提交其哈希值到矿池或者全节点。

参考：

[unmineable挖矿](https://www.unmineable.com/coins/)

# 开源挖矿客户端工具

参考：

[XMRig](https://github.com/xmrig/xmrig)

[ethminer](https://github.com/ethereum-mining/ethminer)



# btcpool矿池源码

参考：

[btcpool-ABANDONED](https://github.com/btccom/btcpool-ABANDONED)


# ethereum矿池源码

参考：

[open-ethereum-pool](https://github.com/sammy007/open-ethereum-pool)