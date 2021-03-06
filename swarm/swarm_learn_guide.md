
<h1>Swarm 学习指南</h>

# bee的部署

## 安装Bee Clef

Bee Clef 是一个私钥对管理工具，提高安全性用到的工具

```
wget https://github.com/ethersphere/bee-clef/releases/download/v0.4.12/bee-clef_0.4.12_amd64.deb
sudo dpkg -i bee-clef_0.4.12_amd64.deb

```

查看 Bee Clef 状态

```

systemctl status bee-clef

```

## 导出私钥

```

[root@mblockServerA ~]# bee-clef-keys
Key exported to /root/bee-clef-key-1622016335.json
Pass exported to /root/bee-clef-password-1622016335.txt

```

把json和txt文件导出，然后使用小狐狸钱包导入。

## 安装Bee客户端

```

wget https://github.com/ethersphere/bee/releases/download/v0.6.1/bee_0.6.0_amd64.deb
sudo dpkg -i bee_0.6.0_amd64.deb

```

## 配置Bee

## 验证刷票

使用cashout.sh脚本，可以查询票数

```
./cashout.sh
```

[cashout文档](https://docs.ethswarm.org/docs/working-with-bee/cashing-out)

## 提现到钱包

使用命令

```
./cashout.sh cashout-all 5
```

遇到问题：goerli网络转账慢，建议提高gas到2500左右



参考：

[最火项目Swarm( bzz）为何受到如此追捧](https://www.163.com/dy/article/GA9PK2O405149MB5.html)

[Ubuntu安装部署Swarm Bee V0.5.3教程](https://www.yuque.com/daxiansheng-ohldj/ilm2lv/nccrxg)

[Swarm刷票教程](https://vlambda.com/wz_7iIczyfM1kw.html)

[DebugAPI工具](https://docs.ethswarm.org/debug-api/)

[keystore导出私钥](https://www.yundongfang.com/Yun41920.html)


[clef命令文档](https://geth.ethereum.org/docs/clef/tutorial)

[ETH官方项目分布式储存swarm搭建bee节点linux教程](https://www.jinse.com/news/blockchain/1097799.html)


# docker-compose部署

## 下载启动文件

```
wget -q https://raw.githubusercontent.com/ethersphere/bee/v0.6.0/packaging/docker/docker-compose.yml

wget -q https://raw.githubusercontent.com/ethersphere/bee/v0.6.0/packaging/docker/env -O .env

```

## 修改docker-compose

打开docker-compose.yaml文件，修改bee和bee-clef的版本号

bee: 0.6.0

bee-clef: 0.4.12

注意bee和bee-clef的版本号，否则会出现版本兼容，签名找不到的问题。

## 修改env环境变量

参考上文bee的部署

## 启动

```

docker-compose up -d

```

## 查看日志

```

docker-compose logs -f bee-1

```

## 验证服务

```

curl localhost:1633

```

## 查看节点状态

```
docker-compose ps -a
```

[Swarm-BZZ踩坑日记之 如何在Linux下使用docker开30个bzz节点](https://blog.csdn.net/Z1404686551/article/details/117486604)

# k8s集群部署

参考：

[swarm集群helm版](https://github.com/ethersphere/helm)

[filecoin集群helm版](https://github.com/glifio/filecoin-chart)

[多集群方案operator版](https://github.com/kotalco/kotal)

# 开发工具

## swarm-cli

## swarm-dashboard
 
 # dapp

 # 白皮书解读

[swarm技术白皮书](https://chinapeace.github.io/pdf/latest.bookofswarm.eth.ZH_CN.pdf)

[swarm官方技术文档bookofswarm](https://gateway.ethswarm.org/bzz/latest.bookofswarm.eth/)

 # 源码导读