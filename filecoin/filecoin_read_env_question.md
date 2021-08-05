


参考：


service 失效问题： https://blog.csdn.net/div_java/article/details/108449082

sudo 失效问题：https://blog.csdn.net/sean_8180/article/details/104172223


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

```bash

lotus daemon >> ~/log/lotus.log 2>&1 &

```