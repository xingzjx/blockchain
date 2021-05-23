<h1>Ubuntu 部署以太坊节点笔记</h1>

# 阿里云服务器的准备

机器配置：CPU 2核，内存2G

镜像选择：公共镜像，Ubuntu，20.04 64位

注意：部署私有节点不需要过大磁盘，如果要同步主网和测试网节点数据，需要准备大概1T的磁盘空间。


# 安装ethereum
登录系统后，依次执行如下命令：

```zsh
apt-get update
apt-get install software-properties-common
add-apt-repository -y ppa:ethereum/ethereum
add-apt-repository -y ppa:ethereum/ethereum-dev
apt-get update
apt-get install ethereum
```

安装完成后，查看版本号

```zsh
geth version
```

# 启动以太坊私有链

## 创始区块信息准备

在工作目录(我的是/root/eth)下新建一个文件test_genesis.json，把下面内容复制进去

```
{
  "nonce":"0x0000000000000042",
  "mixhash":"0x0000000000000000000000000000000000000000000000000000000000000000",
  "difficulty": "0x1",
  "coinbase":"0x0000000000000000000000000000000000000000",
  "timestamp": "0x00",
  "parentHash":"0x0000000000000000000000000000000000000000000000000000000000000000",
  "extraData": "0x20180112",
  "gasLimit":"0x0000ffff",
  "coinbase": "0x660a107ee034Cb54bb05a22B5ecDdF050C8A2c90",
  "alloc": {},
  "config": {
    "chainId": 20180113,
    "homesteadBlock": 0,
    "eip155Block": 0,
    "eip158Block": 0
  }
}
```

## 创始区块初始化

执行下面命令，忽略WARN，如果打印Fatal,确认是否复制的时候有特殊字符插入

```zsh
geth init /root/eth/test_genesis.json
```

## 启动测试链

```zsh
geth --identity "test etherum" --rpc --rpccorsdomain "*" --datadir "/root/eth/data" --port "30303" --rpcapi "db,eth,net,web3" --networkid 20181013 console --dev
```

## Geth命令测试

用户命令测试

```zsh
eth.accounts

# 创建账户地址，参数为账户锁定密码，在转账前需要先解锁账户
# 我们把这个命令运行两次，创建两个地址，加上默认的，一共有了三个账户地址
personal.newAccount('111111')
personal.newAccount('111112')

# 为账户设置别名，方便命令输入
user1=eth.accounts[0]
user2=eth.accounts[1]
user3=eth.accounts[2]

# 查看地址user1余额,这个地址是测试链默认开通的一个地址，里面初始化有很多币
# 我们创建的另外两个地址余额未0
eth.getBalance(user1)

# 查看区块高度，现在为0
eth.blockNumber

# 转账测试，首先解锁账号user1
# 命令运行后要求输入解锁密码，直接回车，默认账号锁定密码为空，返回true成功
personal.unlockAccount(user1)

# 从user1向user2转账3个以太币
# 命令运行后，提交交易立马回出发挖矿
eth.sendTransaction({from:user1,to:user2,value:web3.toWei(3,"ether")})

# 查看区块高度，这时高度为1
eth.blockNumber
挖矿测试
geth启动后，自动启动挖矿，这时运行miner.start()，返回为null 无交易的时候不挖矿，当有交易时自动会触发挖矿流程

# 我们可以先停止挖矿
miner.stop()

# 提交交易，这时候只提交，查看账户余额，但是未确认
eth.sendTransaction({from:user1,to:user2,value:web3.toWei(3,"ether")})

# 启动挖矿，确认交易，再次查看账户余额
miner.start()

# 那么挖矿奖励去哪儿了？查看矿工地址
eth.coinbase

# 设置矿工地址
miner.setEtherbase(eth.coinbase)

```

# 启动主网或者测试网

## 启动主网

```zsh

geth -rpc --datadir  "/root/eth/data"

```

如果启动测试网络，可执行如下命令：

```zsh

geth -rpc --datadir  "/root/eth/data" --goerli

```

注意：如果节点查找不成功，请开启30303端口，验证端口监听情况：

```zsh

netstat -tlpn

```

docker方式启动

```

docker run -it --name eth_node -v "/home/xingzjx/eth/data":/root/.ethereum  -p 8545:8545  -p 30303:30303 ethereum/client-go --networkid 1234 --ws --rpc --rpcaddr 0.0.0.0 --rpccorsdomain '*' --rpcapi "db,eth,net,web3,personal" console

```

参考：

[使用Docker Images ethereum/client-go 搭建以太坊节点](https://blog.csdn.net/weixin_30697239/article/details/96857374)


## 进入控制台

```zsh

geth attach data/geth.ipc

```

## 查看同步状态

```zsh

> eth.syncing
{
  currentBlock: 153206,
  highestBlock: 12475934,
  knownStates: 347738,
  pulledStates: 326529,
  startingBlock: 0
}

```

参考：
 
[10分钟完成阿里云环境搭建以太坊私有链](https://zhuanlan.zhihu.com/p/32911405)

[以太坊主网节点搭建](https://www.jianshu.com/p/719a34fe484d)

[ETH搭建节点区块数据同步的三种模式：full、fast、light](https://www.cnblogs.com/bizzan/p/11341713.html)

[以太坊3个测试网络（ropsten、kovan、Rinkeby）的区别](https://blog.csdn.net/weixin_34194551/article/details/91902194)

[谷歌云部署Georli测试网络](https://medium.com/chainsafe-systems/deployment-automation-for-goerli-testnet-in-10-minutes-5212cef5542a)

[以太坊geth安装并同步主网及测试网区块](https://www.wanghaoyi.com/ethereum-geth-sync-blocks.html)