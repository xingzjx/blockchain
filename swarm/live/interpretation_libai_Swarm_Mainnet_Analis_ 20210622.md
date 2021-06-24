# 整理：币圈李白 Swarm主网技术分析 20210622


视频出处：https://www.youtube.com/watch?v=X1N48abLmdo



## 技术要点

包括抵押、空投、xDai、Time-based Settlement、Bonding curve vending machine等。

### 抵押

- 官方在AMA中说可以无checkquebook启节点
- v1.0.0版本并不能做到这点？ 待验证
- Discord问了没有答复
- 理论上也不成立
  无限起新的节点，白嫖下载流量
- 所以肯定有抵押

### 空投

- 按支票算账
- Queeen节点不再随机
- aBZZ是为了有个申诉的机会
- aBZZ可以转账
- xDAI空投省GAS

### xDAI
  
优点：省钱
  
- 空投、起节点、cashout，gas低
- 猜测有其它联系

缺点：

- 功能重叠：SWAP 与 xDAI 矛盾
- 双币始终不友好
- 说好的作为ETH的功能扩展呢
  
### Time-based Settlement

- 流量被压缩，节点收入进一步下降
- 临时上线，未经充分测试，也没有说明文档
- 设计并不完美：免费下载是需求，不能降低节点收益
  
### Bonding Curve Vending Machine


- 作用是稳定币价：价格太高影响使用
- 唯一的增发渠道：没有挖矿机制
- 没有文档说明：各处的参数不一样。 Coinlist，Github，Ehtherscan，bzz exchange


### 总结

- 上线匆忙
- 上起来节点收益低
- 看私募搏杀一个月

## 分析解读

v1.0.0是否需要抵押待验证。

抵押 bzz 从直播视频来看尚不明确。

xDai 是以太坊的 Lay2 方案，bzz是发行在一层网络，也就是以太坊主网，lay2是可以切回到以太坊主网的。但是，bsc 等虽然在以太坊源码改进，但是不是 layer2 方案，所以bee是无法使用 bsc, heco 等链的。

bond curve 是 bzz 增发和销毁机制，其目的是为了使bzz的价格趋于一定稳定性。

参考：

[Swarm Bond Curve分析](https://zhuanlan.zhihu.com/p/381605708?utm_source=wechat_timeline&utm_medium=social&s_r=0&wechatShare=1)