# Filecoin 环境变量配置问题

版本：Lotus1.11.0

配置环境变量

## 编辑 bashrc 

```bash

vim ~/.bashrc

```

## 添加环境变量

```bash

export LOTUS_MINER_PATH=/data/data1/filecoin/mainnet/miner
export LOTUS_PATH=/data/data1/filecoin/mainnet/lotus # When using a local node.
export BELLMAN_CPU_UTILIZATION=0.875 # Optimal value depends on your exact hardware.
export FIL_PROOFS_MAXIMIZE_CACHING=1
export FIL_PROOFS_USE_GPU_COLUMN_BUILDER=1 # When having GPU.
export FIL_PROOFS_USE_GPU_TREE_BUILDER=1   # When having GPU.
export FIL_PROOFS_PARAMETER_CACHE=/data/data1/filecoin/mainnet/parameter_cache # > 100GiB!
export FIL_PROOFS_PARENT_CACHE=/data/data1/filecoin/mainnet/parent_cache   # > 50GiB!
export TMPDIR=/data/data1/filecoin/mainnet/tmpdir                    # Used when sealing.

```

## 让环境变量生效

```bash

source ~/.bashrc

```

## 后台启动 lotus


```bash

lotus daemon >> ~/log/lotus.log 2>&1 &

```

## 注意事项

- 加上 sudo 或者 nohup 启动，环境变量会重置
- systemctl 方式启动，环境变量会重置，这个时候需要在启动脚本定义
- 直接在命令窗口定义变量，是临时变量，假如启动后换窗口执行 lotus 命令，会出现 **ERROR: websocket: bad handshake**问题

参考：

[linux启动的service服务无法获取当前系统的环境变量的值](https://blog.csdn.net/div_java/article/details/108449082)

[sudo命令无法读取环境变量的解决方法](https://blog.csdn.net/sean_8180/article/details/104172223)

[lotus ubuntu 18.04 安装(当前版本Devnet 7)](https://blog.csdn.net/u010953692/article/details/102684492)

[Linux下环境变量配置方法梳理（.bash_profile和.bashrc的区别）](https://www.cnblogs.com/kevingrace/p/8072860.html)



