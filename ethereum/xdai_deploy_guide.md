# ubuntu 下部署 xDai 指南

## xDAI 客户端

xDai 客户端目前有两种实现：

- OpenEthereum (previously Parity): Rust Client
- Nethermind: .NET Client 

以下教程部署OpenEthereum为例。

## 下载客户端

下载地址：https://github.com/openethereum/openethereum/releases


选择openethereum-linux-v3.2.6.zip文件下载到本地，然后上传到ubuntu服务器。

```bash
 
scp -r ./openethereum/*    用户名@ip地址:/data/openethereum/

```

解压缩openethereum，进入目录，

```bash

ls
ethkey  ethstore  openethereum  openethereum-evm

```

## 启动客户端



```bash

sudo nohup ./openethereum --chain xdai --no-warp --jsonrpc-interface "all" --ws-interface="all" --base-path /data/xdai/data &

```

## 查看区块同步状态


```bash 

curl --data '{"method":"eth_syncing","params":[],"id":1,"jsonrpc":"2.0"}' -H "Content-Type: application/json" -X POST localhost:8545 | jq
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   219  100   160  100    59   156k  59000 --:--:-- --:--:-- --:--:--  213k
{
  "jsonrpc": "2.0",
  "result": {
    "currentBlock": "0x5f571",
    "highestBlock": "0xffa804",
    "startingBlock": "0x0",
    "warpChunksAmount": null,
    "warpChunksProcessed": null
  },
  "id": 1
}

```