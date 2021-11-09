# Sana 部署和运维教程

## 简介

Sana是Bzz的分叉币，目前还有Nbzz，Ebzz等。


Sana官方教程：https://docs.ethsana.org/docs/installation/install

Sana的源码工程：https://github.com/ethsana/sana

私钥导出工具：https://github.com/ethsana/exportSanaKey

仪表盘工具：https://github.com/ethsana/ant-dashboard

## 部署

### 下载安装包

```bash

wget https://github.com/ethsana/sana/releases/download/v0.1.1/ant-linux-amd64

```

### 编辑配置文件

```bash

wget https://github.com/ethsana/sana/blob/master/packaging/bee.yaml

```

编辑如下配置

```yaml

# ...省略
db-open-files-limit: 2000
api-addr: :10001 # ，默认1633
debug-api-addr: 0.0.0.0:10003 # 默认1635
p2p-addr: :10002 # 默认1634
full-node: true
nat-addr: "xx.xx.xx.xx:10002" # 主机的公网ip地址
password: "your-password"
password-file: /data/data0/sana/data/password
data-dir: /data/data0/sana/data
swap-endpoint: http://xx.xx.xx.xx:端口 # xDai网络地址
#...省略

```

注意：swap-endpoint的地址建议自己部署xDai网络节点，使用第三方节点在运行一段时间后会停止，如果部署大量节点还会拒绝连接。

### 启动节点

```bash

nohub ./ant-linux-amd64 start --config bee.yaml &

```

### 备份私钥

首先备份keystore目录keys，然后下载导出私有工具

```bash

wget https://github.com/ethsana/exportSanaKey/releases/download/v0.1.0/export-sana-key-linux-amd64

```

然后执行：

```bash

./export-sana-key-linux-amd64 ./keys your-password

```

### 打币

启动节点成功后，需要获取节点的以太坊地址，然后转帐0.1个xDai和50001个sana币。这个时候节点已经启动成功，开始挖矿。


## 仪表盘

通过仪表盘可以看到节点运营的情况，比如质押数，奖励情况等等。

```bash

npm install -g ant-dashboard

```

然后启动仪表盘：

```bash

ant-dashboard

```

在界面配置自己节点的公网IP和端口号。

## docker 

```bash

sudo docker run -d --cpus=0.5 --memory=512M --restart=always -p 1933:1633 -p 1934:1634 -p 1935:1635 -v ~/sana/bee.yaml:/home/sana/ant.yaml -v /data/data0/sana/data4:/home/sana/.sana  --name sana4 yourdocker/ant:v0.1.3 start --verbosity 5 --full-node --config /home/sana/ant.yaml --debug-api-enable

```

参考：

[SANA挖矿教程](https://www.yuque.com/shirendeyueliang/pv3y6w/rgihw5#OI55u)

[SANA一键部署](https://github.com/espoir1989/sana-install)

[SANA白皮书](https://docs.ethsana.org/sana_yellow_paper.pdf)