# bee 源码解读之配置


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

	c.initGlobalFlags()

	if err := c.initStartCmd(); err != nil {
		return nil, err
	}

	if err := c.initInitCmd(); err != nil {
		return nil, err
	}

	if err := c.initDeployCmd(); err != nil {
		return nil, err
	}

	c.initVersionCmd()
	c.initDBCmd()

	if err := c.initConfigurateOptionsCmd(); err != nil {
		return nil, err
	}

	return c, nil
}

```




## bzz质押


## db 配置




