# Filecoin 源码分析：lotus 启动流程

- [Filecoin 源码分析：lotus 启动流程](#filecoin-源码分析lotus-启动流程)
	- [准备工作](#准备工作)
		- [环境](#环境)
		- [源码目录](#源码目录)
		- [设计思想](#设计思想)
	- [cmd模块](#cmd模块)
		- [入口](#入口)
		- [BackupCmd](#backupcmd)
		- [DaemonCmd](#daemoncmd)
	- [node模块](#node模块)
		- [创建 node](#创建-node)
		- [节点执行的方法](#节点执行的方法)
			- [node.Online](#nodeonline)
			- [node.Repo](#noderepo)
	- [chain模块](#chain模块)
		- [NewSyncer](#newsyncer)
		- [Start](#start)
		- [Sync](#sync)

## 准备工作

### 环境

分析源码：lotus，filecoin的go语言实现，版本号如下

```bash

Daemon:  1.11.0-dev+mainnet+git.715176698.dirty+api1.3.0
Local: lotus version 1.11.0-dev+mainnet+git.715176698.dirty

```

### 源码目录

在前面的笔记中有解读：https://gitee.com/xingzjx/blockchain/blob/master/filecoin/filecoin_source_guide.md

### 设计思想

在阅读源码之前，建议先部署编译filecoin，以及多看Filecoin白皮书，熟悉官方开发文档。这样有利于理解源码设计。

## cmd模块

cmd 目录是应用程序入口，里面包含多个子模块，比如lotus，lotus-seal-worker,lotus-storage-miner。本篇重点分析 *lotus* 的实现流程。

### 入口

在执行命令**lotus daemon**的时候，会启动**cmd/lotus/main.go**，该方法核心逻辑

- 定义Commond，分别是DaemonCmd和BackupCmd
- 启动分布式追踪，jaeger
- RunApp:最后调用urfave库v2，启动命令行应用

**cmd/lotus/main.go**

```go
func main() {
	api.RunningNodeType = api.NodeFull

	lotuslog.SetupLogLevels()

	local := []*cli.Command{
		DaemonCmd,
		backupCmd,
	}
	if AdvanceBlockCmd != nil {
		local = append(local, AdvanceBlockCmd)
	}

	jaeger := tracing.SetupJaegerTracing("lotus")
	defer func() {
		if jaeger != nil {
			jaeger.Flush()
		}
	}()

	for _, cmd := range local {
		cmd := cmd
		originBefore := cmd.Before
		cmd.Before = func(cctx *cli.Context) error {
			trace.UnregisterExporter(jaeger)
			jaeger = tracing.SetupJaegerTracing("lotus/" + cmd.Name)

			if originBefore != nil {
				return originBefore(cctx)
			}
			return nil
		}
	}
	ctx, span := trace.StartSpan(context.Background(), "/cli")
	defer span.End()

	interactiveDef := isatty.IsTerminal(os.Stdout.Fd()) || isatty.IsCygwinTerminal(os.Stdout.Fd())

	app := &cli.App{
		Name:                 "lotus",
		Usage:                "Filecoin decentralized storage network client",
		Version:              build.UserVersion(),
		EnableBashCompletion: true,
		Flags: []cli.Flag{
			&cli.StringFlag{
				Name:    "repo",
				EnvVars: []string{"LOTUS_PATH"},
				Hidden:  true,
				Value:   "~/.lotus", // TODO: Consider XDG_DATA_HOME
			},
			&cli.BoolFlag{
				Name:  "interactive",
				Usage: "setting to false will disable interactive functionality of commands",
				Value: interactiveDef,
			},
			&cli.BoolFlag{
				Name:  "force-send",
				Usage: "if true, will ignore pre-send checks",
			},
		},

		Commands: append(local, lcli.Commands...),
	}

	app.Setup()
	app.Metadata["traceContext"] = ctx
	app.Metadata["repoType"] = repo.FullNode

	lcli.RunApp(app)
}
```
### BackupCmd 

启动backup服务，执行数据备份相关逻辑。

**cmd/lotus/backup.go**

```go
var backupCmd = lcli.BackupCmd("repo", repo.FullNode, func(cctx *cli.Context) (lcli.BackupAPI, jsonrpc.ClientCloser, error) {
	return lcli.GetFullNodeAPI(cctx)
})
```

**cli/backup.go**

```go
func BackupCmd(repoFlag string, rt repo.RepoType, getApi BackupApiFn) *cli.Command {
	var offlineBackup = func(cctx *cli.Context) error {
		logging.SetLogLevel("badger", "ERROR") // nolint:errcheck

		repoPath := cctx.String(repoFlag)
		r, err := repo.NewFS(repoPath)
		if err != nil {
			return err
		}

		ok, err := r.Exists()
		if err != nil {
			return err
		}
		if !ok {
			return xerrors.Errorf("repo at '%s' is not initialized", cctx.String(repoFlag))
		}

		lr, err := r.LockRO(rt)
		if err != nil {
			return xerrors.Errorf("locking repo: %w", err)
		}
		defer lr.Close() // nolint:errcheck

		mds, err := lr.Datastore(context.TODO(), "/metadata")
		if err != nil {
			return xerrors.Errorf("getting metadata datastore: %w", err)
		}

		bds, err := backupds.Wrap(mds, backupds.NoLogdir)
		if err != nil {
			return err
		}

		fpath, err := homedir.Expand(cctx.Args().First())
		if err != nil {
			return xerrors.Errorf("expanding file path: %w", err)
		}

		out, err := os.OpenFile(fpath, os.O_CREATE|os.O_WRONLY, 0644)
		if err != nil {
			return xerrors.Errorf("opening backup file %s: %w", fpath, err)
		}

		if err := bds.Backup(out); err != nil {
			if cerr := out.Close(); cerr != nil {
				log.Errorw("error closing backup file while handling backup error", "closeErr", cerr, "backupErr", err)
			}
			return xerrors.Errorf("backup error: %w", err)
		}

		if err := out.Close(); err != nil {
			return xerrors.Errorf("closing backup file: %w", err)
		}

		return nil
	}

	var onlineBackup = func(cctx *cli.Context) error {
		api, closer, err := getApi(cctx)
		if err != nil {
			return xerrors.Errorf("getting api: %w (if the node isn't running you can use the --offline flag)", err)
		}
		defer closer()

		err = api.CreateBackup(ReqContext(cctx), cctx.Args().First())
		if err != nil {
			return err
		}

		fmt.Println("Success")

		return nil
	}

	return &cli.Command{
		Name:  "backup",
		Usage: "Create node metadata backup",
		Description: `The backup command writes a copy of node metadata under the specified path

Online backups:
For security reasons, the daemon must be have LOTUS_BACKUP_BASE_PATH env var set
to a path where backup files are supposed to be saved, and the path specified in
this command must be within this base path`,
		Flags: []cli.Flag{
			&cli.BoolFlag{
				Name:  "offline",
				Usage: "create backup without the node running",
			},
		},
		ArgsUsage: "[backup file path]",
		Action: func(cctx *cli.Context) error {
			if cctx.Args().Len() != 1 {
				return xerrors.Errorf("expected 1 argument")
			}

			if cctx.Bool("offline") {
				return offlineBackup(cctx)
			}

			return onlineBackup(cctx)
		},
	}
}
```

**lib/backups/datastore.go**

```go
func Wrap(child datastore.Batching, logdir string) (*Datastore, error) {
	ds := &Datastore{
		child: child,
	}

	if logdir != NoLogdir {
		ds.closing, ds.closed = make(chan struct{}), make(chan struct{})
		ds.log = make(chan Entry)

		if err := ds.startLog(logdir); err != nil {
			return nil, err
		}
	}

	return ds, nil
}
```
**lib/backups/log.go**

```go
func (d *Datastore) startLog(logdir string) error {
	if err := os.MkdirAll(logdir, 0755); err != nil && !os.IsExist(err) {
		return xerrors.Errorf("mkdir logdir ('%s'): %w", logdir, err)
	}

	files, err := ioutil.ReadDir(logdir)
	if err != nil {
		return xerrors.Errorf("read logdir ('%s'): %w", logdir, err)
	}

	var latest string
	var latestTs int64

	for _, file := range files {
		fn := file.Name()
		if !strings.HasSuffix(fn, ".log.cbor") {
			log.Warn("logfile with wrong file extension", fn)
			continue
		}
		sec, err := strconv.ParseInt(fn[:len(".log.cbor")], 10, 64)
		if err != nil {
			return xerrors.Errorf("parsing logfile as a number: %w", err)
		}

		if sec > latestTs {
			latestTs = sec
			latest = file.Name()
		}
	}

	var l *logfile
	if latest == "" {
		l, latest, err = d.createLog(logdir)
		if err != nil {
			return xerrors.Errorf("creating log: %w", err)
		}
	} else {
		l, latest, err = d.openLog(filepath.Join(logdir, latest))
		if err != nil {
			return xerrors.Errorf("opening log: %w", err)
		}
	}

	if err := l.writeLogHead(latest, d.child); err != nil {
		return xerrors.Errorf("writing new log head: %w", err)
	}

	go d.runLog(l)

	return nil
}
```

### DaemonCmd

DaemonCmd 是lotus启动后的核心逻辑，包括解析启动参数，p2p连接，同步数据等都在这里定义的逻辑。

- view.Register : 注册所有的 metric views
- lcli.GetGatewayAPI: 如果lotus 以 lite 模式启动，会提供 gateway 的 rpc 调用
- node.New ： 创建 node ， lotus 启动的核心逻辑是在 node 模块实现的
- node.ServeRPC ：启动json-rpc服务
- node.MonitorShutdown ：监控节点关闭状态

**cmd/lotus/daemon.go**

```go
var DaemonCmd = &cli.Command{
	Name:  "daemon",
	Usage: "Start a lotus daemon process",
	Flags: []cli.Flag{
		&cli.StringFlag{
			Name:  "api",
			Value: "1234",
		},
		&cli.StringFlag{
			Name:   makeGenFlag,
			Value:  "",
			Hidden: true,
		},
		&cli.StringFlag{
			Name:   preTemplateFlag,
			Hidden: true,
		},
		&cli.StringFlag{
			Name:   "import-key",
			Usage:  "on first run, import a default key from a given file",
			Hidden: true,
		},
		&cli.StringFlag{
			Name:  "genesis",
			Usage: "genesis file to use for first node run",
		},
		&cli.BoolFlag{
			Name:  "bootstrap",
			Value: true,
		},
		&cli.StringFlag{
			Name:  "import-chain",
			Usage: "on first run, load chain from given file or url and validate",
		},
		&cli.StringFlag{
			Name:  "import-snapshot",
			Usage: "import chain state from a given chain export file or url",
		},
		&cli.BoolFlag{
			Name:  "halt-after-import",
			Usage: "halt the process after importing chain from file",
		},
		&cli.BoolFlag{
			Name:   "lite",
			Usage:  "start lotus in lite mode",
			Hidden: true,
		},
		&cli.StringFlag{
			Name:  "pprof",
			Usage: "specify name of file for writing cpu profile to",
		},
		&cli.StringFlag{
			Name:  "profile",
			Usage: "specify type of node",
		},
		&cli.BoolFlag{
			Name:  "manage-fdlimit",
			Usage: "manage open file limit",
			Value: true,
		},
		&cli.StringFlag{
			Name:  "config",
			Usage: "specify path of config file to use",
		},
		// FIXME: This is not the correct place to put this configuration
		//  option. Ideally it would be part of `config.toml` but at the
		//  moment that only applies to the node configuration and not outside
		//  components like the RPC server.
		&cli.IntFlag{
			Name:  "api-max-req-size",
			Usage: "maximum API request size accepted by the JSON RPC server",
		},
		&cli.PathFlag{
			Name:  "restore",
			Usage: "restore from backup file",
		},
		&cli.PathFlag{
			Name:  "restore-config",
			Usage: "config file to use when restoring from backup",
		},
	},
	Action: func(cctx *cli.Context) error {
		isLite := cctx.Bool("lite")

		err := runmetrics.Enable(runmetrics.RunMetricOptions{
			EnableCPU:    true,
			EnableMemory: true,
		})
		if err != nil {
			return xerrors.Errorf("enabling runtime metrics: %w", err)
		}

		if cctx.Bool("manage-fdlimit") {
			if _, _, err := ulimit.ManageFdLimit(); err != nil {
				log.Errorf("setting file descriptor limit: %s", err)
			}
		}

		if prof := cctx.String("pprof"); prof != "" {
			profile, err := os.Create(prof)
			if err != nil {
				return err
			}

			if err := pprof.StartCPUProfile(profile); err != nil {
				return err
			}
			defer pprof.StopCPUProfile()
		}

		var isBootstrapper dtypes.Bootstrapper
		switch profile := cctx.String("profile"); profile {
		case "bootstrapper":
			isBootstrapper = true
		case "":
			// do nothing
		default:
			return fmt.Errorf("unrecognized profile type: %q", profile)
		}

		ctx, _ := tag.New(context.Background(),
			tag.Insert(metrics.Version, build.BuildVersion),
			tag.Insert(metrics.Commit, build.CurrentCommit),
			tag.Insert(metrics.NodeType, "chain"),
		)
		// Register all metric views
		if err = view.Register(
			metrics.ChainNodeViews...,
		); err != nil {
			log.Fatalf("Cannot register the view: %v", err)
		}
		// Set the metric to one so it is published to the exporter
		stats.Record(ctx, metrics.LotusInfo.M(1))

		{
			dir, err := homedir.Expand(cctx.String("repo"))
			if err != nil {
				log.Warnw("could not expand repo location", "error", err)
			} else {
				log.Infof("lotus repo: %s", dir)
			}
		}

		r, err := repo.NewFS(cctx.String("repo"))
		if err != nil {
			return xerrors.Errorf("opening fs repo: %w", err)
		}

		if cctx.String("config") != "" {
			r.SetConfigPath(cctx.String("config"))
		}

		err = r.Init(repo.FullNode)
		if err != nil && err != repo.ErrRepoExists {
			return xerrors.Errorf("repo init error: %w", err)
		}
		freshRepo := err != repo.ErrRepoExists

		if !isLite {
			if err := paramfetch.GetParams(lcli.ReqContext(cctx), build.ParametersJSON(), build.SrsJSON(), 0); err != nil {
				return xerrors.Errorf("fetching proof parameters: %w", err)
			}
		}

		var genBytes []byte
		if cctx.String("genesis") != "" {
			genBytes, err = ioutil.ReadFile(cctx.String("genesis"))
			if err != nil {
				return xerrors.Errorf("reading genesis: %w", err)
			}
		} else {
			genBytes = build.MaybeGenesis()
		}

		if cctx.IsSet("restore") {
			if !freshRepo {
				return xerrors.Errorf("restoring from backup is only possible with a fresh repo!")
			}
			if err := restore(cctx, r); err != nil {
				return xerrors.Errorf("restoring from backup: %w", err)
			}
		}

		chainfile := cctx.String("import-chain")
		snapshot := cctx.String("import-snapshot")
		if chainfile != "" || snapshot != "" {
			if chainfile != "" && snapshot != "" {
				return fmt.Errorf("cannot specify both 'import-snapshot' and 'import-chain'")
			}
			var issnapshot bool
			if chainfile == "" {
				chainfile = snapshot
				issnapshot = true
			}

			if err := ImportChain(ctx, r, chainfile, issnapshot); err != nil {
				return err
			}
			if cctx.Bool("halt-after-import") {
				fmt.Println("Chain import complete, halting as requested...")
				return nil
			}
		}

		genesis := node.Options()
		if len(genBytes) > 0 {
			genesis = node.Override(new(modules.Genesis), modules.LoadGenesis(genBytes))
		}
		if cctx.String(makeGenFlag) != "" {
			if cctx.String(preTemplateFlag) == "" {
				return xerrors.Errorf("must also pass file with genesis template to `--%s`", preTemplateFlag)
			}
			genesis = node.Override(new(modules.Genesis), testing.MakeGenesis(cctx.String(makeGenFlag), cctx.String(preTemplateFlag)))
		}

		shutdownChan := make(chan struct{})

		// If the daemon is started in "lite mode", provide a  Gateway
		// for RPC calls
		liteModeDeps := node.Options()
		if isLite {
			gapi, closer, err := lcli.GetGatewayAPI(cctx)
			if err != nil {
				return err
			}

			defer closer()
			liteModeDeps = node.Override(new(api.Gateway), gapi)
		}

		// some libraries like ipfs/go-ds-measure and ipfs/go-ipfs-blockstore
		// use ipfs/go-metrics-interface. This injects a Prometheus exporter
		// for those. Metrics are exported to the default registry.
		if err := metricsprom.Inject(); err != nil {
			log.Warnf("unable to inject prometheus ipfs/go-metrics exporter; some metrics will be unavailable; err: %s", err)
		}

		var api api.FullNode
		stop, err := node.New(ctx,
			node.FullAPI(&api, node.Lite(isLite)),

			node.Online(),
			node.Repo(r),

			node.Override(new(dtypes.Bootstrapper), isBootstrapper),
			node.Override(new(dtypes.ShutdownChan), shutdownChan),

			genesis,
			liteModeDeps,

			node.ApplyIf(func(s *node.Settings) bool { return cctx.IsSet("api") },
				node.Override(node.SetApiEndpointKey, func(lr repo.LockedRepo) error {
					apima, err := multiaddr.NewMultiaddr("/ip4/127.0.0.1/tcp/" +
						cctx.String("api"))
					if err != nil {
						return err
					}
					return lr.SetAPIEndpoint(apima)
				})),
			node.ApplyIf(func(s *node.Settings) bool { return !cctx.Bool("bootstrap") },
				node.Unset(node.RunPeerMgrKey),
				node.Unset(new(*peermgr.PeerMgr)),
			),
		)
		if err != nil {
			return xerrors.Errorf("initializing node: %w", err)
		}

		if cctx.String("import-key") != "" {
			if err := importKey(ctx, api, cctx.String("import-key")); err != nil {
				log.Errorf("importing key failed: %+v", err)
			}
		}

		endpoint, err := r.APIEndpoint()
		if err != nil {
			return xerrors.Errorf("getting api endpoint: %w", err)
		}

		//
		// Instantiate JSON-RPC endpoint.
		// ----

		// Populate JSON-RPC options.
		serverOptions := make([]jsonrpc.ServerOption, 0)
		if maxRequestSize := cctx.Int("api-max-req-size"); maxRequestSize != 0 {
			serverOptions = append(serverOptions, jsonrpc.WithMaxRequestSize(int64(maxRequestSize)))
		}

		// Instantiate the full node handler.
		h, err := node.FullNodeHandler(api, true, serverOptions...)
		if err != nil {
			return fmt.Errorf("failed to instantiate rpc handler: %s", err)
		}

		// Serve the RPC.
		rpcStopper, err := node.ServeRPC(h, "lotus-daemon", endpoint)
		if err != nil {
			return fmt.Errorf("failed to start json-rpc endpoint: %s", err)
		}

		// Monitor for shutdown.
		finishCh := node.MonitorShutdown(shutdownChan,
			node.ShutdownHandler{Component: "rpc server", StopFunc: rpcStopper},
			node.ShutdownHandler{Component: "node", StopFunc: stop},
		)
		<-finishCh // fires when shutdown is complete.

		// TODO: properly parse api endpoint (or make it a URL)
		return nil
	},
	Subcommands: []*cli.Command{
		daemonStopCmd,
	},
}
```

接下来，分析 node 模块


## node模块

node 模块包括了区块链节点相关的业务逻辑

### 创建 node

在创建节点中，会传入需要执行的命令

**node/builder.go**

```go
func New(ctx context.Context, opts ...Option) (StopFunc, error) {
	settings := Settings{
		modules: map[interface{}]fx.Option{},
		invokes: make([]fx.Option, _nInvokes),
	}

	// apply module options in the right order
	if err := Options(Options(defaults()...), Options(opts...))(&settings); err != nil {
		return nil, xerrors.Errorf("applying node options failed: %w", err)
	}

	// gather constructors for fx.Options
	ctors := make([]fx.Option, 0, len(settings.modules))
	for _, opt := range settings.modules {
		ctors = append(ctors, opt)
	}

	// fill holes in invokes for use in fx.Options
	for i, opt := range settings.invokes {
		if opt == nil {
			settings.invokes[i] = fx.Options()
		}
	}

	app := fx.New(
		fx.Options(ctors...),
		fx.Options(settings.invokes...),

		fx.NopLogger,
	)

	// TODO: we probably should have a 'firewall' for Closing signal
	//  on this context, and implement closing logic through lifecycles
	//  correctly
	if err := app.Start(ctx); err != nil {
		// comment fx.NopLogger few lines above for easier debugging
		return nil, xerrors.Errorf("starting node: %w", err)
	}

	return app.Stop, nil
}
```

### 节点执行的方法

在创建节点的时候，定义了许多命令，这个定义是在 cmd/lotus/daemon.go 里面定义的，回顾这块代码：

```go
	// 省略 ...
	stop, err := node.New(ctx,
		node.FullAPI(&api, node.Lite(isLite)),

		node.Online(),
		node.Repo(r),

		node.Override(new(dtypes.Bootstrapper), isBootstrapper),
		node.Override(new(dtypes.ShutdownChan), shutdownChan),

		genesis,
		liteModeDeps,

		node.ApplyIf(func(s *node.Settings) bool { return cctx.IsSet("api") },
			node.Override(node.SetApiEndpointKey, func(lr repo.LockedRepo) error {
				apima, err := multiaddr.NewMultiaddr("/ip4/127.0.0.1/tcp/" +
					cctx.String("api"))
				if err != nil {
					return err
				}
				return lr.SetAPIEndpoint(apima)
			})),
		node.ApplyIf(func(s *node.Settings) bool { return !cctx.Bool("bootstrap") },
			node.Unset(node.RunPeerMgrKey),
			node.Unset(new(*peermgr.PeerMgr)),
		),
	)
	// 省略 ...

```

#### node.Online 

启动 p2p 节点

- LibP2P : 定义 libp2p 服务，libp2p 的实现在 node/modules/lp2p 目录
- ChainNode ： 提供访问 filecoin 节点的功能
- MinerNode

**node/builder.go**

```go
func Online() Option {

	return Options(
		// make sure that online is applied before Config.
		// This is important because Config overrides some of Online units
		func(s *Settings) error { s.Online = true; return nil },
		ApplyIf(func(s *Settings) bool { return s.Config },
			Error(errors.New("the Online option must be set before Config option")),
		),

		LibP2P,

		ApplyIf(isFullOrLiteNode, ChainNode),
		ApplyIf(isType(repo.StorageMiner), MinerNode),
	)
}
```
- LibP2p

**node/builder.go**

```go
var LibP2P = Options(
	// Host config
	Override(new(dtypes.Bootstrapper), dtypes.Bootstrapper(false)),

	// Host dependencies
	Override(new(peerstore.Peerstore), pstoremem.NewPeerstore),
	Override(PstoreAddSelfKeysKey, lp2p.PstoreAddSelfKeys),
	Override(StartListeningKey, lp2p.StartListening(config.DefaultFullNode().Libp2p.ListenAddresses)),

	// Host settings
	Override(DefaultTransportsKey, lp2p.DefaultTransports),
	Override(AddrsFactoryKey, lp2p.AddrsFactory(nil, nil)),
	Override(SmuxTransportKey, lp2p.SmuxTransport(true)),
	Override(RelayKey, lp2p.NoRelay()),
	Override(SecurityKey, lp2p.Security(true, false)),

	// Host
	Override(new(lp2p.RawHost), lp2p.Host),
	Override(new(host.Host), lp2p.RoutedHost),
	Override(new(lp2p.BaseIpfsRouting), lp2p.DHTRouting(dht.ModeAuto)),

	Override(DiscoveryHandlerKey, lp2p.DiscoveryHandler),

	// Routing
	Override(new(record.Validator), modules.RecordValidator),
	Override(BaseRoutingKey, lp2p.BaseRouting),
	Override(new(routing.Routing), lp2p.Routing),

	// Services
	Override(NatPortMapKey, lp2p.NatPortMap),
	Override(BandwidthReporterKey, lp2p.BandwidthCounter),
	Override(AutoNATSvcKey, lp2p.AutoNATService),

	// Services (pubsub)
	Override(new(*dtypes.ScoreKeeper), lp2p.ScoreKeeper),
	Override(new(*pubsub.PubSub), lp2p.GossipSub),
	Override(new(*config.Pubsub), func(bs dtypes.Bootstrapper) *config.Pubsub {
		return &config.Pubsub{
			Bootstrapper: bool(bs),
		}
	}),

	// Services (connection management)
	Override(ConnectionManagerKey, lp2p.ConnectionManager(50, 200, 20*time.Second, nil)),
	Override(new(*conngater.BasicConnectionGater), lp2p.ConnGater),
	Override(ConnGaterKey, lp2p.ConnGaterOption),
)
```

- ChainNode：提供了访问 filecoin 网络的服务，比如同步区块， modules.NewSyncer

**node/builder.go**

```go
var ChainNode = Options(
	// Full node or lite node
	// TODO: Fix offline mode

	// Consensus settings
	Override(new(dtypes.DrandSchedule), modules.BuiltinDrandConfig),
	Override(new(stmgr.UpgradeSchedule), stmgr.DefaultUpgradeSchedule()),
	Override(new(dtypes.NetworkName), modules.NetworkName),
	Override(new(modules.Genesis), modules.ErrorGenesis),
	Override(new(dtypes.AfterGenesisSet), modules.SetGenesis),
	Override(SetGenesisKey, modules.DoSetGenesis),
	Override(new(beacon.Schedule), modules.RandomSchedule),

	// Network bootstrap
	Override(new(dtypes.BootstrapPeers), modules.BuiltinBootstrap),
	Override(new(dtypes.DrandBootstrap), modules.DrandBootstrap),

	// Consensus: crypto dependencies
	Override(new(ffiwrapper.Verifier), ffiwrapper.ProofVerifier),

	// Consensus: VM
	Override(new(vm.SyscallBuilder), vm.Syscalls),

	// Consensus: Chain storage/access
	Override(new(*store.ChainStore), modules.ChainStore),
	Override(new(*stmgr.StateManager), modules.StateManager),
	Override(new(dtypes.ChainBitswap), modules.ChainBitswap),
	Override(new(dtypes.ChainBlockService), modules.ChainBlockService), // todo: unused

	// Consensus: Chain sync

	// We don't want the SyncManagerCtor to be used as an fx constructor, but rather as a value.
	// It will be called implicitly by the Syncer constructor.
	Override(new(chain.SyncManagerCtor), func() chain.SyncManagerCtor { return chain.NewSyncManager }),
	Override(new(*chain.Syncer), modules.NewSyncer),
	Override(new(exchange.Client), exchange.NewClient),

	// Chain networking
	Override(new(*hello.Service), hello.NewHelloService),
	Override(new(exchange.Server), exchange.NewServer),
	Override(new(*peermgr.PeerMgr), peermgr.NewPeerMgr),

	// Chain mining API dependencies
	Override(new(*slashfilter.SlashFilter), modules.NewSlashFilter),

	// Service: Message Pool
	Override(new(dtypes.DefaultMaxFeeFunc), modules.NewDefaultMaxFeeFunc),
	Override(new(*messagepool.MessagePool), modules.MessagePool),
	Override(new(*dtypes.MpoolLocker), new(dtypes.MpoolLocker)),

	// Shared graphsync (markets, serving chain)
	Override(new(dtypes.Graphsync), modules.Graphsync(config.DefaultFullNode().Client.SimultaneousTransfers)),

	// Service: Wallet
	Override(new(*messagesigner.MessageSigner), messagesigner.NewMessageSigner),
	Override(new(*wallet.LocalWallet), wallet.NewWallet),
	Override(new(wallet.Default), From(new(*wallet.LocalWallet))),
	Override(new(api.Wallet), From(new(wallet.MultiWallet))),

	// Service: Payment channels
	Override(new(paychmgr.PaychAPI), From(new(modules.PaychAPI))),
	Override(new(*paychmgr.Store), modules.NewPaychStore),
	Override(new(*paychmgr.Manager), modules.NewManager),
	Override(HandlePaymentChannelManagerKey, modules.HandlePaychManager),
	Override(SettlePaymentChannelsKey, settler.SettlePaymentChannels),

	// Markets (common)
	Override(new(*discoveryimpl.Local), modules.NewLocalDiscovery),

	// Markets (retrieval)
	Override(new(discovery.PeerResolver), modules.RetrievalResolver),
	Override(new(retrievalmarket.RetrievalClient), modules.RetrievalClient),
	Override(new(dtypes.ClientDataTransfer), modules.NewClientGraphsyncDataTransfer),

	// Markets (storage)
	Override(new(*market.FundManager), market.NewFundManager),
	Override(new(dtypes.ClientDatastore), modules.NewClientDatastore),
	Override(new(storagemarket.StorageClient), modules.StorageClient),
	Override(new(storagemarket.StorageClientNode), storageadapter.NewClientNodeAdapter),
	Override(HandleMigrateClientFundsKey, modules.HandleMigrateClientFunds),

	Override(new(*full.GasPriceCache), full.NewGasPriceCache),

	// Lite node API
	ApplyIf(isLiteNode,
		Override(new(messagepool.Provider), messagepool.NewProviderLite),
		Override(new(messagesigner.MpoolNonceAPI), From(new(modules.MpoolNonceAPI))),
		Override(new(full.ChainModuleAPI), From(new(api.Gateway))),
		Override(new(full.GasModuleAPI), From(new(api.Gateway))),
		Override(new(full.MpoolModuleAPI), From(new(api.Gateway))),
		Override(new(full.StateModuleAPI), From(new(api.Gateway))),
		Override(new(stmgr.StateManagerAPI), rpcstmgr.NewRPCStateManager),
	),

	// Full node API / service startup
	ApplyIf(isFullNode,
		Override(new(messagepool.Provider), messagepool.NewProvider),
		Override(new(messagesigner.MpoolNonceAPI), From(new(*messagepool.MessagePool))),
		Override(new(full.ChainModuleAPI), From(new(full.ChainModule))),
		Override(new(full.GasModuleAPI), From(new(full.GasModule))),
		Override(new(full.MpoolModuleAPI), From(new(full.MpoolModule))),
		Override(new(full.StateModuleAPI), From(new(full.StateModule))),
		Override(new(stmgr.StateManagerAPI), From(new(*stmgr.StateManager))),

		Override(RunHelloKey, modules.RunHello),
		Override(RunChainExchangeKey, modules.RunChainExchange),
		Override(RunPeerMgrKey, modules.RunPeerMgr),
		Override(HandleIncomingMessagesKey, modules.HandleIncomingMessages),
		Override(HandleIncomingBlocksKey, modules.HandleIncomingBlocks),
	),
)
```

以同步区块服务功能为例，继续分析以下逻辑：

- NewSyncer：调用到chain模块，创建同步器
- syncer.Start： 启动同步器

**node/modules/chain.go**

```go
func NewSyncer(params SyncerParams) (*chain.Syncer, error) {
	var (
		lc     = params.Lifecycle
		ds     = params.MetadataDS
		sm     = params.StateManager
		ex     = params.ChainXchg
		smCtor = params.SyncMgrCtor
		h      = params.Host
		b      = params.Beacon
		v      = params.Verifier
	)
	syncer, err := chain.NewSyncer(ds, sm, ex, smCtor, h.ConnManager(), h.ID(), b, v)
	if err != nil {
		return nil, err
	}

	lc.Append(fx.Hook{
		OnStart: func(_ context.Context) error {
			syncer.Start()
			return nil
		},
		OnStop: func(_ context.Context) error {
			syncer.Stop()
			return nil
		},
	})
	return syncer, nil
}
```

- MinerNode: 提供了挖矿相关的服务实现，比如密封、证明、存储等

**node/builder.go**

```go
var MinerNode = Options(
	// API dependencies
	Override(new(api.Common), From(new(common.CommonAPI))),
	Override(new(sectorstorage.StorageAuth), modules.StorageAuth),

	// Actor config
	Override(new(dtypes.MinerAddress), modules.MinerAddress),
	Override(new(dtypes.MinerID), modules.MinerID),
	Override(new(abi.RegisteredSealProof), modules.SealProofType),
	Override(new(dtypes.NetworkName), modules.StorageNetworkName),

	// Sector storage
	Override(new(*stores.Index), stores.NewIndex),
	Override(new(stores.SectorIndex), From(new(*stores.Index))),
	Override(new(stores.LocalStorage), From(new(repo.LockedRepo))),
	Override(new(*stores.Local), modules.LocalStorage),
	Override(new(*stores.Remote), modules.RemoteStorage),
	Override(new(*sectorstorage.Manager), modules.SectorStorage),
	Override(new(sectorstorage.SectorManager), From(new(*sectorstorage.Manager))),
	Override(new(storiface.WorkerReturn), From(new(sectorstorage.SectorManager))),
	Override(new(sectorstorage.Unsealer), From(new(*sectorstorage.Manager))),

	// Sector storage: Proofs
	Override(new(ffiwrapper.Verifier), ffiwrapper.ProofVerifier),
	Override(new(ffiwrapper.Prover), ffiwrapper.ProofProver),
	Override(new(storage2.Prover), From(new(sectorstorage.SectorManager))),

	// Sealing
	Override(new(sealing.SectorIDCounter), modules.SectorIDCounter),
	Override(GetParamsKey, modules.GetParams),

	// Mining / proving
	Override(new(*slashfilter.SlashFilter), modules.NewSlashFilter),
	Override(new(*storage.Miner), modules.StorageMiner(config.DefaultStorageMiner().Fees)),
	Override(new(*miner.Miner), modules.SetupBlockProducer),
	Override(new(gen.WinningPoStProver), storage.NewWinningPoStProver),

	Override(new(*storage.AddressSelector), modules.AddressSelector(nil)),

	// Markets
	Override(new(dtypes.StagingMultiDstore), modules.StagingMultiDatastore),
	Override(new(dtypes.StagingBlockstore), modules.StagingBlockstore),
	Override(new(dtypes.StagingDAG), modules.StagingDAG),
	Override(new(dtypes.StagingGraphsync), modules.StagingGraphsync),
	Override(new(dtypes.ProviderPieceStore), modules.NewProviderPieceStore),
	Override(new(*sectorblocks.SectorBlocks), sectorblocks.NewSectorBlocks),

	// Markets (retrieval)
	Override(new(sectorstorage.PieceProvider), sectorstorage.NewPieceProvider),
	Override(new(dtypes.RetrievalPricingFunc), modules.RetrievalPricingFunc(config.DealmakingConfig{
		RetrievalPricing: &config.RetrievalPricing{
			Strategy: config.RetrievalPricingDefaultMode,
			Default:  &config.RetrievalPricingDefault{},
		},
	})),
	Override(new(sectorstorage.PieceProvider), sectorstorage.NewPieceProvider),
	Override(new(retrievalmarket.RetrievalProvider), modules.RetrievalProvider),
	Override(new(dtypes.RetrievalDealFilter), modules.RetrievalDealFilter(nil)),

	Override(HandleRetrievalKey, modules.HandleRetrieval),

	// Markets (storage)
	Override(new(dtypes.ProviderDataTransfer), modules.NewProviderDAGServiceDataTransfer),
	Override(new(*storedask.StoredAsk), modules.NewStorageAsk),
	Override(new(dtypes.StorageDealFilter), modules.BasicDealFilter(nil)),
	Override(new(storagemarket.StorageProvider), modules.StorageProvider),
	Override(new(*storageadapter.DealPublisher), storageadapter.NewDealPublisher(nil, storageadapter.PublishMsgConfig{})),
	Override(new(storagemarket.StorageProviderNode), storageadapter.NewProviderNodeAdapter(nil, nil)),
	Override(HandleMigrateProviderFundsKey, modules.HandleMigrateProviderFunds),
	Override(HandleDealsKey, modules.HandleDeals),

	// Config (todo: get a real property system)
	Override(new(dtypes.ConsiderOnlineStorageDealsConfigFunc), modules.NewConsiderOnlineStorageDealsConfigFunc),
	Override(new(dtypes.SetConsiderOnlineStorageDealsConfigFunc), modules.NewSetConsideringOnlineStorageDealsFunc),
	Override(new(dtypes.ConsiderOnlineRetrievalDealsConfigFunc), modules.NewConsiderOnlineRetrievalDealsConfigFunc),
	Override(new(dtypes.SetConsiderOnlineRetrievalDealsConfigFunc), modules.NewSetConsiderOnlineRetrievalDealsConfigFunc),
	Override(new(dtypes.StorageDealPieceCidBlocklistConfigFunc), modules.NewStorageDealPieceCidBlocklistConfigFunc),
	Override(new(dtypes.SetStorageDealPieceCidBlocklistConfigFunc), modules.NewSetStorageDealPieceCidBlocklistConfigFunc),
	Override(new(dtypes.ConsiderOfflineStorageDealsConfigFunc), modules.NewConsiderOfflineStorageDealsConfigFunc),
	Override(new(dtypes.SetConsiderOfflineStorageDealsConfigFunc), modules.NewSetConsideringOfflineStorageDealsFunc),
	Override(new(dtypes.ConsiderOfflineRetrievalDealsConfigFunc), modules.NewConsiderOfflineRetrievalDealsConfigFunc),
	Override(new(dtypes.SetConsiderOfflineRetrievalDealsConfigFunc), modules.NewSetConsiderOfflineRetrievalDealsConfigFunc),
	Override(new(dtypes.ConsiderVerifiedStorageDealsConfigFunc), modules.NewConsiderVerifiedStorageDealsConfigFunc),
	Override(new(dtypes.SetConsiderVerifiedStorageDealsConfigFunc), modules.NewSetConsideringVerifiedStorageDealsFunc),
	Override(new(dtypes.ConsiderUnverifiedStorageDealsConfigFunc), modules.NewConsiderUnverifiedStorageDealsConfigFunc),
	Override(new(dtypes.SetConsiderUnverifiedStorageDealsConfigFunc), modules.NewSetConsideringUnverifiedStorageDealsFunc),
	Override(new(dtypes.SetSealingConfigFunc), modules.NewSetSealConfigFunc),
	Override(new(dtypes.GetSealingConfigFunc), modules.NewGetSealConfigFunc),
	Override(new(dtypes.SetExpectedSealDurationFunc), modules.NewSetExpectedSealDurationFunc),
	Override(new(dtypes.GetExpectedSealDurationFunc), modules.NewGetExpectedSealDurationFunc),
)
```

#### node.Repo


**node/builder.go**

```go
func Repo(r repo.Repo) Option {
	return func(settings *Settings) error {
		lr, err := r.Lock(settings.nodeType)
		if err != nil {
			return err
		}
		c, err := lr.Config()
		if err != nil {
			return err
		}

		var cfg *config.Chainstore
		switch settings.nodeType {
		case repo.FullNode:
			cfgp, ok := c.(*config.FullNode)
			if !ok {
				return xerrors.Errorf("invalid config from repo, got: %T", c)
			}
			cfg = &cfgp.Chainstore
		default:
			cfg = &config.Chainstore{}
		}

		return Options(
			Override(new(repo.LockedRepo), modules.LockedRepo(lr)), // module handles closing

			Override(new(dtypes.UniversalBlockstore), modules.UniversalBlockstore),

			If(cfg.EnableSplitstore,
				If(cfg.Splitstore.HotStoreType == "badger",
					Override(new(dtypes.HotBlockstore), modules.BadgerHotBlockstore)),
				Override(new(dtypes.SplitBlockstore), modules.SplitBlockstore(cfg)),
				Override(new(dtypes.BasicChainBlockstore), modules.ChainSplitBlockstore),
				Override(new(dtypes.BasicStateBlockstore), modules.StateSplitBlockstore),
				Override(new(dtypes.BaseBlockstore), From(new(dtypes.SplitBlockstore))),
				Override(new(dtypes.ExposedBlockstore), From(new(dtypes.SplitBlockstore))),
			),
			If(!cfg.EnableSplitstore,
				Override(new(dtypes.BasicChainBlockstore), modules.ChainFlatBlockstore),
				Override(new(dtypes.BasicStateBlockstore), modules.StateFlatBlockstore),
				Override(new(dtypes.BaseBlockstore), From(new(dtypes.UniversalBlockstore))),
				Override(new(dtypes.ExposedBlockstore), From(new(dtypes.UniversalBlockstore))),
			),

			Override(new(dtypes.ChainBlockstore), From(new(dtypes.BasicChainBlockstore))),
			Override(new(dtypes.StateBlockstore), From(new(dtypes.BasicStateBlockstore))),

			If(os.Getenv("LOTUS_ENABLE_CHAINSTORE_FALLBACK") == "1",
				Override(new(dtypes.ChainBlockstore), modules.FallbackChainBlockstore),
				Override(new(dtypes.StateBlockstore), modules.FallbackStateBlockstore),
				Override(SetupFallbackBlockstoresKey, modules.InitFallbackBlockstores),
			),

			Override(new(dtypes.ClientImportMgr), modules.ClientImportMgr),
			Override(new(dtypes.ClientMultiDstore), modules.ClientMultiDatastore),

			Override(new(dtypes.ClientBlockstore), modules.ClientBlockstore),
			Override(new(dtypes.ClientRetrievalStoreManager), modules.ClientRetrievalStoreManager),
			Override(new(ci.PrivKey), lp2p.PrivKey),
			Override(new(ci.PubKey), ci.PrivKey.GetPublic),
			Override(new(peer.ID), peer.IDFromPublicKey),

			Override(new(types.KeyStore), modules.KeyStore),

			Override(new(*dtypes.APIAlg), modules.APISecret),

			ApplyIf(isType(repo.FullNode), ConfigFullNode(c)),
			ApplyIf(isType(repo.StorageMiner), ConfigStorageMiner(c)),
		)(settings)
	}
}
```

ConfigFullNode : 如果是全节点模式，配置全节点

ConfigStorageMiner： 对于矿工模式，配置矿工

## chain模块

chain 模块包括区块同步以及其它的区块相关实现逻辑，通过node模块调用到。

### NewSyncer

**chain/sync.go**

```go
func NewSyncer(ds dtypes.MetadataDS, sm *stmgr.StateManager, exchange exchange.Client, syncMgrCtor SyncManagerCtor, connmgr connmgr.ConnManager, self peer.ID, beacon beacon.Schedule, verifier ffiwrapper.Verifier) (*Syncer, error) {
	gen, err := sm.ChainStore().GetGenesis()
	if err != nil {
		return nil, xerrors.Errorf("getting genesis block: %w", err)
	}

	gent, err := types.NewTipSet([]*types.BlockHeader{gen})
	if err != nil {
		return nil, err
	}

	s := &Syncer{
		ds:             ds,
		beacon:         beacon,
		bad:            NewBadBlockCache(),
		Genesis:        gent,
		Exchange:       exchange,
		store:          sm.ChainStore(),
		sm:             sm,
		self:           self,
		receiptTracker: newBlockReceiptTracker(),
		connmgr:        connmgr,
		verifier:       verifier,

		incoming: pubsub.New(50),
	}

	if build.InsecurePoStValidation {
		log.Warn("*********************************************************************************************")
		log.Warn(" [INSECURE-POST-VALIDATION] Insecure test validation is enabled. If you see this outside of a test, it is a severe bug! ")
		log.Warn("*********************************************************************************************")
	}

	s.syncmgr = syncMgrCtor(s.Sync)
	return s, nil
}
```

### Start

**chain/sync.go**

```go
func (syncer *Syncer) Start() {
	tickerCtx, tickerCtxCancel := context.WithCancel(context.Background())
	syncer.syncmgr.Start()

	syncer.tickerCtxCancel = tickerCtxCancel

	go syncer.runMetricsTricker(tickerCtx)
}
```

start方法会调用syncmgr.Start方法：

**chain/sync_manager.go**

```go
func (sm *syncManager) Start() {
	go sm.scheduler()
}
```

**chain/sync_manager.go**

```go
func (sm *syncManager) scheduler() {
	ticker := time.NewTicker(time.Minute)
	tickerC := ticker.C
	for {
		select {
		case head := <-sm.workq:
			sm.handlePeerHead(head)
		case status := <-sm.statusq:
			sm.handleWorkerStatus(status)
		case <-tickerC:
			if sm.initialSyncDone {
				ticker.Stop()
				tickerC = nil
				sm.handleInitialSyncDone()
			}
		case <-sm.ctx.Done():
			return
		}
	}
}
```

当通道接收到**workq（peerHead类型）**的时候，执行 sync_manager 的 handlePeerHead 方法

**chain/sync_manager.go**

```go
func (sm *syncManager) handlePeerHead(head peerHead) {
	log.Debugf("new peer head: %s %s", head.p, head.ts)

	// have we started syncing yet?
	if sm.nextWorker == 0 {
		// track the peer head until we start syncing
		sm.heads[head.p] = head.ts

		// not yet; do we have enough peers?
		if len(sm.heads) < BootstrapPeerThreshold {
			log.Debugw("not tracking enough peers to start sync worker", "have", len(sm.heads), "need", BootstrapPeerThreshold)
			// not enough peers; track it and wait
			return
		}

		// we are ready to start syncing; select the sync target and spawn a worker
		target, err := sm.selectInitialSyncTarget()
		if err != nil {
			log.Errorf("failed to select initial sync target: %s", err)
			return
		}

		log.Infof("selected initial sync target: %s", target)
		sm.spawnWorker(target)
		return
	}

	// we have started syncing, add peer head to the queue if applicable and maybe spawn a worker
	// if there is work to do (possibly in a fork)
	target, work, err := sm.addSyncTarget(head.ts)
	if err != nil {
		log.Warnf("failed to add sync target: %s", err)
		return
	}

	if work {
		log.Infof("selected sync target: %s", target)
		sm.spawnWorker(target)
	}
}
```

继续看 sm 的 spawnWorker 方法，该方法表示生成worker

**chian/sync_manager.go**

```go
func (sm *syncManager) spawnWorker(target *types.TipSet) {
	id := sm.nextWorker
	sm.nextWorker++
	ws := &workerState{
		id: id,
		ts: target,
		ss: new(SyncerState),
	}
	ws.ss.data.WorkerID = id

	sm.mx.Lock()
	sm.state[id] = ws
	sm.mx.Unlock()

	go sm.worker(ws)
}
```

接下来，启动 worker ，开始同步数据

**chainb/sync_manager.go**

```go
func (sm *syncManager) worker(ws *workerState) {
	log.Infof("worker %d syncing in %s", ws.id, ws.ts)

	start := build.Clock.Now()

	ctx := context.WithValue(sm.ctx, syncStateKey{}, ws.ss)
	err := sm.doSync(ctx, ws.ts)

	ws.dt = build.Clock.Since(start)
	log.Infof("worker %d done; took %s", ws.id, ws.dt)
	select {
	case sm.statusq <- workerStatus{id: ws.id, err: err}:
	case <-sm.ctx.Done():
	}
}
```

### Sync

sync.go 中的 Sync 方法在 Start 方法定义了，Sync 的核心逻辑在 collectChain 方法

**chain/sync.go**

```go
func (syncer *Syncer) Sync(ctx context.Context, maybeHead *types.TipSet) error {
	ctx, span := trace.StartSpan(ctx, "chain.Sync")
	defer span.End()

	if span.IsRecordingEvents() {
		span.AddAttributes(
			trace.StringAttribute("tipset", fmt.Sprint(maybeHead.Cids())),
			trace.Int64Attribute("height", int64(maybeHead.Height())),
		)
	}

	hts := syncer.store.GetHeaviestTipSet()

	if hts.ParentWeight().GreaterThan(maybeHead.ParentWeight()) {
		return nil
	}
	if syncer.Genesis.Equals(maybeHead) || hts.Equals(maybeHead) {
		return nil
	}

	if err := syncer.collectChain(ctx, maybeHead, hts, false); err != nil {
		span.AddAttributes(trace.StringAttribute("col_error", err.Error()))
		span.SetStatus(trace.Status{
			Code:    13,
			Message: err.Error(),
		})
		return xerrors.Errorf("collectChain failed: %w", err)
	}

	// At this point we have accepted and synced to the new `maybeHead`
	// (`StageSyncComplete`).
	if err := syncer.store.PutTipSet(ctx, maybeHead); err != nil {
		span.AddAttributes(trace.StringAttribute("put_error", err.Error()))
		span.SetStatus(trace.Status{
			Code:    13,
			Message: err.Error(),
		})
		return xerrors.Errorf("failed to put synced tipset to chainstore: %w", err)
	}

	peers := syncer.receiptTracker.GetPeers(maybeHead)
	if len(peers) > 0 {
		syncer.connmgr.TagPeer(peers[0], "new-block", 40)

		for _, p := range peers[1:] {
			syncer.connmgr.TagPeer(p, "new-block", 25)
		}
	}

	return nil
}
```

collectChain 会执行三个阶段：

- 第一阶段：StageHeaders，我们通过请求块头来继续同步进程，从我们的同伴，从他们的头部向后移动，直到我们到达一个tipset。我们有共同之处(这种共同之处一定存在，尽管它可能存在简单的生成块）。
  如果 common tipset 是我们的头脑，我们将同步视为“快进”，我们必须删除链的一部分才能连接到对等体的头部(称为“fork”)。
- 第二阶段：StagePersistHeaders，现在我们已经收集了丢失的头信息，在分叉的另一边，我们将它们持久化到BlockStore。
- 第一阶段：StageMessages，获得了头并找到了一个公共的消息集，我们继续，请求完整的块，包括消息。