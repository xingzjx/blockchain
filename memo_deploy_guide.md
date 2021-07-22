# 去中心化存储 memo 部署指南

## 拉 docker 镜像

```bash

sudo docker pull memoio/mefs-provider:debian

```

## 创建私钥

```bash

sudo docker run -it -v /data/data6/memo:/root --entrypoint="/app/create" memoio/mefs-provider:debian

```

keystore地址：/data/data6/memo/.mefs/keystore

## 注册 provider

### 发送推特地址：

https://twitter.com/intent/tweet?text=@memolabsio%20I%20have%20joined%20the%20Memo%20Testnet.%20

### 生成链接如下：

https://twitter.com/quentin56365164/status/1417817055340892168?s=20


### 发送邮件到：supp@memolabs.io

Email content: 
account address (0xe00CfAF9303cB8875a1Dd5106D17053D0dD2B1bA)、role (provider) 
https://twitter.com/quentin56365164/status/1417817055340892168?s=20

## 运行docker 

```bash

sudo docker run --stop-timeout 30 \
    -p 4001:4001 \
    -v /data/data6/memo/data:/root \
    -e TRANSPORT=4001 \
    -e WALLET="0xe00CfAF9303cB8875a1Dd5106D17053D0dD2B1bA" \
    -e PASSWORD="12345678" \
    -e STORAGESIZE="600GB" \
    -e STORAGEPRICE="4000000000" \
    -e STORAGEDURATION="100" \
    -e POSENABLE="false" \
    --mount type=bind,source="/data/data6/memo/.mefs/keystore",destination=/app/keystore \
    --name memo_test memoio/mefs-provider:debian

2021/07/21 12:37:31 network is not running.
2021/07/21 12:37:32 network is not running.
Error: role is not provider

```
