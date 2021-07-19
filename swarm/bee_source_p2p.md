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

libp2p.go

```go

func New(ctx context.Context, signer beecrypto.Signer, networkID uint64, overlay swarm.Address, addr string, ab addressbook.Putter, storer storage.StateStorer, lightNodes *lightnode.Container, swapBackend handshake.SenderMatcher, logger logging.Logger, tracer *tracing.Tracer, o Options) (*Service, error) {
	host, port, err := net.SplitHostPort(addr)
	if err != nil {
		return nil, fmt.Errorf("address: %w", err)
	}

	ip4Addr := "0.0.0.0"
	ip6Addr := "::"

	if host != "" {
		ip := net.ParseIP(host)
		if ip4 := ip.To4(); ip4 != nil {
			ip4Addr = ip4.String()
			ip6Addr = ""
		} else if ip6 := ip.To16(); ip6 != nil {
			ip6Addr = ip6.String()
			ip4Addr = ""
		}
	}

	var listenAddrs []string
	if ip4Addr != "" {
		listenAddrs = append(listenAddrs, fmt.Sprintf("/ip4/%s/tcp/%s", ip4Addr, port))
		if o.EnableWS {
			listenAddrs = append(listenAddrs, fmt.Sprintf("/ip4/%s/tcp/%s/ws", ip4Addr, port))
		}
		if o.EnableQUIC {
			listenAddrs = append(listenAddrs, fmt.Sprintf("/ip4/%s/udp/%s/quic", ip4Addr, port))
		}
	}

	if ip6Addr != "" {
		listenAddrs = append(listenAddrs, fmt.Sprintf("/ip6/%s/tcp/%s", ip6Addr, port))
		if o.EnableWS {
			listenAddrs = append(listenAddrs, fmt.Sprintf("/ip6/%s/tcp/%s/ws", ip6Addr, port))
		}
		if o.EnableQUIC {
			listenAddrs = append(listenAddrs, fmt.Sprintf("/ip6/%s/udp/%s/quic", ip6Addr, port))
		}
	}

	security := libp2p.DefaultSecurity
	libp2pPeerstore := pstoremem.NewPeerstore()

	var natManager basichost.NATManager

	opts := []libp2p.Option{
		libp2p.ListenAddrStrings(listenAddrs...),
		security,
		// Use dedicated peerstore instead the global DefaultPeerstore
		libp2p.Peerstore(libp2pPeerstore),
	}

	if o.NATAddr == "" {
		opts = append(opts,
			libp2p.NATManager(func(n network.Network) basichost.NATManager {
				natManager = basichost.NewNATManager(n)
				return natManager
			}),
		)
	}

	if o.PrivateKey != nil {
		opts = append(opts,
			libp2p.Identity((*crypto.Secp256k1PrivateKey)(o.PrivateKey)),
		)
	}

	transports := []libp2p.Option{
		libp2p.Transport(func(u *tptu.Upgrader) *tcp.TcpTransport {
			t := tcp.NewTCPTransport(u)
			t.DisableReuseport = true
			return t
		}),
	}

	if o.EnableWS {
		transports = append(transports, libp2p.Transport(ws.New))
	}

	if o.EnableQUIC {
		transports = append(transports, libp2p.Transport(libp2pquic.NewTransport))
	}

	if o.Standalone {
		opts = append(opts, libp2p.NoListenAddrs)
	}

	opts = append(opts, transports...)

	h, err := libp2p.New(ctx, opts...)
	if err != nil {
		return nil, err
	}

	// Support same non default security and transport options as
	// original host.
	dialer, err := libp2p.New(ctx, append(transports, security)...)
	if err != nil {
		return nil, err
	}

	// If you want to help other peers to figure out if they are behind
	// NATs, you can launch the server-side of AutoNAT too (AutoRelay
	// already runs the client)
	if _, err = autonat.New(ctx, h, autonat.EnableService(dialer.Network())); err != nil {
		return nil, fmt.Errorf("autonat: %w", err)
	}

	var advertisableAddresser handshake.AdvertisableAddressResolver
	var natAddrResolver *staticAddressResolver
	if o.NATAddr == "" {
		advertisableAddresser = &UpnpAddressResolver{
			host: h,
		}
	} else {
		natAddrResolver, err = newStaticAddressResolver(o.NATAddr, net.LookupIP)
		if err != nil {
			return nil, fmt.Errorf("static nat: %w", err)
		}
		advertisableAddresser = natAddrResolver
	}

	handshakeService, err := handshake.New(signer, advertisableAddresser, swapBackend, overlay, networkID, o.FullNode, o.Transaction, o.WelcomeMessage, logger)
	if err != nil {
		return nil, fmt.Errorf("handshake service: %w", err)
	}

	peerRegistry := newPeerRegistry()
	s := &Service{
		ctx:               ctx,
		host:              h,
		natManager:        natManager,
		natAddrResolver:   natAddrResolver,
		autonatDialer:     dialer,
		handshakeService:  handshakeService,
		libp2pPeerstore:   libp2pPeerstore,
		metrics:           newMetrics(),
		networkID:         networkID,
		peers:             peerRegistry,
		addressbook:       ab,
		blocklist:         blocklist.NewBlocklist(storer),
		logger:            logger,
		tracer:            tracer,
		connectionBreaker: breaker.NewBreaker(breaker.Options{}), // use default options
		ready:             make(chan struct{}),
		halt:              make(chan struct{}),
		lightNodes:        lightNodes,
	}

	peerRegistry.setDisconnecter(s)

	s.lightNodeLimit = defaultLightNodeLimit
	if o.LightNodeLimit > 0 {
		s.lightNodeLimit = o.LightNodeLimit
	}

	// Construct protocols.
	id := protocol.ID(p2p.NewSwarmStreamName(handshake.ProtocolName, handshake.ProtocolVersion, handshake.StreamName))
	matcher, err := s.protocolSemverMatcher(id)
	if err != nil {
		return nil, fmt.Errorf("protocol version match %s: %w", id, err)
	}

	s.host.SetStreamHandlerMatch(id, matcher, s.handleIncoming)

	h.Network().SetConnHandler(func(_ network.Conn) {
		s.metrics.HandledConnectionCount.Inc()
	})

	h.Network().Notify(peerRegistry)       // update peer registry on network events
	h.Network().Notify(s.handshakeService) // update handshake service on network events
	return s, nil
}

```

在 Bee　节点启动的时候，会创建 p2ps　，该方法主要流程：

- 解析　p2pAddr　，生成地址格式：
- 调用　libp2p（ProtocalLabs开源）　库的ListenAddrStrings方法，进入监听
- 创建　natAddrResolver　解析器
- 通过　libp2p　创建　host，然后通知节点注册

NatAddrr：主要打开外部到当前节点的通信通道，在日志里面收到inbound，代表通道已经打开。

libp2p接口案例：https://github.com/libp2p/go-libp2p/tree/master/examples


## AddProtocol　方法

libp2p.go

```go

func (s *Service) AddProtocol(p p2p.ProtocolSpec) (err error) {
	for _, ss := range p.StreamSpecs {
		ss := ss
		id := protocol.ID(p2p.NewSwarmStreamName(p.Name, p.Version, ss.Name))
		matcher, err := s.protocolSemverMatcher(id)
		if err != nil {
			return fmt.Errorf("protocol version match %s: %w", id, err)
		}

		s.host.SetStreamHandlerMatch(id, matcher, func(streamlibp2p network.Stream) {
			peerID := streamlibp2p.Conn().RemotePeer()
			overlay, found := s.peers.overlay(peerID)
			if !found {
				_ = streamlibp2p.Reset()
				s.logger.Debugf("overlay address for peer %q not found", peerID)
				return
			}
			full, found := s.peers.fullnode(peerID)
			if !found {
				_ = streamlibp2p.Reset()
				s.logger.Debugf("fullnode info for peer %q not found", peerID)
				return
			}

			stream := newStream(streamlibp2p)

			// exchange headers
			if err := handleHeaders(ss.Headler, stream, overlay); err != nil {
				s.logger.Debugf("handle protocol %s/%s: stream %s: peer %s: handle headers: %v", p.Name, p.Version, ss.Name, overlay, err)
				_ = stream.Reset()
				return
			}

			ctx, cancel := context.WithCancel(s.ctx)

			s.peers.addStream(peerID, streamlibp2p, cancel)
			defer s.peers.removeStream(peerID, streamlibp2p)

			// tracing: get span tracing context and add it to the context
			// silently ignore if the peer is not providing tracing
			ctx, err := s.tracer.WithContextFromHeaders(ctx, stream.Headers())
			if err != nil && !errors.Is(err, tracing.ErrContextNotFound) {
				s.logger.Debugf("handle protocol %s/%s: stream %s: peer %s: get tracing context: %v", p.Name, p.Version, ss.Name, overlay, err)
				_ = stream.Reset()
				return
			}

			logger := tracing.NewLoggerWithTraceID(ctx, s.logger)

			s.metrics.HandledStreamCount.Inc()
			if err := ss.Handler(ctx, p2p.Peer{Address: overlay, FullNode: full}, stream); err != nil {
				var de *p2p.DisconnectError
				if errors.As(err, &de) {
					_ = stream.Reset()
					_ = s.Disconnect(overlay)
				}

				var bpe *p2p.BlockPeerError
				if errors.As(err, &bpe) {
					_ = stream.Reset()
					if err := s.Blocklist(overlay, bpe.Duration()); err != nil {
						logger.Debugf("blocklist: could not blocklist peer %s: %v", peerID, err)
						logger.Errorf("unable to blocklist peer %v", peerID)
					}
					logger.Tracef("blocklisted a peer %s", peerID)
				}
				// count unexpected requests
				if errors.Is(err, p2p.ErrUnexpected) {
					s.metrics.UnexpectedProtocolReqCount.Inc()
				}
				logger.Debugf("could not handle protocol %s/%s: stream %s: peer %s: error: %v", p.Name, p.Version, ss.Name, overlay, err)
				return
			}
		})
	}

	s.protocolsmu.Lock()
	s.protocols = append(s.protocols, p)
	s.protocolsmu.Unlock()
	return nil
}

```

在前面已经讲过会有７个协议被添加，在　AddProtocal　方法中，会调用 libp2p host　的　SetStreamHandlerMatch　方法：　设置　host mux　（将传入流复用到协议处理程序） 的协议处理器。 

libp2p.go　中的　Connect　和　Disconnect 方法将在下一节分析。

参考：

[200行go代码实现区块链之五——P2P网络](https://blog.csdn.net/liuzhijun301/article/details/80433557)