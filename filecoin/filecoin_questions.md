# Filecoin 问题汇总

## 环境变量问题

使用环境变量自定义 lotus 数据路径的时候，出现又恢复了默认路径 **~/.lotus**

解决：https://gitee.com/xingzjx/blockchain/blob/master/filecoin/filecoin_read_env_question.md

## Calibnet 测试网络区块同步问题

启动 lotus 节点后，出现报错提示 genesis 不一致，日志如下

```bash
ERROR: initializing node: starting node: could not build arguments for function 

#  ... 中间省略

asm_amd64.s:14): genesis in the repo is not the one expected by this version of Lotus!

```

解决：更新源码到 **https://network.filecoin.io/#calibration** 指定版本，然后构建编译出可执行文件，并且清理 **LOTUS_PATH** 目录和 **FIL_PROOFS_PARAMETER_CACHE** 目录，然后重新启动 lotus 节点。 


## Error: repo is already locked

执行命令停止 lotus 节点， 

```bash

lotus daemon stop

```

或者

```bash

lsof -i:1234

```

然后，找到进程id，使用 kill 命令停止进程。

## cant't find -lhwloc

确认已经安装hwloc依赖：

```bash

sudo apt install hwloc libhwloc-dev  -y && sudo apt upgrade -y

```

然后，建立软链接

```bash

ln -s /usr/lib/x86_64-linux-gnu/libhwloc.so.15 /usr/lib/x86_64-linux-gnu/libhwloc.so.5

```