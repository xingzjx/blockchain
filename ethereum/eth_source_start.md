# 以太坊源码解读：启动流程

- [以太坊源码解读：启动流程](#以太坊源码解读启动流程)
  - [cmd模块](#cmd模块)
    - [prepare](#prepare)
    - [makeFullNode](#makefullnode)
    - [startNode](#startnode)
  - [node模块](#node模块)
    - [创建节点](#创建节点)
    - [启动节点](#启动节点)
  - [p2p模块](#p2p模块)
    - [setupLocalNode](#setuplocalnode)
    - [setupListening](#setuplistening)
    - [setupDiscovery](#setupdiscovery)
    - [setupDialScheduler](#setupdialscheduler)

分析环境：go-ethereum,　版本：1.10

本文的思路，以模块划分为主，并且结合geth客户端的执行时序和打印的日志进行梳理分析。

打印日志如下：

```log
INFO [07-12|03:41:39.922] Starting Geth on Görli testnet... 
INFO [07-12|03:41:39.924] Maximum peer count                       ETH=50 LES=0 total=50
WARN [07-12|03:41:39.925] The flag --rpc is deprecated and will be removed June 2021, please use --http 
WARN [07-12|03:41:39.925] The flag --rpcaddr is deprecated and will be removed June 2021, please use --http.addr 
INFO [07-12|03:41:39.925] Smartcard socket not found, disabling    err="stat /run/pcscd/pcscd.comm: no such file or directory"
INFO [07-12|03:41:39.925] Set global gas cap                       cap=50,000,000
INFO [07-12|03:41:39.926] Allocated trie memory caches             clean=154.00MiB dirty=256.00MiB
INFO [07-12|03:41:39.926] Allocated cache and file handles         database=/data/data7/eth/data/geth/chaindata cache=512.00MiB handles=524,288
INFO [07-12|03:41:39.952] Opened ancient database                  database=/data/data7/eth/data/geth/chaindata/ancient readonly=false
INFO [07-12|03:41:39.964] Persisted trie from memory database      nodes=361 size=51.17KiB time=1.696604ms gcnodes=0 gcsize=0.00B gctime=0s livenodes=1 livesize=0.00B
INFO [07-12|03:41:39.964] Initialised chain configuration          config="{ChainID: 5 Homestead: 0 DAO: <nil> DAOSupport: true EIP150: 0 EIP155: 0 EIP158: 0 Byzantium: 0 Constantinople: 0 Petersburg: 0 Istanbul: 1561651, Muir Glacier: <nil>, Berlin: 4460644, London: 5062605, Engine: clique}"
INFO [07-12|03:41:39.965] Initialising Ethereum protocol           network=5 dbversion=8
INFO [07-12|03:41:39.966] Loaded most recent local header          number=0 hash=bf7e33..b88c1a td=1 age=2y5mo3w
INFO [07-12|03:41:39.966] Loaded most recent local full block      number=0 hash=bf7e33..b88c1a td=1 age=2y5mo3w
INFO [07-12|03:41:39.966] Loaded most recent local fast block      number=0 hash=bf7e33..b88c1a td=1 age=2y5mo3w
WARN [07-12|03:41:39.966] Loaded snapshot journal                  diskroot=5d6cde..8b3008 diffs=missing
INFO [07-12|03:41:39.966] Resuming state snapshot generation       root=5d6cde..8b3008 accounts=0 slots=0 storage=0.00B elapsed="73.759µs"
INFO [07-12|03:41:39.966] Loaded local transaction journal         transactions=0 dropped=0
INFO [07-12|03:41:39.967] Regenerated local transaction journal    transactions=0 accounts=0
INFO [07-12|03:41:39.967] Gasprice oracle is ignoring threshold set threshold=2
WARN [07-12|03:41:39.968] Unclean shutdown detected                booted=2021-07-12T03:39:07+0000 age=2m32s
INFO [07-12|03:41:39.968] Starting peer-to-peer node               instance=Geth/v1.10.4-stable-aa637fd3/linux-amd64/go1.16.4
INFO [07-12|03:41:39.978] Generated state snapshot                 accounts=260 slots=0 storage=9.69KiB elapsed=12.109ms
INFO [07-12|03:41:39.978] New local node record                    seq=1 id=62c09d73f246a60c ip=127.0.0.1 udp=30303 tcp=30303
INFO [07-12|03:41:39.979] Started P2P networking                   self=enode://87eec0d359a14ebe9bd386c43f8e468e9074137471d5279e8aa196013760f4c297d6ba5cec352c0932023545d9f3851206c7962e6c6f36f13978782166f8af96@127.0.0.1:30303
INFO [07-12|03:41:39.980] IPC endpoint opened                      url=/data/data7/eth/data/geth.ipc
INFO [07-12|03:41:39.981] HTTP server started                      endpoint=[::]:8545 prefix= cors= vhosts=localhost
INFO [07-12|03:41:39.981] WebSocket enabled                        url=ws://[::]:8546
INFO [07-12|03:41:43.502] New local node record                    seq=2 id=62c09d73f246a60c ip=183.60.41.47 udp=30303 tcp=30303
WARN [07-12|03:41:50.978] Served eth_coinbase                      conn=195.123.222.16:34600 reqid=1 t="70.48µs" err="etherbase must be explicitly specified"
INFO [07-12|03:41:51.864] Looking for peers                        peercount=0 tried=1 static=0
INFO [07-12|03:42:03.484] Looking for peers                        peercount=0 tried=11 static=0
WARN [07-12|03:42:04.370] Served eth_coinbase                      reqid=3 t="49.36µs" err="etherbase must be explicitly specified"
WARN [07-12|03:42:08.031] Served miner_setEtherbase                conn=195.123.222.16:47558 reqid=1 t="38.39µs" err="the method miner_setEtherbase does not exist/is not available"
INFO [07-12|03:42:13.513] Block synchronisation started 
WARN [07-12|03:42:13.513] Enabling snapshot sync prototype 
INFO [07-12|03:42:14.473] Looking for peers                        peercount=1 tried=20 static=0
WARN [07-12|03:42:18.434] Served personal_importRawKey             conn=195.123.222.16:53146 reqid=1 t="24.42µs" err="the method personal_importRawKey does not exist/is not available"
WARN [07-12|03:42:19.048] Served personal_importRawKey             conn=195.123.222.16:58420 reqid=1 t="27.32µs" err="the method personal_importRawKey does not exist/is not available"
INFO [07-12|03:42:20.262] Imported new block headers               count=192 elapsed=37.096ms   number=192 hash=b85251..8090f8 age=2y5mo3w
INFO [07-12|03:42:20.264] Downloader queue stats                   receiptTasks=0 blockTasks=0 itemSize=650.00B throttle=8192
INFO [07-12|03:42:20.265] Wrote genesis to ancients 
INFO [07-12|03:42:20.288] Imported new block receipts              count=192 elapsed=22.908ms   number=192 hash=b85251..8090f8 age=2y5mo3w size=120.52KiB
INFO [07-12|03:42:20.523] Imported new block headers               count=192 elapsed=31.958ms   number=384 hash=684118..7c9bf9 age=2y5mo3w
INFO [07-12|03:42:20.546] Imported new block receipts              count=192 elapsed=20.075ms   number=384 hash=684118..7c9bf9 age=2y5mo3w size=120.31KiB
```

## cmd模块

cmd模块是以太坊的入口，当执行 geth　命令，启动一个以太坊客户端的时候，会执行 cmd/geth/main.go，在这个类里面用到了开源库：[github.com/codegangsta/cli](https://github.com/urfave/cli)，用来构建命令行应用。

接下来看，main.go这个类的init方法

cmd/geth/main.go

```go

func init() {
	// Initialize the CLI app and start Geth
	app.Action = geth
	app.HideVersion = true // we have a command to print the version
	app.Copyright = "Copyright 2013-2021 The go-ethereum Authors"
	app.Commands = []cli.Command{
		// See chaincmd.go:
		initCommand,
		importCommand,
		exportCommand,
		importPreimagesCommand,
		exportPreimagesCommand,
		removedbCommand,
		dumpCommand,
		dumpGenesisCommand,
		// See accountcmd.go:
		accountCommand,
		walletCommand,
		// See consolecmd.go:
		consoleCommand,
		attachCommand,
		javascriptCommand,
		// See misccmd.go:
		makecacheCommand,
		makedagCommand,
		versionCommand,
		versionCheckCommand,
		licenseCommand,
		// See config.go
		dumpConfigCommand,
		// see dbcmd.go
		dbCommand,
		// See cmd/utils/flags_legacy.go
		utils.ShowDeprecated,
		// See snapshot.go
		snapshotCommand,
	}
	sort.Sort(cli.CommandsByName(app.Commands))

	app.Flags = append(app.Flags, nodeFlags...)
	app.Flags = append(app.Flags, rpcFlags...)
	app.Flags = append(app.Flags, consoleFlags...)
	app.Flags = append(app.Flags, debug.Flags...)
	app.Flags = append(app.Flags, metricsFlags...)

	app.Before = func(ctx *cli.Context) error {
		return debug.Setup(ctx)
	}
	app.After = func(ctx *cli.Context) error {
		debug.Exit()
		prompt.Stdin.Close() // Resets terminal mode.
		return nil
	}
}

```

在init方法中定义了Flags，Action，Before以及After。

- Flags  :   在全局的var变量中，定义了四种flags，分别是nodeFlags,rpcFlags,consolefFlags,debugFlags，用来接收命令行下的参数。
- Before :　 会调用internal/debug/flags.go中的Setup方法，会初始化profile和log工具。
- Action : 　定义了以太坊启动过程的主要过程，后面重点分析代码。
- After  :   停止profile和log的工作，返回执行geth之前的命令行模式。

接下来，看Action中的geth方法，

cmd/geth/main.go

```go

func geth(ctx *cli.Context) error {
	if args := ctx.Args(); len(args) > 0 {
		return fmt.Errorf("invalid command: %q", args[0])
	}

	prepare(ctx)
	stack, backend := makeFullNode(ctx)
	defer stack.Close()

	startNode(ctx, stack, backend)
	stack.Wait()
	return nil
}

```
在geth方法中，执行了三个重要的方法，prepare，makeFullNode以及startNode

### prepare

cmd/geth/main.go

```go
func prepare(ctx *cli.Context) {
	// If we're running a known preset, log it for convenience.
	switch {
	case ctx.GlobalIsSet(utils.RopstenFlag.Name):
		log.Info("Starting Geth on Ropsten testnet...")

	case ctx.GlobalIsSet(utils.RinkebyFlag.Name):
		log.Info("Starting Geth on Rinkeby testnet...")

	case ctx.GlobalIsSet(utils.GoerliFlag.Name):
		log.Info("Starting Geth on Görli testnet...")

	case ctx.GlobalIsSet(utils.CalaverasFlag.Name):
		log.Info("Starting Geth on Calaveras testnet...")

	case ctx.GlobalIsSet(utils.DeveloperFlag.Name):
		log.Info("Starting Geth in ephemeral dev mode...")

	case !ctx.GlobalIsSet(utils.NetworkIdFlag.Name):
		log.Info("Starting Geth on Ethereum mainnet...")
	}
	// If we're a full node on mainnet without --cache specified, bump default cache allowance
	if ctx.GlobalString(utils.SyncModeFlag.Name) != "light" && !ctx.GlobalIsSet(utils.CacheFlag.Name) && !ctx.GlobalIsSet(utils.NetworkIdFlag.Name) {
		// Make sure we're not on any supported preconfigured testnet either
		if !ctx.GlobalIsSet(utils.RopstenFlag.Name) && !ctx.GlobalIsSet(utils.RinkebyFlag.Name) && !ctx.GlobalIsSet(utils.GoerliFlag.Name) && !ctx.GlobalIsSet(utils.DeveloperFlag.Name) {
			// Nope, we're really on mainnet. Bump that cache up!
			log.Info("Bumping default cache on mainnet", "provided", ctx.GlobalInt(utils.CacheFlag.Name), "updated", 4096)
			ctx.GlobalSet(utils.CacheFlag.Name, strconv.Itoa(4096))
		}
	}
	// If we're running a light client on any network, drop the cache to some meaningfully low amount
	if ctx.GlobalString(utils.SyncModeFlag.Name) == "light" && !ctx.GlobalIsSet(utils.CacheFlag.Name) {
		log.Info("Dropping default light client cache", "provided", ctx.GlobalInt(utils.CacheFlag.Name), "updated", 128)
		ctx.GlobalSet(utils.CacheFlag.Name, strconv.Itoa(128))
	}

	// Start metrics export if enabled
	utils.SetupMetrics(ctx)

	// Start system runtime metrics collection
	go metrics.CollectProcessMetrics(3 * time.Second)
}
```
该方法准备操作 **memory cache allowance** 和设置 **metrics system**。

### makeFullNode

cmd/geth/config.go

```go
func makeFullNode(ctx *cli.Context) (*node.Node, ethapi.Backend) {
	stack, cfg := makeConfigNode(ctx)
	if ctx.GlobalIsSet(utils.OverrideLondonFlag.Name) {
		cfg.Eth.OverrideLondon = new(big.Int).SetUint64(ctx.GlobalUint64(utils.OverrideLondonFlag.Name))
	}
	backend, eth := utils.RegisterEthService(stack, &cfg.Eth)

	// Configure catalyst.
	if ctx.GlobalBool(utils.CatalystFlag.Name) {
		if eth == nil {
			utils.Fatalf("Catalyst does not work in light client mode.")
		}
		if err := catalyst.Register(stack, eth); err != nil {
			utils.Fatalf("%v", err)
		}
	}

	// Configure GraphQL if requested
	if ctx.GlobalIsSet(utils.GraphQLEnabledFlag.Name) {
		utils.RegisterGraphQLService(stack, backend, cfg.Node)
	}
	// Add the Ethereum Stats daemon if requested.
	if cfg.Ethstats.URL != "" {
		utils.RegisterEthStatsService(stack, backend, cfg.Ethstats.URL)
	}
	return stack, backend
}
```
makeFullNode方法：
- makeConfigNode，会创建一个Node
- RegisterEthService，注册以太坊服务，返回backend
- RegisterGraphQLService，可选服务
- RegisterEthStatsService，可选服务

cmd/geth/config.go

```go
func makeConfigNode(ctx *cli.Context) (*node.Node, gethConfig) {
	// Load defaults.
	cfg := gethConfig{
		Eth:     ethconfig.Defaults,
		Node:    defaultNodeConfig(),
		Metrics: metrics.DefaultConfig,
	}

	// Load config file.
	if file := ctx.GlobalString(configFileFlag.Name); file != "" {
		if err := loadConfig(file, &cfg); err != nil {
			utils.Fatalf("%v", err)
		}
	}

	// Apply flags.
	utils.SetNodeConfig(ctx, &cfg.Node)
	stack, err := node.New(&cfg.Node)
	if err != nil {
		utils.Fatalf("Failed to create the protocol stack: %v", err)
	}
	utils.SetEthConfig(ctx, stack, &cfg.Eth)
	if ctx.GlobalIsSet(utils.EthStatsURLFlag.Name) {
		cfg.Ethstats.URL = ctx.GlobalString(utils.EthStatsURLFlag.Name)
	}
	applyMetricConfig(ctx, &cfg)

	return stack, cfg
}
```
makeConfigNode方法会调用node模块，创建一个Node，Node模块创建的过程在后面会继续分析。

cmd/utils/flags.go

```go
func RegisterEthService(stack *node.Node, cfg *ethconfig.Config) (ethapi.Backend, *eth.Ethereum) {
	if cfg.SyncMode == downloader.LightSync {
		backend, err := les.New(stack, cfg)
		if err != nil {
			Fatalf("Failed to register the Ethereum service: %v", err)
		}
		stack.RegisterAPIs(tracers.APIs(backend.ApiBackend))
		return backend.ApiBackend, nil
	}
	backend, err := eth.New(stack, cfg)
	if err != nil {
		Fatalf("Failed to register the Ethereum service: %v", err)
	}
	if cfg.LightServ > 0 {
		_, err := les.NewLesServer(stack, backend, cfg)
		if err != nil {
			Fatalf("Failed to create the LES server: %v", err)
		}
	}
	stack.RegisterAPIs(tracers.APIs(backend.APIBackend))
	return backend.APIBackend, backend
}
```
RegisterEthService重要步骤如下：

- RegisterAPIs: 注册API到Node
- eth.New: 调用eth模块的backend.go，创建一个Ethereum，返回

### startNode

cmd/geth/main.go

```go
func startNode(ctx *cli.Context, stack *node.Node, backend ethapi.Backend) {
	debug.Memsize.Add("node", stack)

	// Start up the node itself
	utils.StartNode(ctx, stack)

	// Unlock any account specifically requested
	unlockAccounts(ctx, stack)

	// Register wallet event handlers to open and auto-derive wallets
	events := make(chan accounts.WalletEvent, 16)
	stack.AccountManager().Subscribe(events)

	// Create a client to interact with local geth node.
	rpcClient, err := stack.Attach()
	if err != nil {
		utils.Fatalf("Failed to attach to self: %v", err)
	}
	ethClient := ethclient.NewClient(rpcClient)

	go func() {
		// Open any wallets already attached
		for _, wallet := range stack.AccountManager().Wallets() {
			if err := wallet.Open(""); err != nil {
				log.Warn("Failed to open wallet", "url", wallet.URL(), "err", err)
			}
		}
		// Listen for wallet event till termination
		for event := range events {
			switch event.Kind {
			case accounts.WalletArrived:
				if err := event.Wallet.Open(""); err != nil {
					log.Warn("New wallet appeared, failed to open", "url", event.Wallet.URL(), "err", err)
				}
			case accounts.WalletOpened:
				status, _ := event.Wallet.Status()
				log.Info("New wallet appeared", "url", event.Wallet.URL(), "status", status)

				var derivationPaths []accounts.DerivationPath
				if event.Wallet.URL().Scheme == "ledger" {
					derivationPaths = append(derivationPaths, accounts.LegacyLedgerBaseDerivationPath)
				}
				derivationPaths = append(derivationPaths, accounts.DefaultBaseDerivationPath)

				event.Wallet.SelfDerive(derivationPaths, ethClient)

			case accounts.WalletDropped:
				log.Info("Old wallet dropped", "url", event.Wallet.URL())
				event.Wallet.Close()
			}
		}
	}()

	// Spawn a standalone goroutine for status synchronization monitoring,
	// close the node when synchronization is complete if user required.
	if ctx.GlobalBool(utils.ExitWhenSyncedFlag.Name) {
		go func() {
			sub := stack.EventMux().Subscribe(downloader.DoneEvent{})
			defer sub.Unsubscribe()
			for {
				event := <-sub.Chan()
				if event == nil {
					continue
				}
				done, ok := event.Data.(downloader.DoneEvent)
				if !ok {
					continue
				}
				if timestamp := time.Unix(int64(done.Latest.Time), 0); time.Since(timestamp) < 10*time.Minute {
					log.Info("Synchronisation completed", "latestnum", done.Latest.Number, "latesthash", done.Latest.Hash(),
						"age", common.PrettyAge(timestamp))
					stack.Close()
				}
			}
		}()
	}

	// Start auxiliary services if enabled
	if ctx.GlobalBool(utils.MiningEnabledFlag.Name) || ctx.GlobalBool(utils.DeveloperFlag.Name) {
		// Mining only makes sense if a full Ethereum node is running
		if ctx.GlobalString(utils.SyncModeFlag.Name) == "light" {
			utils.Fatalf("Light clients do not support mining")
		}
		ethBackend, ok := backend.(*eth.EthAPIBackend)
		if !ok {
			utils.Fatalf("Ethereum service not running: %v", err)
		}
		// Set the gas price to the limits from the CLI and start mining
		gasprice := utils.GlobalBig(ctx, utils.MinerGasPriceFlag.Name)
		ethBackend.TxPool().SetGasPrice(gasprice)
		// start mining
		threads := ctx.GlobalInt(utils.MinerThreadsFlag.Name)
		if err := ethBackend.StartMining(threads); err != nil {
			utils.Fatalf("Failed to start mining: %v", err)
		}
	}
}
```

startNode核心逻辑如下：

- utils.StartNode：启动一个系统节点
- unlockAccounts：解锁任何特别要求的帐户
- 启动 RPC 接口

cmd/utils/cmd.go

```go
func StartNode(ctx *cli.Context, stack *node.Node) {
	if err := stack.Start(); err != nil {
		Fatalf("Error starting protocol stack: %v", err)
	}
	go func() {
		sigc := make(chan os.Signal, 1)
		signal.Notify(sigc, syscall.SIGINT, syscall.SIGTERM)
		defer signal.Stop(sigc)

		minFreeDiskSpace := ethconfig.Defaults.TrieDirtyCache
		if ctx.GlobalIsSet(MinFreeDiskSpaceFlag.Name) {
			minFreeDiskSpace = ctx.GlobalInt(MinFreeDiskSpaceFlag.Name)
		} else if ctx.GlobalIsSet(CacheFlag.Name) || ctx.GlobalIsSet(CacheGCFlag.Name) {
			minFreeDiskSpace = ctx.GlobalInt(CacheFlag.Name) * ctx.GlobalInt(CacheGCFlag.Name) / 100
		}
		if minFreeDiskSpace > 0 {
			go monitorFreeDiskSpace(sigc, stack.InstanceDir(), uint64(minFreeDiskSpace)*1024*1024)
		}

		<-sigc
		log.Info("Got interrupt, shutting down...")
		go stack.Close()
		for i := 10; i > 0; i-- {
			<-sigc
			if i > 1 {
				log.Warn("Already shutting down, interrupt more to panic.", "times", i-1)
			}
		}
		debug.Exit() // ensure trace and CPU profile data is flushed.
		debug.LoudPanic("boom")
	}()
}
```
StartNode方法会调用Node模块，启动Node。


## node模块

node模块包括了节点创建，启动等相关工作。


### 创建节点

node/node.go

```go
func New(conf *Config) (*Node, error) {
	// Copy config and resolve the datadir so future changes to the current
	// working directory don't affect the node.
	confCopy := *conf
	conf = &confCopy
	if conf.DataDir != "" {
		absdatadir, err := filepath.Abs(conf.DataDir)
		if err != nil {
			return nil, err
		}
		conf.DataDir = absdatadir
	}
	if conf.Logger == nil {
		conf.Logger = log.New()
	}

	// Ensure that the instance name doesn't cause weird conflicts with
	// other files in the data directory.
	if strings.ContainsAny(conf.Name, `/\`) {
		return nil, errors.New(`Config.Name must not contain '/' or '\'`)
	}
	if conf.Name == datadirDefaultKeyStore {
		return nil, errors.New(`Config.Name cannot be "` + datadirDefaultKeyStore + `"`)
	}
	if strings.HasSuffix(conf.Name, ".ipc") {
		return nil, errors.New(`Config.Name cannot end in ".ipc"`)
	}

	node := &Node{
		config:        conf,
		inprocHandler: rpc.NewServer(),
		eventmux:      new(event.TypeMux),
		log:           conf.Logger,
		stop:          make(chan struct{}),
		server:        &p2p.Server{Config: conf.P2P},
		databases:     make(map[*closeTrackingDB]struct{}),
	}

	// Register built-in APIs.
	node.rpcAPIs = append(node.rpcAPIs, node.apis()...)

	// Acquire the instance directory lock.
	if err := node.openDataDir(); err != nil {
		return nil, err
	}
	// Ensure that the AccountManager method works before the node has started. We rely on
	// this in cmd/geth.
	am, ephemeralKeystore, err := makeAccountManager(conf)
	if err != nil {
		return nil, err
	}
	node.accman = am
	node.ephemKeystore = ephemeralKeystore

	// Initialize the p2p server. This creates the node key and discovery databases.
	node.server.Config.PrivateKey = node.config.NodeKey()
	node.server.Config.Name = node.config.NodeName()
	node.server.Config.Logger = node.log
	if node.server.Config.StaticNodes == nil {
		node.server.Config.StaticNodes = node.config.StaticNodes()
	}
	if node.server.Config.TrustedNodes == nil {
		node.server.Config.TrustedNodes = node.config.TrustedNodes()
	}
	if node.server.Config.NodeDatabase == "" {
		node.server.Config.NodeDatabase = node.config.NodeDB()
	}

	// Check HTTP/WS prefixes are valid.
	if err := validatePrefix("HTTP", conf.HTTPPathPrefix); err != nil {
		return nil, err
	}
	if err := validatePrefix("WebSocket", conf.WSPathPrefix); err != nil {
		return nil, err
	}

	// Configure RPC servers.
	node.http = newHTTPServer(node.log, conf.HTTPTimeouts)
	node.ws = newHTTPServer(node.log, rpc.DefaultHTTPTimeouts)
	node.ipc = newIPCServer(node.log, conf.IPCEndpoint())

	return node, nil
}
```

New方法的核心步骤：

- 定义Node的数据结构
- 初始化 P2P Server
- 配置RPC服务，包括HTTP,RPC和IPC方式

### 启动节点

node/node.go

```go
func (n *Node) Start() error {
	n.startStopLock.Lock()
	defer n.startStopLock.Unlock()

	n.lock.Lock()
	switch n.state {
	case runningState:
		n.lock.Unlock()
		return ErrNodeRunning
	case closedState:
		n.lock.Unlock()
		return ErrNodeStopped
	}
	n.state = runningState
	// open networking and RPC endpoints
	err := n.openEndpoints()
	lifecycles := make([]Lifecycle, len(n.lifecycles))
	copy(lifecycles, n.lifecycles)
	n.lock.Unlock()

	// Check if endpoint startup failed.
	if err != nil {
		n.doClose(nil)
		return err
	}
	// Start all registered lifecycles.
	var started []Lifecycle
	for _, lifecycle := range lifecycles {
		if err = lifecycle.Start(); err != nil {
			break
		}
		started = append(started, lifecycle)
	}
	// Check if any lifecycle failed to start.
	if err != nil {
		n.stopServices(started)
		n.doClose(nil)
	}
	return err
}
```

Start方法执行逻辑：

- openEndpoints：启动p2p和RPC
- 启动所有注册的Lifecycle（包括backends, services, 和 auxiliary）

```go
func (n *Node) openEndpoints() error {
	// start networking endpoints
	n.log.Info("Starting peer-to-peer node", "instance", n.server.Name)
	if err := n.server.Start(); err != nil {
		return convertFileLockError(err)
	}
	// start RPC endpoints
	err := n.startRPC()
	if err != nil {
		n.stopRPC()
		n.server.Stop()
	}
	return err
}
```
openEndpoints中，n.server.Start，会调用p2p模块，启动p2p服务，这个在后面的章节分析，然后调用startRPC方法，启动RPC。

node/node.go

```go
func (n *Node) startRPC() error {
	if err := n.startInProc(); err != nil {
		return err
	}

	// Configure IPC.
	if n.ipc.endpoint != "" {
		if err := n.ipc.start(n.rpcAPIs); err != nil {
			return err
		}
	}

	// Configure HTTP.
	if n.config.HTTPHost != "" {
		config := httpConfig{
			CorsAllowedOrigins: n.config.HTTPCors,
			Vhosts:             n.config.HTTPVirtualHosts,
			Modules:            n.config.HTTPModules,
			prefix:             n.config.HTTPPathPrefix,
		}
		if err := n.http.setListenAddr(n.config.HTTPHost, n.config.HTTPPort); err != nil {
			return err
		}
		if err := n.http.enableRPC(n.rpcAPIs, config); err != nil {
			return err
		}
	}

	// Configure WebSocket.
	if n.config.WSHost != "" {
		server := n.wsServerForPort(n.config.WSPort)
		config := wsConfig{
			Modules: n.config.WSModules,
			Origins: n.config.WSOrigins,
			prefix:  n.config.WSPathPrefix,
		}
		if err := server.setListenAddr(n.config.WSHost, n.config.WSPort); err != nil {
			return err
		}
		if err := server.enableWS(n.rpcAPIs, config); err != nil {
			return err
		}
	}

	if err := n.http.start(); err != nil {
		return err
	}
	return n.ws.start()
}
```

startRPC方法会调用rpcstack.go，启动ipc和htpp和rpc。

启动ipc方法如下：

node/rpcstack.go

```go
func (is *ipcServer) start(apis []rpc.API) error {
	is.mu.Lock()
	defer is.mu.Unlock()

	if is.listener != nil {
		return nil // already running
	}
	listener, srv, err := rpc.StartIPCEndpoint(is.endpoint, apis)
	if err != nil {
		is.log.Warn("IPC opening failed", "url", is.endpoint, "error", err)
		return err
	}
	is.log.Info("IPC endpoint opened", "url", is.endpoint)
	is.listener, is.srv = listener, srv
	return nil
}
```
StartIPCEndpoint方法会调用rpc模块，启动ipc。

node/rpcstack.go

```go
func (h *httpServer) start() error {
	h.mu.Lock()
	defer h.mu.Unlock()

	if h.endpoint == "" || h.listener != nil {
		return nil // already running or not configured
	}

	// Initialize the server.
	h.server = &http.Server{Handler: h}
	if h.timeouts != (rpc.HTTPTimeouts{}) {
		CheckTimeouts(&h.timeouts)
		h.server.ReadTimeout = h.timeouts.ReadTimeout
		h.server.WriteTimeout = h.timeouts.WriteTimeout
		h.server.IdleTimeout = h.timeouts.IdleTimeout
	}

	// Start the server.
	listener, err := net.Listen("tcp", h.endpoint)
	if err != nil {
		// If the server fails to start, we need to clear out the RPC and WS
		// configuration so they can be configured another time.
		h.disableRPC()
		h.disableWS()
		return err
	}
	h.listener = listener
	go h.server.Serve(listener)

	if h.wsAllowed() {
		url := fmt.Sprintf("ws://%v", listener.Addr())
		if h.wsConfig.prefix != "" {
			url += h.wsConfig.prefix
		}
		h.log.Info("WebSocket enabled", "url", url)
	}
	// if server is websocket only, return after logging
	if !h.rpcAllowed() {
		return nil
	}
	// Log http endpoint.
	h.log.Info("HTTP server started",
		"endpoint", listener.Addr(),
		"prefix", h.httpConfig.prefix,
		"cors", strings.Join(h.httpConfig.CorsAllowedOrigins, ","),
		"vhosts", strings.Join(h.httpConfig.Vhosts, ","),
	)

	// Log all handlers mounted on server.
	var paths []string
	for path := range h.handlerNames {
		paths = append(paths, path)
	}
	sort.Strings(paths)
	logged := make(map[string]bool, len(paths))
	for _, path := range paths {
		name := h.handlerNames[path]
		if !logged[name] {
			log.Info(name+" enabled", "url", "http://"+listener.Addr().String()+path)
			logged[name] = true
		}
	}
	return nil
}
```
该start方法包括了http和ws模式的启动逻辑。


## p2p模块

在node模块分析过，在openEndPoints执行的时候，会调用　p2p/server.go　的 start　方法，

p2p/server.go

```go
func (srv *Server) Start() (err error) {
	srv.lock.Lock()
	defer srv.lock.Unlock()
	if srv.running {
		return errors.New("server already running")
	}
	srv.running = true
	srv.log = srv.Config.Logger
	if srv.log == nil {
		srv.log = log.Root()
	}
	if srv.clock == nil {
		srv.clock = mclock.System{}
	}
	if srv.NoDial && srv.ListenAddr == "" {
		srv.log.Warn("P2P server will be useless, neither dialing nor listening")
	}

	// static fields
	if srv.PrivateKey == nil {
		return errors.New("Server.PrivateKey must be set to a non-nil key")
	}
	if srv.newTransport == nil {
		srv.newTransport = newRLPX
	}
	if srv.listenFunc == nil {
		srv.listenFunc = net.Listen
	}
	srv.quit = make(chan struct{})
	srv.delpeer = make(chan peerDrop)
	srv.checkpointPostHandshake = make(chan *conn)
	srv.checkpointAddPeer = make(chan *conn)
	srv.addtrusted = make(chan *enode.Node)
	srv.removetrusted = make(chan *enode.Node)
	srv.peerOp = make(chan peerOpFunc)
	srv.peerOpDone = make(chan struct{})

	if err := srv.setupLocalNode(); err != nil {
		return err
	}
	if srv.ListenAddr != "" {
		if err := srv.setupListening(); err != nil {
			return err
		}
	}
	if err := srv.setupDiscovery(); err != nil {
		return err
	}
	srv.setupDialScheduler()

	srv.loopWG.Add(1)
	go srv.run()
	return nil
}
```
p2p Server 的 start 方法核心逻辑：

- setupLocalNode　：启动本地节点
- setupListening　：启动监听
- setupDiscovery　：启动Discovery协议
- setupDialScheduler　：启动拨号调度器，主要是p2p连接
- run : 执行main loop ，通过 select 机制，处理节点操作逻辑

### setupLocalNode

p2p/server.go

```go
func (srv *Server) setupLocalNode() error {
	// Create the devp2p handshake.
	pubkey := crypto.FromECDSAPub(&srv.PrivateKey.PublicKey)
	srv.ourHandshake = &protoHandshake{Version: baseProtocolVersion, Name: srv.Name, ID: pubkey[1:]}
	for _, p := range srv.Protocols {
		srv.ourHandshake.Caps = append(srv.ourHandshake.Caps, p.cap())
	}
	sort.Sort(capsByNameAndVersion(srv.ourHandshake.Caps))

	// Create the local node.
	db, err := enode.OpenDB(srv.Config.NodeDatabase)
	if err != nil {
		return err
	}
	srv.nodedb = db
	srv.localnode = enode.NewLocalNode(db, srv.PrivateKey)
	srv.localnode.SetFallbackIP(net.IP{127, 0, 0, 1})
	// TODO: check conflicts
	for _, p := range srv.Protocols {
		for _, e := range p.Attributes {
			srv.localnode.Set(e)
		}
	}
	switch srv.NAT.(type) {
	case nil:
		// No NAT interface, do nothing.
	case nat.ExtIP:
		// ExtIP doesn't block, set the IP right away.
		ip, _ := srv.NAT.ExternalIP()
		srv.localnode.SetStaticIP(ip)
	default:
		// Ask the router about the IP. This takes a while and blocks startup,
		// do it in the background.
		srv.loopWG.Add(1)
		go func() {
			defer srv.loopWG.Done()
			if ip, err := srv.NAT.ExternalIP(); err == nil {
				srv.localnode.SetStaticIP(ip)
			}
		}()
	}
	return nil
}
```

### setupListening

p2p/server.go

```go
func (srv *Server) setupListening() error {
	// Launch the listener.
	listener, err := srv.listenFunc("tcp", srv.ListenAddr)
	if err != nil {
		return err
	}
	srv.listener = listener
	srv.ListenAddr = listener.Addr().String()

	// Update the local node record and map the TCP listening port if NAT is configured.
	if tcp, ok := listener.Addr().(*net.TCPAddr); ok {
		srv.localnode.Set(enr.TCP(tcp.Port))
		if !tcp.IP.IsLoopback() && srv.NAT != nil {
			srv.loopWG.Add(1)
			go func() {
				nat.Map(srv.NAT, srv.quit, "tcp", tcp.Port, tcp.Port, "ethereum p2p")
				srv.loopWG.Done()
			}()
		}
	}

	srv.loopWG.Add(1)
	go srv.listenLoop()
	return nil
}
```

### setupDiscovery

p2p/server.go

```go
func (srv *Server) setupDiscovery() error {
	srv.discmix = enode.NewFairMix(discmixTimeout)

	// Add protocol-specific discovery sources.
	added := make(map[string]bool)
	for _, proto := range srv.Protocols {
		if proto.DialCandidates != nil && !added[proto.Name] {
			srv.discmix.AddSource(proto.DialCandidates)
			added[proto.Name] = true
		}
	}

	// Don't listen on UDP endpoint if DHT is disabled.
	if srv.NoDiscovery && !srv.DiscoveryV5 {
		return nil
	}

	addr, err := net.ResolveUDPAddr("udp", srv.ListenAddr)
	if err != nil {
		return err
	}
	conn, err := net.ListenUDP("udp", addr)
	if err != nil {
		return err
	}
	realaddr := conn.LocalAddr().(*net.UDPAddr)
	srv.log.Debug("UDP listener up", "addr", realaddr)
	if srv.NAT != nil {
		if !realaddr.IP.IsLoopback() {
			srv.loopWG.Add(1)
			go func() {
				nat.Map(srv.NAT, srv.quit, "udp", realaddr.Port, realaddr.Port, "ethereum discovery")
				srv.loopWG.Done()
			}()
		}
	}
	srv.localnode.SetFallbackUDP(realaddr.Port)

	// Discovery V4
	var unhandled chan discover.ReadPacket
	var sconn *sharedUDPConn
	if !srv.NoDiscovery {
		if srv.DiscoveryV5 {
			unhandled = make(chan discover.ReadPacket, 100)
			sconn = &sharedUDPConn{conn, unhandled}
		}
		cfg := discover.Config{
			PrivateKey:  srv.PrivateKey,
			NetRestrict: srv.NetRestrict,
			Bootnodes:   srv.BootstrapNodes,
			Unhandled:   unhandled,
			Log:         srv.log,
		}
		ntab, err := discover.ListenV4(conn, srv.localnode, cfg)
		if err != nil {
			return err
		}
		srv.ntab = ntab
		srv.discmix.AddSource(ntab.RandomNodes())
	}

	// Discovery V5
	if srv.DiscoveryV5 {
		cfg := discover.Config{
			PrivateKey:  srv.PrivateKey,
			NetRestrict: srv.NetRestrict,
			Bootnodes:   srv.BootstrapNodesV5,
			Log:         srv.log,
		}
		var err error
		if sconn != nil {
			srv.DiscV5, err = discover.ListenV5(sconn, srv.localnode, cfg)
		} else {
			srv.DiscV5, err = discover.ListenV5(conn, srv.localnode, cfg)
		}
		if err != nil {
			return err
		}
	}
	return nil
}
```

Discovery协议：有v5和v4,代码可以看出，支持可配置的协议实现，其代码实现库是devp2p

devp2p代码仓库：https://github.com/ethereum/devp2p

devp2p目前在以太坊１.0中采用，在大部分区块链项目中，比如eth2.0，filecoin，swarm等项目都使用了libp2p，devp2p可以不做重点分析。

libp2p代码仓库：https://github.com/libp2p/go-libp2p

### setupDialScheduler

p2p/server.go

```go
func (srv *Server) setupDialScheduler() {
	config := dialConfig{
		self:           srv.localnode.ID(),
		maxDialPeers:   srv.maxDialedConns(),
		maxActiveDials: srv.MaxPendingPeers,
		log:            srv.Logger,
		netRestrict:    srv.NetRestrict,
		dialer:         srv.Dialer,
		clock:          srv.clock,
	}
	if srv.ntab != nil {
		config.resolver = srv.ntab
	}
	if config.dialer == nil {
		config.dialer = tcpDialer{&net.Dialer{Timeout: defaultDialTimeout}}
	}
	srv.dialsched = newDialScheduler(config, srv.discmix, srv.SetupConn)
	for _, n := range srv.StaticNodes {
		srv.dialsched.addStatic(n)
	}
}
```

p2p/dial.go

```go
func newDialScheduler(config dialConfig, it enode.Iterator, setupFunc dialSetupFunc) *dialScheduler {
	d := &dialScheduler{
		dialConfig:  config.withDefaults(),
		setupFunc:   setupFunc,
		dialing:     make(map[enode.ID]*dialTask),
		static:      make(map[enode.ID]*dialTask),
		peers:       make(map[enode.ID]connFlag),
		doneCh:      make(chan *dialTask),
		nodesIn:     make(chan *enode.Node),
		addStaticCh: make(chan *enode.Node),
		remStaticCh: make(chan *enode.Node),
		addPeerCh:   make(chan *conn),
		remPeerCh:   make(chan *conn),
	}
	d.lastStatsLog = d.clock.Now()
	d.ctx, d.cancel = context.WithCancel(context.Background())
	d.wg.Add(2)
	go d.readNodes(it)
	go d.loop(it)
	return d
}
```

p2p/dial.go

```go
func (d *dialScheduler) loop(it enode.Iterator) {
	var (
		nodesCh    chan *enode.Node
		historyExp = make(chan struct{}, 1)
	)

loop:
	for {
		// Launch new dials if slots are available.
		slots := d.freeDialSlots()
		slots -= d.startStaticDials(slots)
		if slots > 0 {
			nodesCh = d.nodesIn
		} else {
			nodesCh = nil
		}
		d.rearmHistoryTimer(historyExp)
		d.logStats()

		select {
		case node := <-nodesCh:
			if err := d.checkDial(node); err != nil {
				d.log.Trace("Discarding dial candidate", "id", node.ID(), "ip", node.IP(), "reason", err)
			} else {
				d.startDial(newDialTask(node, dynDialedConn))
			}

		case task := <-d.doneCh:
			id := task.dest.ID()
			delete(d.dialing, id)
			d.updateStaticPool(id)
			d.doneSinceLastLog++

		case c := <-d.addPeerCh:
			if c.is(dynDialedConn) || c.is(staticDialedConn) {
				d.dialPeers++
			}
			id := c.node.ID()
			d.peers[id] = c.flags
			// Remove from static pool because the node is now connected.
			task := d.static[id]
			if task != nil && task.staticPoolIndex >= 0 {
				d.removeFromStaticPool(task.staticPoolIndex)
			}
			// TODO: cancel dials to connected peers

		case c := <-d.remPeerCh:
			if c.is(dynDialedConn) || c.is(staticDialedConn) {
				d.dialPeers--
			}
			delete(d.peers, c.node.ID())
			d.updateStaticPool(c.node.ID())

		case node := <-d.addStaticCh:
			id := node.ID()
			_, exists := d.static[id]
			d.log.Trace("Adding static node", "id", id, "ip", node.IP(), "added", !exists)
			if exists {
				continue loop
			}
			task := newDialTask(node, staticDialedConn)
			d.static[id] = task
			if d.checkDial(node) == nil {
				d.addToStaticPool(task)
			}

		case node := <-d.remStaticCh:
			id := node.ID()
			task := d.static[id]
			d.log.Trace("Removing static node", "id", id, "ok", task != nil)
			if task != nil {
				delete(d.static, id)
				if task.staticPoolIndex >= 0 {
					d.removeFromStaticPool(task.staticPoolIndex)
				}
			}

		case <-historyExp:
			d.expireHistory()

		case <-d.ctx.Done():
			it.Close()
			break loop
		}
	}

	d.stopHistoryTimer(historyExp)
	for range d.dialing {
		<-d.doneCh
	}
	d.wg.Done()
}
```

在 dial.go　的 loop　方法里面，会启动轮循器，并且使用select机制，处理节点连接。

接下来，同步区块，打包挖矿的逻辑以后再补充。


参考：

[以太坊Go-ethereum源码分析之启动流程](https://blog.csdn.net/vohyeah/article/details/84138980)

[geth启动流程分析](https://gitee.com/ywbrj042/go-ethereum-code-analysis/blob/master/geth%E5%90%AF%E5%8A%A8%E6%B5%81%E7%A8%8B%E5%88%86%E6%9E%90.md)