# bee 源码解读之初始化过程

环境：bee1.0.0

## 配置方式

bee 提供了两种配置加载方式，如下

### 命令行启动指定

```bash

bee start --config /home/<user>/bee-config.yaml 

```

在命令行下也可以指定其它参数。

### yaml文件指定

bee 默认的配置文件在 etc/bee/bee.yaml 。

```yaml

api-addr: :1633
block-hash: ""
block-time: "15"
bootnode: []
bootnode-mode: false
cache-capacity: "1000000"
clef-signer-enable: false
clef-signer-endpoint: ""
clef-signer-ethereum-address: ""
config: /home/user/.bee.yaml
cors-allowed-origins: []
data-dir: /home/user/.bee
db-block-cache-capacity: "33554432"
db-disable-seeks-compaction: false
db-open-files-limit: "200"
db-write-buffer-size: "33554432"
debug-api-addr: :1635
debug-api-enable: false
full-node: false
gateway-mode: false
global-pinning-enable: false
help: false
mainnet: false
nat-addr: ""
network-id: "10"
p2p-addr: :1634
p2p-quic-enable: false
p2p-ws-enable: false
password: ""
password-file: ""
payment-early: "10000000"
payment-threshold: "100000000"
payment-tolerance: "100000000"
postage-stamp-address: ""
price-oracle-address: ""
resolver-options: []
standalone: false
swap-deployment-gas-price: ""
swap-enable: true
swap-endpoint: ws://localhost:8546
swap-factory-address: ""
swap-initial-deposit: "10000000000000000"
swap-legacy-factory-addresses: []
tracing-enable: false
tracing-endpoint: 127.0.0.1:6831
tracing-service-name: bee
transaction: ""
verbosity: "5"
warmup-time: 20m0s
welcome-message: ""

```

## 配置初始化

### 初始化流程

cmd.go

```go

func newCommand(opts ...option) (c *command, err error) {
	c = &command{
		root: &cobra.Command{
			Use:           "bee",
			Short:         "Ethereum Swarm Bee",
			SilenceErrors: true,
			SilenceUsage:  true,
			PersistentPreRunE: func(cmd *cobra.Command, args []string) error {
				// 1. 初始化参数配置
				return c.initConfig()
			},
		},
	}

	for _, o := range opts {
		o(c)
	}
	if c.passwordReader == nil {
		c.passwordReader = new(stdInPasswordReader)
	}

	// Find home directory.
	if err := c.setHomeDir(); err != nil {
		return nil, err
	}

    // 2. 初始化默认参数
	c.initGlobalFlags()

    // 3. 启动一个 Swarm 节点
	if err := c.initStartCmd(); err != nil {
		return nil, err
	}

    // 4. 初始化 Swarm 节点
	if err := c.initInitCmd(); err != nil {
		return nil, err
	}

    // 5. 部署合约
	if err := c.initDeployCmd(); err != nil {
		return nil, err
	}

    // 6. 初始化版本
	c.initVersionCmd()

	// 7. 初始化 leveldb
	c.initDBCmd()

	if err := c.initConfigurateOptionsCmd(); err != nil {
		return nil, err
	}

	return c, nil
}

```

### 初始化参数配置

初始化参数配置，会从默认的 bee.yaml 文件中读取基本的配置参数。

cmd.go

```go

func (c *command) initConfig() (err error) {
	config := viper.New()
	configName := ".bee"
	if c.cfgFile != "" {
		// Use config file from the flag.
		config.SetConfigFile(c.cfgFile)
	} else {
		// Search config in home directory with name ".bee" (without extension).
		config.AddConfigPath(c.homeDir)
		config.SetConfigName(configName)
	}

	// Environment
	config.SetEnvPrefix("bee")
	config.AutomaticEnv() // read in environment variables that match
	config.SetEnvKeyReplacer(strings.NewReplacer("-", "_"))

	if c.homeDir != "" && c.cfgFile == "" {
		c.cfgFile = filepath.Join(c.homeDir, configName+".yaml")
	}

	// If a config file is found, read it in.
	if err := config.ReadInConfig(); err != nil {
		var e viper.ConfigFileNotFoundError
		if !errors.As(err, &e) {
			return err
		}
	}
	c.config = config
	return nil
}

```

### 初始化默认参数

SwapInitialDeposit: 质押的 bzz 数量，默认是1个 bzz。

```go
func (c *command) setAllFlags(cmd *cobra.Command) {
	cmd.Flags().String(optionNameDataDir, filepath.Join(c.homeDir, ".bee"), "data directory")
	cmd.Flags().Uint64(optionNameCacheCapacity, 1000000, fmt.Sprintf("cache capacity in chunks, multiply by %d to get approximate capacity in bytes", swarm.ChunkSize))
	cmd.Flags().Uint64(optionNameDBOpenFilesLimit, 200, "number of open files allowed by database")
	cmd.Flags().Uint64(optionNameDBBlockCacheCapacity, 32*1024*1024, "size of block cache of the database in bytes")
	cmd.Flags().Uint64(optionNameDBWriteBufferSize, 32*1024*1024, "size of the database write buffer in bytes")
	cmd.Flags().Bool(optionNameDBDisableSeeksCompaction, false, "disables db compactions triggered by seeks")
	cmd.Flags().String(optionNamePassword, "", "password for decrypting keys")
	cmd.Flags().String(optionNamePasswordFile, "", "path to a file that contains password for decrypting keys")
	cmd.Flags().String(optionNameAPIAddr, ":1633", "HTTP API listen address")
	cmd.Flags().String(optionNameP2PAddr, ":1634", "P2P listen address")
	cmd.Flags().String(optionNameNATAddr, "", "NAT exposed address")
	cmd.Flags().Bool(optionNameP2PWSEnable, false, "enable P2P WebSocket transport")
	cmd.Flags().Bool(optionNameP2PQUICEnable, false, "enable P2P QUIC transport")
	cmd.Flags().StringSlice(optionNameBootnodes, []string{}, "initial nodes to connect to")
	cmd.Flags().Bool(optionNameDebugAPIEnable, false, "enable debug HTTP API")
	cmd.Flags().String(optionNameDebugAPIAddr, ":1635", "debug HTTP API listen address")
	cmd.Flags().Uint64(optionNameNetworkID, 10, "ID of the Swarm network")
	cmd.Flags().StringSlice(optionCORSAllowedOrigins, []string{}, "origins with CORS headers enabled")
	cmd.Flags().Bool(optionNameStandalone, false, "whether we want the node to start with no listen addresses for p2p")
	cmd.Flags().Bool(optionNameTracingEnabled, false, "enable tracing")
	cmd.Flags().String(optionNameTracingEndpoint, "127.0.0.1:6831", "endpoint to send tracing data")
	cmd.Flags().String(optionNameTracingServiceName, "bee", "service name identifier for tracing")
	cmd.Flags().String(optionNameVerbosity, "info", "log verbosity level 0=silent, 1=error, 2=warn, 3=info, 4=debug, 5=trace")
	cmd.Flags().String(optionWelcomeMessage, "", "send a welcome message string during handshakes")
	cmd.Flags().Bool(optionNameGlobalPinningEnabled, false, "enable global pinning")
	cmd.Flags().String(optionNamePaymentThreshold, "100000000", "threshold in BZZ where you expect to get paid from your peers")
	cmd.Flags().String(optionNamePaymentTolerance, "100000000", "excess debt above payment threshold in BZZ where you disconnect from your peer")
	cmd.Flags().String(optionNamePaymentEarly, "10000000", "amount in BZZ below the peers payment threshold when we initiate settlement")
	cmd.Flags().StringSlice(optionNameResolverEndpoints, []string{}, "ENS compatible API endpoint for a TLD and with contract address, can be repeated, format [tld:][contract-addr@]url")
	cmd.Flags().Bool(optionNameGatewayMode, false, "disable a set of sensitive features in the api")
	cmd.Flags().Bool(optionNameBootnodeMode, false, "cause the node to always accept incoming connections")
	cmd.Flags().Bool(optionNameClefSignerEnable, false, "enable clef signer")
	cmd.Flags().String(optionNameClefSignerEndpoint, "", "clef signer endpoint")
	cmd.Flags().String(optionNameClefSignerEthereumAddress, "", "ethereum address to use from clef signer")
	cmd.Flags().String(optionNameSwapEndpoint, "ws://localhost:8546", "swap ethereum blockchain endpoint")
	cmd.Flags().String(optionNameSwapFactoryAddress, "", "swap factory addresses")
	cmd.Flags().StringSlice(optionNameSwapLegacyFactoryAddresses, nil, "legacy swap factory addresses")
	cmd.Flags().String(optionNameSwapInitialDeposit, "10000000000000000", "initial deposit if deploying a new chequebook")
	cmd.Flags().Bool(optionNameSwapEnable, true, "enable swap")
	cmd.Flags().Bool(optionNameFullNode, false, "cause the node to start in full mode")
	cmd.Flags().String(optionNamePostageContractAddress, "", "postage stamp contract address")
	cmd.Flags().String(optionNamePriceOracleAddress, "", "price oracle contract address")
	cmd.Flags().String(optionNameTransactionHash, "", "proof-of-identity transaction hash")
	cmd.Flags().String(optionNameBlockHash, "", "block hash of the block whose parent is the block that contains the transaction hash")
	cmd.Flags().Uint64(optionNameBlockTime, 15, "chain block time")
	cmd.Flags().String(optionNameSwapDeploymentGasPrice, "", "gas price in wei to use for deployment and funding")
	cmd.Flags().Duration(optionWarmUpTime, time.Minute*20, "time to warmup the node before pull/push protocols can be kicked off.")
	cmd.Flags().Bool(optionNameMainNet, false, "triggers connect to main net bootnodes.")
}
```

### 启动 Swarm 节点

initStartCmd 方法会从config中读取之前配置的参数，启动一个 swarm 节点。

start.go

```go

func (c *command) initStartCmd() (err error) {

	cmd := &cobra.Command{
		Use:   "start",
		Short: "Start a Swarm node",
		RunE: func(cmd *cobra.Command, args []string) (err error) {
			if len(args) > 0 {
				return cmd.Help()
			}

			v := strings.ToLower(c.config.GetString(optionNameVerbosity))
			logger, err := newLogger(cmd, v)
			if err != nil {
				return fmt.Errorf("new logger: %v", err)
			}

			go startTimeBomb(logger)

			isWindowsService, err := isWindowsService()
			if err != nil {
				return fmt.Errorf("failed to determine if we are running in service: %w", err)
			}

			if isWindowsService {
				var err error
				logger, err = createWindowsEventLogger(serviceName, logger)
				if err != nil {
					return fmt.Errorf("failed to create windows logger %w", err)
				}
			}

			// If the resolver is specified, resolve all connection strings
			// and fail on any errors.
			var resolverCfgs []multiresolver.ConnectionConfig
			resolverEndpoints := c.config.GetStringSlice(optionNameResolverEndpoints)
			if len(resolverEndpoints) > 0 {
				resolverCfgs, err = multiresolver.ParseConnectionStrings(resolverEndpoints)
				if err != nil {
					return err
				}
			}

			beeASCII := `
Welcome to Swarm.... Bzzz Bzzzz Bzzzz
                \     /
            \    o ^ o    /
              \ (     ) /
   ____________(%%%%%%%)____________
  (     /   /  )%%%%%%%(  \   \     )
  (___/___/__/           \__\___\___)
     (     /  /(%%%%%%%)\  \     )
      (__/___/ (%%%%%%%) \___\__)
              /(       )\
            /   (%%%%%)   \
                 (%%%)
                   !                   `

			fmt.Println(beeASCII)
			fmt.Print(`
DISCLAIMER:
This software is provided to you "as is", use at your own risk and without warranties of any kind.
It is your responsibility to read and understand how Swarm works and the implications of running this software.
The usage of Bee involves various risks, including, but not limited to:
damage to hardware or loss of funds associated with the Ethereum account connected to your node.
No developers or entity involved will be liable for any claims and damages associated with your use,
inability to use, or your interaction with other nodes or the software.`)

			fmt.Printf("\n\nversion: %v - planned to be supported until %v, please follow http://ethswarm.org/\n\n", bee.Version, endSupportDate())

			debugAPIAddr := c.config.GetString(optionNameDebugAPIAddr)
			if !c.config.GetBool(optionNameDebugAPIEnable) {
				debugAPIAddr = ""
			}

			signerConfig, err := c.configureSigner(cmd, logger)
			if err != nil {
				return err
			}

			logger.Infof("version: %v", bee.Version)

			bootNode := c.config.GetBool(optionNameBootnodeMode)
			fullNode := c.config.GetBool(optionNameFullNode)

			if bootNode && !fullNode {
				return errors.New("boot node must be started as a full node")
			}

			mainnet := c.config.GetBool(optionNameMainNet)

			networkID := c.config.GetUint64(optionNameNetworkID)
			networkID, err = parseNetworks(mainnet, networkID)
			if err != nil {
				return err
			}

			bootnodes := c.config.GetStringSlice(optionNameBootnodes)
			bootnodes = parseBootnodes(logger, mainnet, networkID, bootnodes)

			b, err := node.NewBee(c.config.GetString(optionNameP2PAddr), signerConfig.publicKey, signerConfig.signer, networkID, logger, signerConfig.libp2pPrivateKey, signerConfig.pssPrivateKey, &node.Options{
				DataDir:                    c.config.GetString(optionNameDataDir),
				CacheCapacity:              c.config.GetUint64(optionNameCacheCapacity),
				DBOpenFilesLimit:           c.config.GetUint64(optionNameDBOpenFilesLimit),
				DBBlockCacheCapacity:       c.config.GetUint64(optionNameDBBlockCacheCapacity),
				DBWriteBufferSize:          c.config.GetUint64(optionNameDBWriteBufferSize),
				DBDisableSeeksCompaction:   c.config.GetBool(optionNameDBDisableSeeksCompaction),
				APIAddr:                    c.config.GetString(optionNameAPIAddr),
				DebugAPIAddr:               debugAPIAddr,
				Addr:                       c.config.GetString(optionNameP2PAddr),
				NATAddr:                    c.config.GetString(optionNameNATAddr),
				EnableWS:                   c.config.GetBool(optionNameP2PWSEnable),
				EnableQUIC:                 c.config.GetBool(optionNameP2PQUICEnable),
				WelcomeMessage:             c.config.GetString(optionWelcomeMessage),
				Bootnodes:                  bootnodes,
				CORSAllowedOrigins:         c.config.GetStringSlice(optionCORSAllowedOrigins),
				Standalone:                 c.config.GetBool(optionNameStandalone),
				TracingEnabled:             c.config.GetBool(optionNameTracingEnabled),
				TracingEndpoint:            c.config.GetString(optionNameTracingEndpoint),
				TracingServiceName:         c.config.GetString(optionNameTracingServiceName),
				Logger:                     logger,
				GlobalPinningEnabled:       c.config.GetBool(optionNameGlobalPinningEnabled),
				PaymentThreshold:           c.config.GetString(optionNamePaymentThreshold),
				PaymentTolerance:           c.config.GetString(optionNamePaymentTolerance),
				PaymentEarly:               c.config.GetString(optionNamePaymentEarly),
				ResolverConnectionCfgs:     resolverCfgs,
				GatewayMode:                c.config.GetBool(optionNameGatewayMode),
				BootnodeMode:               bootNode,
				SwapEndpoint:               c.config.GetString(optionNameSwapEndpoint),
				SwapFactoryAddress:         c.config.GetString(optionNameSwapFactoryAddress),
				SwapLegacyFactoryAddresses: c.config.GetStringSlice(optionNameSwapLegacyFactoryAddresses),
				SwapInitialDeposit:         c.config.GetString(optionNameSwapInitialDeposit),
				SwapEnable:                 c.config.GetBool(optionNameSwapEnable),
				FullNodeMode:               fullNode,
				Transaction:                c.config.GetString(optionNameTransactionHash),
				BlockHash:                  c.config.GetString(optionNameBlockHash),
				PostageContractAddress:     c.config.GetString(optionNamePostageContractAddress),
				PriceOracleAddress:         c.config.GetString(optionNamePriceOracleAddress),
				BlockTime:                  c.config.GetUint64(optionNameBlockTime),
				DeployGasPrice:             c.config.GetString(optionNameSwapDeploymentGasPrice),
				WarmupTime:                 c.config.GetDuration(optionWarmUpTime),
			})
			if err != nil {
				return err
			}

			// Wait for termination or interrupt signals.
			// We want to clean up things at the end.
			interruptChannel := make(chan os.Signal, 1)
			signal.Notify(interruptChannel, syscall.SIGINT, syscall.SIGTERM)

			p := &program{
				start: func() {
					// Block main goroutine until it is interrupted
					sig := <-interruptChannel

					logger.Debugf("received signal: %v", sig)
					logger.Info("shutting down")
				},
				stop: func() {
					// Shutdown
					done := make(chan struct{})
					go func() {
						defer close(done)

						ctx, cancel := context.WithTimeout(context.Background(), 15*time.Second)
						defer cancel()

						if err := b.Shutdown(ctx); err != nil {
							logger.Errorf("shutdown: %v", err)
						}
					}()

					// If shutdown function is blocking too long,
					// allow process termination by receiving another signal.
					select {
					case sig := <-interruptChannel:
						logger.Debugf("received signal: %v", sig)
					case <-done:
					}
				},
			}

			if isWindowsService {
				s, err := service.New(p, &service.Config{
					Name:        serviceName,
					DisplayName: "Bee",
					Description: "Bee, Swarm client.",
				})
				if err != nil {
					return err
				}

				if err = s.Run(); err != nil {
					return err
				}
			} else {
				// start blocks until some interrupt is received
				p.start()
				p.stop()
			}

			return nil
		},
		PreRunE: func(cmd *cobra.Command, args []string) error {
			return c.config.BindPFlags(cmd.Flags())
		},
	}

	c.setAllFlags(cmd)
	c.root.AddCommand(cmd)
	return nil
}

```

### 初始化 Swarm 节点 

ini.go

```go
func (c *command) initInitCmd() (err error) {
	cmd := &cobra.Command{
		Use:   "init",
		Short: "Initialise a Swarm node",
		RunE: func(cmd *cobra.Command, args []string) (err error) {
			if len(args) > 0 {
				return cmd.Help()
			}

			v := strings.ToLower(c.config.GetString(optionNameVerbosity))
			logger, err := newLogger(cmd, v)
			if err != nil {
				return fmt.Errorf("new logger: %v", err)
			}
			_, err = c.configureSigner(cmd, logger)
			if err != nil {
				return err
			}

			dataDir := c.config.GetString(optionNameDataDir)
			stateStore, err := node.InitStateStore(logger, dataDir)
			if err != nil {
				return err
			}

			defer stateStore.Close()

			return nil
		},
		PreRunE: func(cmd *cobra.Command, args []string) error {
			return c.config.BindPFlags(cmd.Flags())
		},
	}

	c.setAllFlags(cmd)
	c.root.AddCommand(cmd)
	return nil
}

```

### 初始化合约

initDeployCmd 方法：部署支票薄合约，目前在主网测试，可以不需要质押 bzz，但是要提供手续费 xDai。

deploy.go

```go

func (c *command) initDeployCmd() error {
	cmd := &cobra.Command{
		Use:   "deploy",
		Short: "Deploy and fund the chequebook contract",
		RunE: func(cmd *cobra.Command, args []string) (err error) {
			if (len(args)) > 0 {
				return cmd.Help()
			}

			v := strings.ToLower(c.config.GetString(optionNameVerbosity))
			logger, err := newLogger(cmd, v)
			if err != nil {
				return fmt.Errorf("new logger: %v", err)
			}

			dataDir := c.config.GetString(optionNameDataDir)
			factoryAddress := c.config.GetString(optionNameSwapFactoryAddress)
			swapInitialDeposit := c.config.GetString(optionNameSwapInitialDeposit)
			swapEndpoint := c.config.GetString(optionNameSwapEndpoint)
			deployGasPrice := c.config.GetString(optionNameSwapDeploymentGasPrice)
			networkID := c.config.GetUint64(optionNameNetworkID)

			stateStore, err := node.InitStateStore(logger, dataDir)
			if err != nil {
				return err
			}

			defer stateStore.Close()

			signerConfig, err := c.configureSigner(cmd, logger)
			if err != nil {
				return err
			}
			signer := signerConfig.signer

			ctx := cmd.Context()

			swapBackend, overlayEthAddress, chainID, transactionMonitor, transactionService, err := node.InitChain(
				ctx,
				logger,
				stateStore,
				swapEndpoint,
				signer,
				blocktime,
			)
			if err != nil {
				return err
			}
			defer swapBackend.Close()
			defer transactionMonitor.Close()

			chequebookFactory, err := node.InitChequebookFactory(
				logger,
				swapBackend,
				chainID,
				transactionService,
				factoryAddress,
				nil,
			)
			if err != nil {
				return err
			}

			_, err = node.InitChequebookService(
				ctx,
				logger,
				stateStore,
				signer,
				chainID,
				swapBackend,
				overlayEthAddress,
				transactionService,
				chequebookFactory,
				swapInitialDeposit,
				deployGasPrice,
			)
			if err != nil {
				return err
			}

			optionTrxHash := c.config.GetString(optionNameTransactionHash)
			optionBlockHash := c.config.GetString(optionNameBlockHash)

			txHash, err := node.GetTxHash(stateStore, logger, optionTrxHash)
			if err != nil {
				return fmt.Errorf("invalid transaction hash: %w", err)
			}

			blockTime := time.Duration(c.config.GetUint64(optionNameBlockTime)) * time.Second

			blockHash, err := node.GetTxNextBlock(ctx, logger, swapBackend, transactionMonitor, blockTime, txHash, optionBlockHash)
			if err != nil {
				return err
			}

			pubKey, err := signer.PublicKey()
			if err != nil {
				return err
			}

			swarmAddress, err := crypto.NewOverlayAddress(*pubKey, networkID, blockHash)

			err = node.CheckOverlayWithStore(swarmAddress, stateStore)

			return err
		},
		PreRunE: func(cmd *cobra.Command, args []string) error {
			return c.config.BindPFlags(cmd.Flags())
		},
	}

	c.setAllFlags(cmd)
	c.root.AddCommand(cmd)

	return nil
}

```

### 初始化版本

version.go

```go

func (c *command) initVersionCmd() {
	v := &cobra.Command{
		Use:   "version",
		Short: "Print version number",
		Run: func(cmd *cobra.Command, args []string) {
			cmd.Println(bee.Version)
		},
	}
	v.SetOut(c.root.OutOrStdout())
	c.root.AddCommand(v)
}

```

### 初始化 db

```go

func (c *command) initDBCmd() {
	cmd := &cobra.Command{
		Use:   "db",
		Short: "Perform basic DB related operations",
	}

	dbExportCmd(cmd)
	dbImportCmd(cmd)

	c.root.AddCommand(cmd)
}

```

在 db 模块主要可以配置4个参数：

cmd.go

```go

const (
	...
	optionNameDBOpenFilesLimit           = "db-open-files-limit"
	optionNameDBBlockCacheCapacity       = "db-block-cache-capacity"
	optionNameDBWriteBufferSize          = "db-write-buffer-size"
	optionNameDBDisableSeeksCompaction   = "db-disable-seeks-compaction"
	...
)

```

db-open-files-limit：为了适应功能较弱的硬件和操作系统，开放文件限制被故意设置得很低，默认200。如果使用硬件时可能的话，可以尝试将其增加到10000或更多。

leveldb 的初始化参数如下：

shed/db.go

```go

func NewDB(path string, o *Options) (db *DB, err error) {
	if o == nil {
		o = &Options{
			OpenFilesLimit:         defaultOpenFilesLimit,
			BlockCacheCapacity:     defaultBlockCacheCapacity,
			WriteBufferSize:        defaultWriteBufferSize,
			DisableSeeksCompaction: defaultDisableSeeksCompaction,
		}
	}
	var ldb *leveldb.DB
	if path == "" {
		ldb, err = leveldb.Open(storage.NewMemStorage(), nil)
	} else {
		ldb, err = leveldb.OpenFile(path, &opt.Options{
			OpenFilesCacheCapacity: int(o.OpenFilesLimit),
			BlockCacheCapacity:     int(o.BlockCacheCapacity),
			WriteBuffer:            int(o.WriteBufferSize),
			DisableSeeksCompaction: o.DisableSeeksCompaction,
		})
	}

	if err != nil {
		return nil, err
	}

	return NewDBWrap(ldb)
}

```
