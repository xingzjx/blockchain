# Bee 源码解读：P2P 网络


## 启动 Bee 节点

node.go

```go

func NewBee(addr string, swarmAddress swarm.Address, publicKey ecdsa.PublicKey, signer crypto.Signer, networkID uint64, logger logging.Logger, libp2pPrivateKey, pssPrivateKey *ecdsa.PrivateKey, o Options) (b *Bee, err error) {

// ... 前面省略

// 先创建 p2p 的服务 p2ps
p2ps, err := libp2p.New(p2pCtx, signer, networkID, swarmAddress, addr, addressbook, stateStore, lightNodes, senderMatcher, logger, tracer, libp2p.Options{
		PrivateKey:     libp2pPrivateKey,
		NATAddr:        o.NATAddr,
		EnableWS:       o.EnableWS,
		EnableQUIC:     o.EnableQUIC,
		Standalone:     o.Standalone,
		WelcomeMessage: o.WelcomeMessage,
		FullNode:       o.FullNodeMode,
		Transaction:    txHash,
	})
	if err != nil {
		return nil, fmt.Errorf("p2p service: %w", err)
	}
    // 把 p2ps 传入到 bee 节点
	b.p2pService = p2ps
	b.p2pHalter = p2ps

	// localstore depends on batchstore
	var path string

	if o.DataDir != "" {
		logger.Infof("using datadir in: '%s'", o.DataDir)
		path = filepath.Join(o.DataDir, "localstore")
	}
	lo := &localstore.Options{
		Capacity:               o.CacheCapacity,
		OpenFilesLimit:         o.DBOpenFilesLimit,
		BlockCacheCapacity:     o.DBBlockCacheCapacity,
		WriteBufferSize:        o.DBWriteBufferSize,
		DisableSeeksCompaction: o.DBDisableSeeksCompaction,
	}

	storer, err := localstore.New(path, swarmAddress.Bytes(), lo, logger)
	if err != nil {
		return nil, fmt.Errorf("localstore: %w", err)
	}
	b.localstoreCloser = storer

	batchStore, err := batchstore.New(stateStore, storer.UnreserveBatch)
	if err != nil {
		return nil, fmt.Errorf("batchstore: %w", err)
	}
	validStamp := postage.ValidStamp(batchStore)
	post, err := postage.NewService(stateStore, batchStore, chainID)
	if err != nil {
		return nil, fmt.Errorf("postage service load: %w", err)
	}
	b.postageServiceCloser = post

	var (
		postageContractService postagecontract.Interface
		batchSvc               postage.EventUpdater
		eventListener          postage.Listener
	)

	var postageSyncStart uint64 = 0
	if !o.Standalone {
		postageContractAddress, startBlock, found := listener.DiscoverAddresses(chainID)
		if o.PostageContractAddress != "" {
			if !common.IsHexAddress(o.PostageContractAddress) {
				return nil, errors.New("malformed postage stamp address")
			}
			postageContractAddress = common.HexToAddress(o.PostageContractAddress)
		} else if !found {
			return nil, errors.New("no known postage stamp addresses for this network")
		}
		if found {
			postageSyncStart = startBlock
		}

		eventListener = listener.New(logger, swapBackend, postageContractAddress, o.BlockTime, &pidKiller{node: b})
		b.listenerCloser = eventListener

		batchSvc = batchservice.New(stateStore, batchStore, logger, eventListener, overlayEthAddress.Bytes(), post)

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
	}

	if !o.Standalone {
		if natManager := p2ps.NATManager(); natManager != nil {
			// wait for nat manager to init
			logger.Debug("initializing NAT manager")
			select {
			case <-natManager.Ready():
				// this is magic sleep to give NAT time to sync the mappings
				// this is a hack, kind of alchemy and should be improved
				time.Sleep(3 * time.Second)
				logger.Debug("NAT manager initialized")
			case <-time.After(10 * time.Second):
				logger.Warning("NAT manager init timeout")
			}
		}
	}

	// Construct protocols.
	pingPong := pingpong.New(p2ps, logger, tracer)

    // 添加 pingPong 协议
	if err = p2ps.AddProtocol(pingPong.Protocol()); err != nil {
		return nil, fmt.Errorf("pingpong service: %w", err)
	}

	hive := hive.New(p2ps, addressbook, networkID, logger)
    // 添加 hive 协议
	if err = p2ps.AddProtocol(hive.Protocol()); err != nil {
		return nil, fmt.Errorf("hive service: %w", err)
	}

	var bootnodes []ma.Multiaddr
	if o.Standalone {
		logger.Info("Starting node in standalone mode, no p2p connections will be made or accepted")
	} else {
		for _, a := range o.Bootnodes {
			addr, err := ma.NewMultiaddr(a)
			if err != nil {
				logger.Debugf("multiaddress fail %s: %v", a, err)
				logger.Warningf("invalid bootnode address %s", a)
				continue
			}

			bootnodes = append(bootnodes, addr)
		}
	}

	var swapService *swap.Service

	metricsDB, err := shed.NewDBWrap(stateStore.DB())
	if err != nil {
		return nil, fmt.Errorf("unable to create metrics storage for kademlia: %w", err)
	} 

    // p2ps 作为参数，创建 kad 服务
	kad := kademlia.New(swarmAddress, addressbook, hive, p2ps, metricsDB, logger, kademlia.Options{Bootnodes: bootnodes, StandaloneMode: o.Standalone, BootnodeMode: o.BootnodeMode})
	b.topologyCloser = kad
	b.topologyHalter = kad
	hive.SetAddPeersHandler(kad.AddPeers)
	p2ps.SetPickyNotifier(kad)
	batchStore.SetRadiusSetter(kad)

	if batchSvc != nil {
		syncedChan, err := batchSvc.Start(postageSyncStart)
		if err != nil {
			return nil, fmt.Errorf("unable to start batch service: %w", err)
		}
		// wait for the postage contract listener to sync
		logger.Info("waiting to sync postage contract data, this may take a while... more info available in Debug loglevel")

		// arguably this is not a very nice solution since we dont support
		// interrupts at this stage of the application lifecycle. some changes
		// would be needed on the cmd level to support context cancellation at
		// this stage
		<-syncedChan

	}

	minThreshold := big.NewInt(2 * refreshRate)

	paymentThreshold, ok := new(big.Int).SetString(o.PaymentThreshold, 10)
	if !ok {
		return nil, fmt.Errorf("invalid payment threshold: %s", paymentThreshold)
	}

	pricer := pricer.NewFixedPricer(swarmAddress, basePrice)

	if paymentThreshold.Cmp(minThreshold) < 0 {
		return nil, fmt.Errorf("payment threshold below minimum generally accepted value, need at least %s", minThreshold)
	}

	pricing := pricing.New(p2ps, logger, paymentThreshold, minThreshold)

    // 添加 pricing 协议
	if err = p2ps.AddProtocol(pricing.Protocol()); err != nil {
		return nil, fmt.Errorf("pricing service: %w", err)
	}

	addrs, err := p2ps.Addresses()
	if err != nil {
		return nil, fmt.Errorf("get server addresses: %w", err)
	}

	for _, addr := range addrs {
		logger.Debugf("p2p address: %s", addr)
	}

	paymentTolerance, ok := new(big.Int).SetString(o.PaymentTolerance, 10)
	if !ok {
		return nil, fmt.Errorf("invalid payment tolerance: %s", paymentTolerance)
	}
	paymentEarly, ok := new(big.Int).SetString(o.PaymentEarly, 10)
	if !ok {
		return nil, fmt.Errorf("invalid payment early: %s", paymentEarly)
	}

	acc, err := accounting.NewAccounting(
		paymentThreshold,
		paymentTolerance,
		paymentEarly,
		logger,
		stateStore,
		pricing,
		big.NewInt(refreshRate),
		p2ps,
	)
	if err != nil {
		return nil, fmt.Errorf("accounting: %w", err)
	}
	b.accountingCloser = acc

	pseudosettleService := pseudosettle.New(p2ps, logger, stateStore, acc, big.NewInt(refreshRate), p2ps)

    // 添加伪结算协议
	if err = p2ps.AddProtocol(pseudosettleService.Protocol()); err != nil {
		return nil, fmt.Errorf("pseudosettle service: %w", err)
	}

	acc.SetRefreshFunc(pseudosettleService.Pay)

	if o.SwapEnable {
		var priceOracle priceoracle.Service
		swapService, priceOracle, err = InitSwap(
			p2ps,
			logger,
			stateStore,
			networkID,
			overlayEthAddress,
			chequebookService,
			chequeStore,
			cashoutService,
			acc,
			o.PriceOracleAddress,
			chainID,
			transactionService,
		)
		if err != nil {
			return nil, err
		}
		b.priceOracleCloser = priceOracle
		acc.SetPayFunc(swapService.Pay)
	}

	pricing.SetPaymentThresholdObserver(acc)

	retrieve := retrieval.New(swarmAddress, storer, p2ps, kad, logger, acc, pricer, tracer)
	tagService := tags.NewTags(stateStore, logger)
	b.tagsCloser = tagService

	pssService := pss.New(pssPrivateKey, logger)
	b.pssCloser = pssService

	var ns storage.Storer
	if o.GlobalPinningEnabled {
		// create recovery callback for content repair
		recoverFunc := recovery.NewCallback(pssService)
		ns = netstore.New(storer, validStamp, recoverFunc, retrieve, logger)
	} else {
		ns = netstore.New(storer, validStamp, nil, retrieve, logger)
	}

	traversalService := traversal.New(ns)

	pinningService := pinning.NewService(storer, stateStore, traversalService)

	pushSyncProtocol := pushsync.New(swarmAddress, p2ps, storer, kad, tagService, o.FullNodeMode, pssService.TryUnwrap, validStamp, logger, acc, pricer, signer, tracer, warmupTime)

	// set the pushSyncer in the PSS
	pssService.SetPushSyncer(pushSyncProtocol)

	if o.GlobalPinningEnabled {
		// register function for chunk repair upon receiving a trojan message
		chunkRepairHandler := recovery.NewRepairHandler(ns, logger, pushSyncProtocol)
		b.recoveryHandleCleanup = pssService.Register(recovery.Topic, chunkRepairHandler)
	}

	pusherService := pusher.New(networkID, storer, kad, pushSyncProtocol, tagService, logger, tracer, warmupTime)
	b.pusherCloser = pusherService

	pullStorage := pullstorage.New(storer)

	pullSyncProtocol := pullsync.New(p2ps, pullStorage, pssService.TryUnwrap, validStamp, logger)
	b.pullSyncCloser = pullSyncProtocol

	var pullerService *puller.Puller
	if o.FullNodeMode {
		pullerService := puller.New(stateStore, kad, pullSyncProtocol, logger, puller.Options{}, warmupTime)
		b.pullerCloser = pullerService
	}

	retrieveProtocolSpec := retrieve.Protocol()
	pushSyncProtocolSpec := pushSyncProtocol.Protocol()
	pullSyncProtocolSpec := pullSyncProtocol.Protocol()

	if o.FullNodeMode {
		logger.Info("starting in full mode")
	} else {
		logger.Info("starting in light mode")
		p2p.WithBlocklistStreams(p2p.DefaultBlocklistTime, retrieveProtocolSpec)
		p2p.WithBlocklistStreams(p2p.DefaultBlocklistTime, pushSyncProtocolSpec)
		p2p.WithBlocklistStreams(p2p.DefaultBlocklistTime, pullSyncProtocolSpec)
	}

    // 添加检索协议
	if err = p2ps.AddProtocol(retrieveProtocolSpec); err != nil {
		return nil, fmt.Errorf("retrieval service: %w", err)
	}
    // 添加推同步协议
	if err = p2ps.AddProtocol(pushSyncProtocolSpec); err != nil {
		return nil, fmt.Errorf("pushsync service: %w", err)
	}
    // 添加拉同步协议
	if err = p2ps.AddProtocol(pullSyncProtocolSpec); err != nil {
		return nil, fmt.Errorf("pullsync protocol: %w", err)
	}

	multiResolver := multiresolver.NewMultiResolver(
		multiresolver.WithConnectionConfigs(o.ResolverConnectionCfgs),
		multiresolver.WithLogger(o.Logger),
	)
	b.resolverCloser = multiResolver

	var apiService api.Service
	if o.APIAddr != "" {
		// API server
		feedFactory := factory.New(ns)
		steward := steward.New(storer, traversalService, pushSyncProtocol)
		apiService = api.New(tagService, ns, multiResolver, pssService, traversalService, pinningService, feedFactory, post, postageContractService, steward, signer, logger, tracer, api.Options{
			CORSAllowedOrigins: o.CORSAllowedOrigins,
			GatewayMode:        o.GatewayMode,
			WsPingPeriod:       60 * time.Second,
		})
		apiListener, err := net.Listen("tcp", o.APIAddr)
		if err != nil {
			return nil, fmt.Errorf("api listener: %w", err)
		}

		apiServer := &http.Server{
			IdleTimeout:       30 * time.Second,
			ReadHeaderTimeout: 3 * time.Second,
			Handler:           apiService,
			ErrorLog:          log.New(b.errorLogWriter, "", 0),
		}

		go func() {
			logger.Infof("api address: %s", apiListener.Addr())

			if err := apiServer.Serve(apiListener); err != nil && err != http.ErrServerClosed {
				logger.Debugf("api server: %v", err)
				logger.Error("unable to serve api")
			}
		}()

		b.apiServer = apiServer
		b.apiCloser = apiService
	}

	if debugAPIService != nil {
		// register metrics from components
		debugAPIService.MustRegisterMetrics(p2ps.Metrics()...)
		debugAPIService.MustRegisterMetrics(pingPong.Metrics()...)
		debugAPIService.MustRegisterMetrics(acc.Metrics()...)
		debugAPIService.MustRegisterMetrics(storer.Metrics()...)
		debugAPIService.MustRegisterMetrics(kad.Metrics()...)

		if pullerService != nil {
			debugAPIService.MustRegisterMetrics(pullerService.Metrics()...)
		}

		debugAPIService.MustRegisterMetrics(pushSyncProtocol.Metrics()...)
		debugAPIService.MustRegisterMetrics(pusherService.Metrics()...)
		debugAPIService.MustRegisterMetrics(pullSyncProtocol.Metrics()...)
		debugAPIService.MustRegisterMetrics(pullStorage.Metrics()...)
		debugAPIService.MustRegisterMetrics(retrieve.Metrics()...)
		debugAPIService.MustRegisterMetrics(lightNodes.Metrics()...)

		if bs, ok := batchStore.(metrics.Collector); ok {
			debugAPIService.MustRegisterMetrics(bs.Metrics()...)
		}

		if eventListener != nil {
			if ls, ok := eventListener.(metrics.Collector); ok {
				debugAPIService.MustRegisterMetrics(ls.Metrics()...)
			}
		}

		if pssServiceMetrics, ok := pssService.(metrics.Collector); ok {
			debugAPIService.MustRegisterMetrics(pssServiceMetrics.Metrics()...)
		}

		if apiService != nil {
			debugAPIService.MustRegisterMetrics(apiService.Metrics()...)
		}
		if l, ok := logger.(metrics.Collector); ok {
			debugAPIService.MustRegisterMetrics(l.Metrics()...)
		}

		debugAPIService.MustRegisterMetrics(pseudosettleService.Metrics()...)

		if swapService != nil {
			debugAPIService.MustRegisterMetrics(swapService.Metrics()...)
		}

		// inject dependencies and configure full debug api http path routes
		debugAPIService.Configure(p2ps, pingPong, kad, lightNodes, storer, tagService, acc, pseudosettleService, o.SwapEnable, swapService, chequebookService, batchStore, transactionService)
	}

	if err := kad.Start(p2pCtx); err != nil {
		return nil, err
	}
    // kad 启动后，p2ps 进入准备状态
	p2ps.Ready()

	return b, nil
}

```

在创建 Bee 节点的时候，会创建 p2p 服务，然后添加协议，分别是：

- pingPong
- hive
- pricing
- pseudosettleService：伪结算服务，在真正结算之前，先会进行伪结算
- retrieveProtocol
- pushSyncProtocol
- pullSyncProtocol

然后，通过会创建 kad 网络，p2p 服务进入准备状态，ready channal 会关闭。

## 创建 p2ps




参考：

[200行go代码实现区块链之五——P2P网络](https://blog.csdn.net/liuzhijun301/article/details/80433557)