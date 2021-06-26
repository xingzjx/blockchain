# ubuntu docker-compose 部署100个bee节点


## 安装ubuntu20.0.4

裸机安装ubuntu20.0.4，参考：

[安装Ubuntu 20.04 LTS服务器的图文教程](https://blog.csdn.net/weixin_45763600/article/details/114833521)

安装系统后注意挂载磁盘到根目录，比如data0-data7， 参考：

[ubuntu 分区_Ubuntu系统挂载新硬盘的方法](https://blog.csdn.net/weixin_39582656/article/details/111116619)


## 配置docker-compose模板

创建bee1目录，在bee1目录下生成.env和docker-compose.yml文件。

.env文件：

```conf
# Copy this file to .env, then update it with your own settings

### CLEF

## chain id to use for signing (1=mainnet, 3=ropsten, 4=rinkeby, 5=goerli) (default: 12345)
CLEF_CHAINID=5

### BEE

## HTTP API listen address (default :1633)
# BEE_API_ADDR=:1633
## chain block time (default 15)
# BEE_BLOCK_TIME=15
## initial nodes to connect to (default [/dnsaddr/bootnode.ethswarm.org])
# BEE_BOOTNODE=[/dnsaddr/bootnode.ethswarm.org]
## cause the node to always accept incoming connections
# BEE_BOOTNODE_MODE=false
## enable clef signer
BEE_CLEF_SIGNER_ENABLE=true
## clef signer endpoint
# BEE_CLEF_SIGNER_ENDPOINT=
## config file (default is /home/<user>/.bee.yaml)
# BEE_CONFIG=/home/bee/.bee.yaml
## origins with CORS headers enabled
# BEE_CORS_ALLOWED_ORIGINS=[]
## data directory (default /home/<user>/.bee)
# BEE_DATA_DIR=/home/bee/.bee
## cache capacity in chunks, multiply by 4096 to get approximate capacity in bytes
# BEE_CACHE_CAPACITY=1000000
## number of open files allowed by database
# BEE_DB_OPEN_FILES_LIMIT=200
## size of block cache of the database in bytes
# BEE_DB_BLOCK_CACHE_CAPACITY=33554432
## size of the database write buffer in bytes
# BEE_DB_WRITE_BUFFER_SIZE=33554432
## disables db compactions triggered by seeks
# BEE_DB_DISABLE_SEEKS_COMPACTION=false
## debug HTTP API listen address (default :1635)
# BEE_DEBUG_API_ADDR=:1635
## enable debug HTTP API
BEE_DEBUG_API_ENABLE=true
## disable a set of sensitive features in the api
# BEE_GATEWAY_MODE=false
## enable global pinning
# BEE_GLOBAL_PINNING_ENABLE=false
## cause the node to start in full mode
BEE_FULL_NODE=true
## NAT exposed address
BEE_NAT_ADDR=xx.xx.xx.xx:1734
## ID of the Swarm network (default 1)
# BEE_NETWORK_ID=1
## P2P listen address (default :1634)
# BEE_P2P_ADDR=:1634
## enable P2P QUIC protocol
BEE_P2P_QUIC_ENABLE=true
## enable P2P WebSocket transport
# BEE_P2P_WS_ENABLE=false
## password for decrypting keys
BEE_PASSWORD=123456
## path to a file that contains password for decrypting keys
# BEE_PASSWORD_FILE=
## amount in BZZ below the peers payment threshold when we initiate settlement (default 1000000000000)
# BEE_PAYMENT_EARLY=1000000000000
## threshold in BZZ where you expect to get paid from your peers (default 10000000000000)
# BEE_PAYMENT_THRESHOLD=10000000000000
## excess debt above payment threshold in BZZ where you disconnect from your peer (default 10000000000000)
# BEE_PAYMENT_TOLERANCE=10000000000000
## postage stamp contract address
# BEE_POSTAGE_STAMP_ADDRESS=
## ENS compatible API endpoint for a TLD and with contract address, can be repeated, format [tld:][contract-addr@]url
# BEE_RESOLVER_OPTIONS=[]
## whether we want the node to start with no listen addresses for p2p
# BEE_STANDALONE=false
## enable swap (default true)
# BEE_SWAP_ENABLE=true
## swap ethereum blockchain endpoint (default ws://localhost:8546)
BEE_SWAP_ENDPOINT=wss://goerli.infura.io/ws/v3/f820d78ccf88447d99249ba44b31bfe2
## swap factory address
# BEE_SWAP_FACTORY_ADDRESS=
## legacy swap factory addresses
# BEE_SWAP_LEGACY_FACTORY_ADDRESSES=
## initial deposit if deploying a new chequebook (default 10000000000000000)
# BEE_SWAP_INITIAL_DEPOSIT=10000000000000000
## gas price in wei to use for deployment and funding (default "")
# BEE_SWAP_DEPLOYMENT_GAS_PRICE=
## enable tracing
# BEE_TRACING_ENABLE=false
## endpoint to send tracing data (default 127.0.0.1:6831)
# BEE_TRACING_ENDPOINT=127.0.0.1:6831
## service name identifier for tracing (default bee)
# BEE_TRACING_SERVICE_NAME=bee
## proof-of-identity transaction hash
# BEE_TRANSACTION=
## log verbosity level 0=silent, 1=error, 2=warn, 3=info, 4=debug, 5=trace (default info)
# BEE_VERBOSITY=info
## send a welcome message string during handshakes
# BEE_WELCOME_MESSAGE=
```

docker-compose.yml文件：

```yml
version: "3"

services:
  clef-1:
    image: ethersphere/clef:0.4.12
    restart: always
    environment:
      - CLEF_CHAINID
    volumes:
      - /data/data0/clef-1:/app/data
    command: full

  bee-1:
    image: ethersphere/bee:beta
    restart: always
    environment:
      - BEE_API_ADDR
      - BEE_BOOTNODE
      - BEE_BOOTNODE_MODE
      - BEE_CLEF_SIGNER_ENABLE
      - BEE_CLEF_SIGNER_ENDPOINT=http://clef-1:8550
      - BEE_CONFIG
      - BEE_CORS_ALLOWED_ORIGINS
      - BEE_DATA_DIR
      - BEE_CACHE_CAPACITY
      - BEE_DB_OPEN_FILES_LIMIT
      - BEE_DB_BLOCK_CACHE_CAPACITY
      - BEE_DB_WRITE_BUFFER_SIZE
      - BEE_DB_DISABLE_SEEKS_COMPACTION
      - BEE_DEBUG_API_ADDR
      - BEE_DEBUG_API_ENABLE
      - BEE_GATEWAY_MODE
      - BEE_GLOBAL_PINNING_ENABLE
      - BEE_NAT_ADDR
      - BEE_NETWORK_ID
      - BEE_P2P_ADDR
      - BEE_P2P_QUIC_ENABLE
      - BEE_P2P_WS_ENABLE
      - BEE_PASSWORD
      - BEE_PASSWORD_FILE
      - BEE_PAYMENT_EARLY
      - BEE_PAYMENT_THRESHOLD
      - BEE_PAYMENT_TOLERANCE
      - BEE_RESOLVER_OPTIONS
      - BEE_STANDALONE
      - BEE_SWAP_ENABLE
      - BEE_SWAP_ENDPOINT
      - BEE_SWAP_FACTORY_ADDRESS
      - BEE_SWAP_INITIAL_DEPOSIT
      - BEE_TRACING_ENABLE
      - BEE_TRACING_ENDPOINT
      - BEE_TRACING_SERVICE_NAME
      - BEE_VERBOSITY
      - BEE_WELCOME_MESSAGE
      - BEE_FULL_NODE 
    ports:
      - "${API_ADDR:-10010}${BEE_API_ADDR:-:1633}"
      - "${P2P_ADDR:-10011}${BEE_P2P_ADDR:-:1634}"
      - "${DEBUG_API_ADDR:-127.0.0.1:10012}${BEE_DEBUG_API_ADDR:-:1635}"
    volumes:
      - /data/data0/bee-1:/home/bee/.bee
    command: start
    depends_on:
      - clef-1
```

配置参数说明：

BEE_SWAP_ENDPOINT：这个是以太坊节点的交换地址，部署100个节点需要自己部署以太坊goerli网络全节点，参考：

[Ubuntu 部署以太坊节点笔记](https://gitee.com/xingzjx/blockchain/blob/master/ubuntu%20deploy%20ethereum%20node.md)

BEE_NAT_ADDR：配置成主机的ip，端口是p2p的端口号。

BEE_FULL_NODE：这个参数在0.6.0以上需要设置为true,默认是Light模式。

注意：使用docker-compose，启动的容器默认是用的root权限，但是docker中的root只是相当于普通用户,所以需要给挂载的目录或者文件开启权限，代码如下：

```bash

#开启主机目录权限
chmod a+rwx /data/data0/ 

#开启docker挂载权限
chmod a+rw /var/run/docker.sock 

```

## 批量复制bee1目录

编写shell或者python脚本，批量复制bee1目录，并且替换参数。

API_ADDR：10000 + n * 10

P2P_ADDR：10001 + n * 10

DEBUG_API_ADDR：10002 + n * 10

bee-1用bee-n替换，clef-1用clef-n替换。

规则：n表示第n个节点，每个节点端口加10。

## 批量启动节点

```bash

cd bee-1
sudo docker-compose up -d

```

## 获取打水地址

```bash
#!/bin/bash
for   fileName in `find  /data/data0/  -name "UTC--2021*"  |  sort`
do
   #echo $fileName
   name=${fileName##*/}
   address="0x${name:0-40},1"
   echo $address
   echo ${address} >> address.txt
done
```

使用cointool工具批量打水，一个gETH和一个gBZZ。


## 节点健康检查

```bash 
#!/bin/bash
a=0
until [ ! $a -lt 100 ]
do
   # echo $a
   a=`expr $a + 1`
   echo $a
   # port=10000+10*$a
   port_tail=`expr $a \* 10`
   curl localhost:`expr $port_tail + 10000`
```
## 获取连接数

```bash
#!/bin/bash
a=0
currtime=`date "+%G-%m-%d%H:%M:%S" `
echo $currtime
until [ ! $a -lt 100 ]
do
   # echo $a
   a=`expr $a + 1`
   title="bee-$a : peers count"
   echo $title  >> tmp.txt
   # port=10000+10*$a
   port_tail=`expr $a \* 10`
   curl localhost:`expr $port_tail + 10002`/peers | jq '.peers | length' >> tmp.txt
done
mv  tmp.txt peers${currtime}.txt
```










