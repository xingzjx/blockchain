# filecoin 源码解读：编译过程分析

环境：lotus，版本：1.11.0

##　准备工作

### 安装go和rust

go安装：https://golang.google.cn/doc/install

rust安装：https://www.rust-lang.org/tools/install

### 下载源码

```bash

git clone -b v1.10.0 https://github.com/filecoin-project/lotus.git

```

### 子模块依赖

下载子模块，如果下载不下来，直接复制工程源码到 extern 目录

```bash

git submodule update --init --recursive

```

## 编译

### 编译Mainnet

Mainnet：Filecoin网络的生产环境。FIL在这个网络中具有现实价值。

```bash

make clean all

```

如果只编译部分模块，比如lotus或者lotus-miner，执行

```bash

make clean lotus

```

### 编译Calinet

Calinet： 最小扇区大小为32 GiB的测试网络。在这个网络中，FIL没有实际价值。

```bash

make clean calibnet

```

### 编译Nerpanet

Nerpanet： 最小扇区大小为512 MiB的测试网络。在这个网络中，FIL没有实际价值。

```bash

make clean nerpanet

```

## 编译流程

### 清理工作

在执行 make clean nerpanet，首先先执行clean命令，进行清理相关工作。

命令如下：

Makefile

```Makefile

clean:
	rm -rf $(CLEAN) $(BINS)
	-$(MAKE) -C $(FFI_PATH) clean

```

该命令，包括清理工程目录的可执行文件。

```bash

rm -rf  build/.filecoin-install build/.update-modules  lotus lotus-miner lotus-worker lotus-shed lotus-gateway lotus-seed lotus-pond lotus-townhall lotus-fountain lotus-chainwat

```

清理外部依赖编译文件

```bash

make -C extern/filecoin-ffi/ clean

rm -rf filcrypto.h filcrypto.pc libfilcrypto.a .install-filcrypt

rm -f ./runner

```

### 更新子模块

```bash

git submodule update --init --recursive

```

### 编译filecoin-ffi模块

```bash

make -C extern/filecoin-ffi/ .install-filcrypto

./install-filcrypto

```
### 编译cmd目录下的子模块

```bash

rm -f lotus

go build  -ldflags="-X=github.com/filecoin-project/lotus/build.CurrentCommit=+git.715176698.dirty" -tags=nerpanet -o lotus ./cmd/lotus

rm -f lotus-miner

go build  -ldflags="-X=github.com/filecoin-project/lotus/build.CurrentCommit=+git.715176698.dirty" -tags=nerpanet -o lotus-miner ./cmd/lotus-storage-miner

rm -f lotus-worker

go build  -ldflags="-X=github.com/filecoin-project/lotus/build.CurrentCommit=+git.715176698.dirty" -tags=nerpanet -o lotus-worker ./cmd/lotus-seal-worker

Caution: you have an existing lotus binary in your PATH. This may cause problems if you don\'t run 'sudo make install'
rm -f lotus-seed

go build  -ldflags="-X=github.com/filecoin-project/lotus/build.CurrentCommit=+git.715176698.dirty" -tags=nerpanet -o lotus-seed ./cmd/lotus-seed

rm -f lotus-shed

go build  -ldflags="-X=github.com/filecoin-project/lotus/build.CurrentCommit=+git.715176698.dirty" -tags=nerpanet -o lotus-shed ./cmd/lotus-shed

rm -f lotus-wallet

go build -o lotus-wallet ./cmd/lotus-wallet

rm -f lotus-gateway

go build  -ldflags="-X=github.com/filecoin-project/lotus/build.CurrentCommit=+git.715176698.dirty" -tags=nerpanet -o lotus-gateway ./cmd/lotus-gateway

```

从编译命令中可以看到，编译的二进制可执行文件，会在当前工程目录生成，还有一个重要参数 **-tags=nerpanet**，根据这个参数，加载不同的配置文件，从而编译出不同网络环境的可执行文件。

build/params_calibnet.go

```go

// +build nerpanet

package build

import (
	"github.com/filecoin-project/go-state-types/abi"
	"github.com/filecoin-project/lotus/chain/actors/policy"
	"github.com/ipfs/go-cid"

	builtin2 "github.com/filecoin-project/specs-actors/v2/actors/builtin"
)

var DrandSchedule = map[abi.ChainEpoch]DrandEnum{
	0: DrandMainnet,
}

const BootstrappersFile = "nerpanet.pi"
const GenesisFile = "nerpanet.car"

const UpgradeBreezeHeight = -1
const BreezeGasTampingDuration = 0

const UpgradeSmokeHeight = -1

const UpgradeIgnitionHeight = -2
const UpgradeRefuelHeight = -3

const UpgradeLiftoffHeight = -5

const UpgradeAssemblyHeight = 30 // critical: the network can bootstrap from v1 only
const UpgradeTapeHeight = 60

const UpgradeKumquatHeight = 90

const UpgradeCalicoHeight = 100
const UpgradePersianHeight = UpgradeCalicoHeight + (builtin2.EpochsInHour * 1)

const UpgradeClausHeight = 250

const UpgradeOrangeHeight = 300

const UpgradeTrustHeight = 600
const UpgradeNorwegianHeight = 201000
const UpgradeTurboHeight = 203000
const UpgradeHyperdriveHeight = 999999999

func init() {
	// Minimum block production power is set to 4 TiB
	// Rationale is to discourage small-scale miners from trying to take over the network
	// One needs to invest in ~2.3x the compute to break consensus, making it not worth it
	//
	// DOWNSIDE: the fake-seals need to be kept alive/protected, otherwise network will seize
	//
	policy.SetConsensusMinerMinPower(abi.NewStoragePower(4 << 40))

	policy.SetSupportedProofTypes(
		abi.RegisteredSealProof_StackedDrg512MiBV1,
		abi.RegisteredSealProof_StackedDrg32GiBV1,
		abi.RegisteredSealProof_StackedDrg64GiBV1,
	)

	// Lower the most time-consuming parts of PoRep
	policy.SetPreCommitChallengeDelay(10)

	// TODO - make this a variable
	//miner.WPoStChallengeLookback = abi.ChainEpoch(2)

	Devnet = false
}

const BlockDelaySecs = uint64(builtin2.EpochDurationSeconds)

const PropagationDelaySecs = uint64(6)

// BootstrapPeerThreshold is the minimum number peers we need to track for a sync worker to start
const BootstrapPeerThreshold = 4

var WhitelistedBlock = cid.Undef

```

- BootstrappersFile 指定p2p网络引导节点的地址

build/bootstrap/nerpanet.pi

```pi

/dns4/bootstrap-2.nerpa.interplanetary.dev/tcp/1347/p2p/12D3KooWQcL6ReWmR6ASWx4iT7EiAmxKDQpvgq1MKNTQZp5NPnWW
/dns4/bootstrap-0.nerpa.interplanetary.dev/tcp/1347/p2p/12D3KooWGyJCwCm7EfupM15CFPXM4c7zRVHwwwjcuy9umaGeztMX
/dns4/bootstrap-3.nerpa.interplanetary.dev/tcp/1347/p2p/12D3KooWNK9RmfksKXSCQj7ZwAM7L6roqbN4kwJteihq7yPvSgPs
/dns4/bootstrap-1.nerpa.interplanetary.dev/tcp/1347/p2p/12D3KooWCWSaH6iUyXYspYxELjDfzToBsyVGVz3QvC7ysXv7wESo

```

- GenesisFile 创世，nerpanet.car


- init方法

在init方法中定义了扇区的最小类型是512MiB。

```go

policy.SetSupportedProofTypes(
		abi.RegisteredSealProof_StackedDrg512MiBV1,
		abi.RegisteredSealProof_StackedDrg32GiBV1,
		abi.RegisteredSealProof_StackedDrg64GiBV1,
	)

```

[Go语言 通过go bulid -tags 实现编译控制](https://zhuanlan.zhihu.com/p/269746831)