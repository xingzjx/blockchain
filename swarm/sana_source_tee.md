# Sana 源码解读：挖矿


## Sana简介

Sana是Swarm Bzz的分叉项目，其源码和Bee客户端基本差不多，只不过加入了挖矿的功能。其中，Tust协议是可信协议的定义，也是Tee实现的核心协议。

协议规格如下：

**pkg/mine/trust/pb/trust.proto**

```proto

syntax = "proto3";

package trust;

option go_package = "pb";


message TrustSign {
  int32 id = 1;
  int32 op = 2;
  bytes Peer = 3;
  bytes data = 4;
  int64 expire = 5;
  bool result = 6;
}


message Trust {
  int64 expire = 1;
  bytes stream = 2;
}

```

## 挖矿启动流程

在前面的文章中，有讲到过bee的启动流程，sana的启动流程基本一直。其中，会调用到node模块的node.go的 *NewAnt* 方法，核心逻辑如下：

- mine.NewService： 创建挖矿服务
- trust.New： 初始化trust协议
- 

**pkg/node/node.go**

```go
func NewAnt(addr string, publicKey *ecdsa.PublicKey, signer crypto.Signer, networkID uint64, logger logging.Logger, libp2pPrivateKey, pssPrivateKey *ecdsa.PrivateKey, o *Options) (b *Ant, err error) {
	tracer, tracerCloser, err := tracing.NewTracer(&tracing.Options{
		Enabled:     o.TracingEnabled,
		Endpoint:    o.TracingEndpoint,
		ServiceName: o.TracingServiceName,
	})
	if err != nil {
		return nil, fmt.Errorf("tracer: %w", err)
	}

	p2pCtx, p2pCancel := context.WithCancel(context.Background())

    // ...创建挖矿服务

    if !o.Standalone {
		syncSvc = syncer.New(logger, swapBackend, o.BlockTime, &pidKiller{node: b})

		chainCfg, found := config.GetChainConfig(chainID)
		postageContractAddress, startBlock := chainCfg.PostageStamp, chainCfg.StartBlock
		if o.PostageContractAddress != "" {
			if !common.IsHexAddress(o.PostageContractAddress) {
				return nil, errors.New("malformed postage stamp address")
			}
			postageContractAddress = common.HexToAddress(o.PostageContractAddress)
		} else if !found {
			return nil, errors.New("no known postage stamp addresses for this network")
		}

		if cs := batchStore.GetChainState(); cs.Block == 0 {
			cs.Block = startBlock
			err := batchStore.PutChainState(cs)
			if err != nil {
				return nil, err
			}
		}

		batchSvc, err = batchservice.New(stateStore, batchStore, logger, postageContractAddress, overlayEthAddress.Bytes(), startBlock, post, sha3.New256)
		if err != nil {
			return nil, err
		}
		syncSvc.AddSync(batchSvc.Sync())

		erc20Address, err := postagecontract.LookupERC20Address(p2pCtx, transactionService, postageContractAddress)
		if err != nil {
			return nil, err
		}

		postageContractService = postagecontract.New(
			overlayEthAddress,
			postageContractAddress,
			erc20Address,
			transactionService,
			post,
		)

		if o.MineEnabled {
			mineContractAddress := chainCfg.MinerAddress
			if o.MineContractAddress != "" {
				if !common.IsHexAddress(o.MineContractAddress) {
					return nil, errors.New("malformed mine address")
				}
				mineContractAddress = common.HexToAddress(o.MineContractAddress)
			}

			nodeStore, err := nodestore.New(stateStore, startBlock, logger)
			if err != nil {
				return nil, fmt.Errorf("nodestore: %w", err)
			}

			mineService := minecontract.New(swapBackend, transactionService, mineContractAddress)

			nodeSvc, err := nodeservice.New(stateStore, nodeStore, logger, swapBackend, startBlock, mineContractAddress)
			if err != nil {
				return nil, err
			}

			syncSvc.AddSync(nodeSvc.Sync())

			if o.UniswapEnable {
				oracleSvr, err = oracle.New(o.UniswapEndpoint, chainCfg.UniV2PairAddress, o.UniswapValidTime, logger)
				if err != nil {
					return nil, err
				}

			}

			mineSvr = mine.NewService(swarmAddress, mineService, nodeSvc, signer, oracleSvr, logger, warmupTime, mine.Options{
				Store:              stateStore,
				Backend:            swapBackend,
				TransactionService: transactionService,
				OverlayEthAddress:  overlayEthAddress,
				DeployGasPrice:     o.DeployGasPrice,
			})

			b.mineCloser = mineSvr
		}
	}


    // ... 创建trust协议
    if o.MineEnabled {
		trust := trust.New(p2ps, logger, swarmAddress)
		if err = p2ps.AddProtocol(trust.Protocol()); err != nil {
			return nil, fmt.Errorf("rollcall service: %w", err)
		}
		mineSvr.SetTrust(trust)
		trust.SetTopology(kad)
		trust.SetMineObserver(mineSvr)
	}


    // ...启动挖矿

    if o.MineEnabled && mineSvr != nil {
		mineSvr.Start()
	}
}

```

启动mine服务的时候，会初始化tee的DeviceID，

**pkg/mine/service.go**

```go
func NewService(
	base swarm.Address,
	contract MineContract,
	nodes NodeService,
	signer crypto.Signer,
	oracle Oracle,
	logger logging.Logger,
	warmupTime time.Duration,
	opt Options,
) Service {

	device, err := tee.DeviceID()
	if err == nil {
		platform := "Unknown"
		switch device.Platform {
		case tee.AMD:
			platform = "AMD Platform"

		case tee.Intel:
			platform = "Intel Platform"
		}

		logger.Infof("using the %s Tee device", platform)
	}

	return &service{
		base:     base,
		signer:   signer,
		contract: contract,
		nodes:    nodes,
		oracle:   oracle,
		logger:   logger,
		device:   device,
		opt:      &opt,
		rcnc:     make(chan rcn, 1024),
		quit:     make(chan struct{}),
	}
}
```

Trust协议后面在分析，接下来，看挖矿的start方法：启动协程，执行manage方法

**pkg/mine/service.go**

```go

func (s *service) Start() {
	s.wg.Add(1)
	go s.manange()
}

```

manage方法如下：在manage中，会启动一个for循环，然后使用select机制，接收通道信息。

**pkg/mine/service.go**

```go
func (s *service) manange() {
	defer s.wg.Done()

	c, unsubscribe := s.nodes.SubscribeRollCall()
	defer unsubscribe()

	s.logger.Info("mine worker starting.")

	timer := time.NewTimer(0)
	defer timer.Stop()

	var expireChan <-chan time.Time

	for {
		select {
		case height := <-c:
			if s.nodes.TrustOf(s.base) {
				s.logger.Infof("mine: start to detect node online")
				expire := time.Now().Add(time.Minute).Unix()

				byts := make([]byte, 8)
				binary.BigEndian.PutUint64(byts, height)
				err := s.trust.PushRollCall(context.Background(), expire, append(s.base.Bytes(), byts...))
				if err != nil {
					s.logger.Infof("push to detect online message failed: %s", err)
				}

				// check expire nodes
				expireChan = time.After(time.Second)
			}
			err := s.checkSelfTrustRollCallSign(height)
			if err != nil {
				s.logger.Debugf("self check rollcall sign failed: at %s", err)
			}

		case <-expireChan:
			err := s.checkExpireMiners()
			if err != nil {
				s.logger.Infof("inaction expire miner failed at %s", err)
			}

		case rc := <-s.rcnc:
			if rc.Expire <= time.Now().Unix() {
				break
			}

			err := s.signRollCallToTrust(rc.Expire, rc.Height, rc.Address)
			if err != nil {
				s.logger.Errorf("sign rollcall to trust %s fail: %v", rc.Address.String(), err.Error())
				s.rcnc <- rc
			}

		case <-timer.C:
			ok, err := s.checkWorkingWorker()
			if err != nil {
				s.logger.Infof("check mine working fail: %v", err)
			}

			if ok {
				timer.Reset(time.Minute * 30)
				s.logger.Infof("the overlay address %v mining", s.base.String())
			} else if errors.Is(err, errNodeIsCashout) || errors.Is(err, errNodeIsFreeze) {
				timer.Reset(time.Hour)
			} else {
				timer.Reset(time.Second * 30)
			}

		case <-s.quit:
			return
		}
	}
}
```

## 状态检查

当矿工被激活后，执行命令，会返回矿工的状态和激励

```bash

curl localhost:1635/mine/status

```

其接口实现在debugapi目录下：

**pkg/debugapi/mine.go**

```go

func (s *Service) mineStatusHandler(w http.ResponseWriter, r *http.Request) {
	if !s.minerEnabled {
		jsonhttp.InternalServerError(w, errMineDisable)
		return
	}

	work, freeze, reward, pending, expire, deposit, err := s.mine.Status(context.Background())
	if err != nil {
		s.logger.Infof("get mine status err: %s", err)
		jsonhttp.InternalServerError(w, fmt.Sprint("Cannot get mine status at:", err.Error()))
		return
	}

	jsonhttp.OK(w, mineStatusResponse{
		Work:    work,
		Freeze:  freeze,
		Reward:  bigint.Wrap(reward),
		Pending: bigint.Wrap(pending),
		Expire:  bigint.Wrap(expire),
		Deposit: bigint.Wrap(deposit),
	})
}

```

Status的实现在mine目录的service.go，Status方法会调用挖矿合约中实现

**pkg/mine/service.go**

```go
func (s *service) Status(ctx context.Context) (work, freeze bool, withdraw *big.Int, reward *big.Int, expire *big.Int, deposit *big.Int, err error) {
	node := common.BytesToHash(s.base.Bytes())

	work, err = s.contract.IsWorking(ctx, node)
	if err != nil {
		return
	}
	freeze, err = s.contract.FreezeOf(ctx, node)
	if err != nil {
		return
	}
	withdraw, err = s.contract.MinersWithdraw(ctx, node)
	if err != nil {
		return
	}
	reward, err = s.contract.Reward(ctx, node)
	if err != nil {
		return
	}
	expire, err = s.contract.ExpireOf(ctx, node)
	if err != nil {
		return
	}
	deposit, err = s.contract.DepositOf(ctx, node)
	return
}
```

这里以分析IsWorker方法为例


**pkg/mine/minecontract/contract.go**

```go

func (svc *service) IsWorking(ctx context.Context, node common.Hash) (bool, error) {
	callData, err := mineABI.Pack(`miners`, node)
	if err != nil {
		return false, err
	}

	output, err := svc.transactionService.Call(ctx, &transaction.TxRequest{
		To:   &svc.address,
		Data: callData,
	})
	if err != nil {
		return false, err
	}

	results, err := mineABI.Unpack("miners", output)
	if err != nil {
		return false, err
	}
	if len(results) != 6 {
		return false, errDecodeABI
	}
	return results[0].(bool), nil
}

```

其中，mine合约地址的定义如下：

**pkg/config/chain.go**

```go
var (
	// chain ID
	goerliChainID = int64(5)
	xdaiChainID   = int64(100)
	devChainID    = int64(31337)
	// start block
	goerliStartBlock = uint64(5167986)
	xdaiStartBlock   = uint64(17525957)
	devStartBlock    = uint64(0)
	// factory address
	goerliContractAddress = common.HexToAddress("0x2469391F81F38313CfC6BfBb6132EDf27B0d55A0")
	xdaiContractAddress   = common.HexToAddress("0x61E7cdF724446aAfCBF56312b501a0072Ae90Eee")
	devContractAddress    = common.HexToAddress("0xe7f1725E7734CE288F8367e1Bb143E90bb3F0512")
	goerliFactoryAddress  = common.HexToAddress("0x6737bA0bFA19EEDf1FC7FA73947bc5885F4b511c")
	xdaiFactoryAddress    = common.HexToAddress("0xf1829378C26cE9f8D3b1b8610334258515d0A7B5")
	// goerliLegacyFactoryAddress = common.HexToAddress("0xf0277caffea72734853b834afc9892461ea18474")
	devFactoryAddress = common.HexToAddress("0xCf7Ed3AccA5a467e9e704C703E8D87F634fB0Fc9")
	// postage stamp
	goerliPostageStampContractAddress = common.HexToAddress("0x2D1Fb33057a5022a870707aEe74eA991fDed764e")
	xdaiPostageStampContractAddress   = common.HexToAddress("0xeb2baF84d972a091232654CaCc719E6D833E2118")
	devPostageStampComtractAddress    = common.HexToAddress("0x9fE46736679d2D9a65F0992F2272dE9f3c7fa6e0")

	// miner address
	goerliMinerAddress = common.HexToAddress("0xfeb4c0E59329A183c75bbE579c3aC4915241Af0c")
	devMinerAddress    = common.HexToAddress("0xDc64a140Aa3E981100a9becA4E685f962f0cF6C9")
	xdaiMinerAddress   = common.HexToAddress("0x36032EA08fbdE8143e53afa9C752AcD87e8FAd7C")

	// uniswap v2 pool
	// devUniV2Pair =
	mainUniV2Pair = common.HexToAddress("0xa67741e5929d970c133cc8943ce3b8d9115bf392")
)
```