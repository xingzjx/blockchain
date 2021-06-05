# Swarm之交换合约源码解读
  
- [Swarm之交换合约源码解读](#swarm之交换合约源码解读)
	- [启动过程](#启动过程)
		- [cmd模块](#cmd模块)
		- [node模块](#node模块)
		- [settlement模块](#settlement模块)
	- [工厂合约](#工厂合约)
	- [交换合约](#交换合约)
		- [cashChequeBeneficiary 方法](#cashchequebeneficiary-方法)
		- [paidOut 方法](#paidout-方法)
		- [withdraw 方法](#withdraw-方法)
		- [重要的外部变量](#重要的外部变量)

分析环境：

客户端版本：bee v0.6.0

合约版本：s3 v0.4.0

## 启动过程

### cmd模块

以下方法在工程的cmd目录下执行，bee客户端启动的时候执行如下方法：

main.go

```go
func main() {
	if err := cmd.Execute(); err != nil {
		fmt.Fprintln(os.Stderr, "Error:", err)
		os.Exit(1)
	}
}
```

cmd.go

```go
// Execute parses command line arguments and runs appropriate functions.
func Execute() (err error) {
	c, err := newCommand()
	if err != nil {
		return err
	}
	return c.Execute()
}

```

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

    // 部署合约的启动入口
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

			err = node.CheckOverlayWithStore(signerConfig.address, stateStore)
			if err != nil {
				return err
			}

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
			)

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

### node模块

这个过程会进入node包下，该目录目前有三个类，分别是node.go和chain.go以及statestore.go

chain.go

```go
// InitChequebookService will initialize the chequebook service with the given
// chequebook factory and chain backend.
func InitChequebookService(
	ctx context.Context,
	logger logging.Logger,
	stateStore storage.StateStorer,
	signer crypto.Signer,
	chainID int64,
	backend *ethclient.Client,
	overlayEthAddress common.Address,
	transactionService transaction.Service,
	chequebookFactory chequebook.Factory,
	initialDeposit string,
) (chequebook.Service, error) {
	chequeSigner := chequebook.NewChequeSigner(signer, chainID)

	deposit, ok := new(big.Int).SetString(initialDeposit, 10)
	if !ok {
		return nil, fmt.Errorf("initial swap deposit \"%s\" cannot be parsed", initialDeposit)
	}

	chequebookService, err := chequebook.Init(
		ctx,
		chequebookFactory,
		stateStore,
		logger,
		deposit,
		transactionService,
		backend,
		chainID,
		overlayEthAddress,
		chequeSigner,
	)
	if err != nil {
		return nil, fmt.Errorf("chequebook init: %w", err)
	}

	return chequebookService, nil
}
```

### settlement模块

下面的代码会进入到settlement包下，执行init方法，这个时候开始进入到结算模块的功能：

init.go

```go
// Init initialises the chequebook service.
func Init(
	ctx context.Context,
	chequebookFactory Factory,
	stateStore storage.StateStorer,
	logger logging.Logger,
	swapInitialDeposit *big.Int,
	transactionService transaction.Service,
	swapBackend transaction.Backend,
	chainId int64,
	overlayEthAddress common.Address,
	chequeSigner ChequeSigner,
) (chequebookService Service, err error) {
	// verify that the supplied factory is valid
	err = chequebookFactory.VerifyBytecode(ctx)
	if err != nil {
		return nil, err
	}

	erc20Address, err := chequebookFactory.ERC20Address(ctx)
	if err != nil {
		return nil, err
	}

	erc20Service := erc20.New(swapBackend, transactionService, erc20Address)

	var chequebookAddress common.Address
	err = stateStore.Get(chequebookKey, &chequebookAddress)
	if err != nil {
		if err != storage.ErrNotFound {
			return nil, err
		}

		var txHash common.Hash
		err = stateStore.Get(ChequebookDeploymentKey, &txHash)
		if err != nil && err != storage.ErrNotFound {
			return nil, err
		}
		if err == storage.ErrNotFound {
			logger.Info("no chequebook found, deploying new one.")
			if swapInitialDeposit.Cmp(big.NewInt(0)) != 0 {
				err = checkBalance(ctx, logger, swapInitialDeposit, swapBackend, chainId, overlayEthAddress, erc20Service)
				if err != nil {
					return nil, err
				}
			}

			nonce := make([]byte, 32)
			_, err = rand.Read(nonce)
			if err != nil {
				return nil, err
			}

			// if we don't yet have a chequebook, deploy a new one
			txHash, err = chequebookFactory.Deploy(ctx, overlayEthAddress, big.NewInt(0), common.BytesToHash(nonce))
			if err != nil {
				return nil, err
			}

			logger.Infof("deploying new chequebook in transaction %x", txHash)

			err = stateStore.Put(ChequebookDeploymentKey, txHash)
			if err != nil {
				return nil, err
			}
		} else {
			logger.Infof("waiting for chequebook deployment in transaction %x", txHash)
		}

		chequebookAddress, err = chequebookFactory.WaitDeployed(ctx, txHash)
		if err != nil {
			return nil, err
		}

		logger.Infof("deployed chequebook at address %x", chequebookAddress)

		// save the address for later use
		err = stateStore.Put(chequebookKey, chequebookAddress)
		if err != nil {
			return nil, err
		}

		chequebookService, err = New(transactionService, chequebookAddress, overlayEthAddress, stateStore, chequeSigner, erc20Service)
		if err != nil {
			return nil, err
		}

		if swapInitialDeposit.Cmp(big.NewInt(0)) != 0 {
			logger.Infof("depositing %d token into new chequebook", swapInitialDeposit)
			depositHash, err := chequebookService.Deposit(ctx, swapInitialDeposit)
			if err != nil {
				return nil, err
			}

			logger.Infof("sent deposit transaction %x", depositHash)
			err = chequebookService.WaitForDeposit(ctx, depositHash)
			if err != nil {
				return nil, err
			}

			logger.Info("successfully deposited to chequebook")
		}
	} else {
		chequebookService, err = New(transactionService, chequebookAddress, overlayEthAddress, stateStore, chequeSigner, erc20Service)
		if err != nil {
			return nil, err
		}

		logger.Infof("using existing chequebook %x", chequebookAddress)
	}

	// regardless of how the chequebook service was initialised make sure that the chequebook is valid
	err = chequebookFactory.VerifyChequebook(ctx, chequebookService.Address())
	if err != nil {
		return nil, err
	}

	return chequebookService, nil
}
```

最后，会部署工厂合约，

factory.go

```go
// Deploy deploys a new chequebook and returns once the transaction has been submitted.
func (c *factory) Deploy(ctx context.Context, issuer common.Address, defaultHardDepositTimeoutDuration *big.Int, nonce common.Hash) (common.Hash, error) {
	callData, err := factoryABI.Pack("deploySimpleSwap", issuer, big.NewInt(0).Set(defaultHardDepositTimeoutDuration), nonce)
	if err != nil {
		return common.Hash{}, err
	}

	request := &transaction.TxRequest{
		To:       &c.address,
		Data:     callData,
		GasPrice: nil,
		GasLimit: 0,
		Value:    big.NewInt(0),
	}

	txHash, err := c.transactionService.Send(ctx, request)
	if err != nil {
		return common.Hash{}, err
	}

	return txHash, nil
}
```



## 工厂合约

deploySimpleSwap方法是工厂合约中部署ERC20SimpleSwap合约的入口。

SimpleSwapFactory.sol

```javascript
 /**
  @notice creates a clone of the master SimpleSwap contract
  @param issuer the issuer of cheques for the new chequebook
  @param defaultHardDepositTimeoutDuration duration in seconds which by default will be used to reduce hardDeposit allocations
  @param salt salt to include in create2 to enable the same address to deploy multiple chequebooks
  */
  function deploySimpleSwap(address issuer, uint defaultHardDepositTimeoutDuration, bytes32 salt)
  public returns (address) {    
    address contractAddress = Clones.cloneDeterministic(master, keccak256(abi.encode(msg.sender, salt)));
    ERC20SimpleSwap(contractAddress).init(issuer, ERC20Address, defaultHardDepositTimeoutDuration);
    deployedContracts[contractAddress] = true;
    emit SimpleSwapDeployed(contractAddress);
    return contractAddress;
  }
```

## 交换合约

交换合约实在ERC20SimpleSwap.sol中实现。在ERC20SimpleSwap合约中有许多重要的方法在bee客户端调用，下面分析：

节点启动的时候在node模块的node.go中的NewBee方法，会调用debugapi的注册路由方法，执行过程如下：

debugapi.go

```go
// Configure injects required dependencies and configuration parameters and
// constructs HTTP routes that depend on them. It is intended and safe to call
// this method only once.
func (s *Service) Configure(p2p p2p.DebugService, pingpong pingpong.Interface, topologyDriver topology.Driver, lightNodes *lightnode.Container, storer storage.Storer, tags *tags.Tags, accounting accounting.Interface, pseudosettle settlement.Interface, chequebookEnabled bool, swap swap.Interface, chequebook chequebook.Service, batchStore postage.Storer) {
	s.p2p = p2p
	s.pingpong = pingpong
	s.topologyDriver = topologyDriver
	s.storer = storer
	s.tags = tags
	s.accounting = accounting
	s.chequebookEnabled = chequebookEnabled
	s.chequebook = chequebook
	s.swap = swap
	s.lightNodes = lightNodes
	s.batchStore = batchStore
	s.pseudosettle = pseudosettle

	s.setRouter(s.newRouter())
}
```

router.go

```go
func (s *Service) newRouter() *mux.Router {

    // ...
   
    // 该接口用于提现
    router.Handle("/chequebook/cashout/{peer}", jsonhttp.MethodHandler{
			"GET":  http.HandlerFunc(s.swapCashoutStatusHandler),
			"POST": http.HandlerFunc(s.swapCashoutHandler),
		})

	 // ...
}
```

checkbook.go

```go
func (s *Service) swapCashoutHandler(w http.ResponseWriter, r *http.Request) {
	addr := mux.Vars(r)["peer"]
	peer, err := swarm.ParseHexAddress(addr)
	if err != nil {
		s.logger.Debugf("debug api: cashout peer: invalid peer address %s: %v", addr, err)
		s.logger.Errorf("debug api: cashout peer: invalid peer address %s", addr)
		jsonhttp.NotFound(w, errInvalidAddress)
		return
	}

	ctx := r.Context()
	if price, ok := r.Header[gasPriceHeader]; ok {
		p, ok := big.NewInt(0).SetString(price[0], 10)
		if !ok {
			s.logger.Error("debug api: cashout peer: bad gas price")
			jsonhttp.BadRequest(w, errBadGasPrice)
			return
		}
		ctx = sctx.SetGasPrice(ctx, p)
	}

	if limit, ok := r.Header[gasLimitHeader]; ok {
		l, err := strconv.ParseUint(limit[0], 10, 64)
		if err != nil {
			s.logger.Debugf("debug api: cashout peer: bad gas limit: %v", err)
			s.logger.Error("debug api: cashout peer: bad gas limit")
			jsonhttp.BadRequest(w, errBadGasLimit)
			return
		}
		ctx = sctx.SetGasLimit(ctx, l)
	}

	txHash, err := s.swap.CashCheque(ctx, peer)
	if err != nil {
		s.logger.Debugf("debug api: cashout peer: cannot cash %s: %v", addr, err)
		s.logger.Errorf("debug api: cashout peer: cannot cash %s", addr)
		jsonhttp.InternalServerError(w, errCannotCash)
		return
	}

	jsonhttp.OK(w, swapCashoutResponse{TransactionHash: txHash.String()})
}
```

swap.go

```go
// CashCheque sends a cashing transaction for the last cheque of the peer
func (s *Service) CashCheque(ctx context.Context, peer swarm.Address) (common.Hash, error) {
	chequebookAddress, known, err := s.addressbook.Chequebook(peer)
	if err != nil {
		return common.Hash{}, err
	}
	if !known {
		return common.Hash{}, chequebook.ErrNoCheque
	}
	return s.cashout.CashCheque(ctx, chequebookAddress, s.chequebook.Address())
}
```

cashout.go

```go
// CashCheque sends a cashout transaction for the last cheque of the chequebook
func (s *cashoutService) CashCheque(ctx context.Context, chequebook, recipient common.Address) (common.Hash, error) {
	cheque, err := s.chequeStore.LastCheque(chequebook)
	if err != nil {
		return common.Hash{}, err
	}

	callData, err := chequebookABI.Pack("cashChequeBeneficiary", recipient, cheque.CumulativePayout, cheque.Signature)
	if err != nil {
		return common.Hash{}, err
	}
	lim := sctx.GetGasLimit(ctx)
	if lim == 0 {
		// fix for out of gas errors
		lim = 300000
	}
	request := &transaction.TxRequest{
		To:       &chequebook,
		Data:     callData,
		GasPrice: sctx.GetGasPrice(ctx),
		GasLimit: lim,
		Value:    big.NewInt(0),
	}

	txHash, err := s.transactionService.Send(ctx, request)
	if err != nil {
		return common.Hash{}, err
	}

	err = s.store.Put(cashoutActionKey(chequebook), &cashoutAction{
		TxHash: txHash,
		Cheque: *cheque,
	})
	if err != nil {
		return common.Hash{}, err
	}

	return txHash, nil
}
```

最终，会调用checkbook合约的cashChequeBeneficiary方法，接下来分析智能合约的方法。


### cashChequeBeneficiary 方法

```javascript
 /**
  @dev internal function responsible for checking the issuerSignature, updating hardDeposit balances and doing transfers.
  Called by cashCheque and cashChequeBeneficary
  @param beneficiary the beneficiary to which cheques were assigned. Beneficiary must be an Externally Owned Account
  @param recipient receives the differences between cumulativePayment and what was already paid-out to the beneficiary minus callerPayout
  @param cumulativePayout cumulative amount of cheques assigned to beneficiary
  @param issuerSig if issuer is not the sender, issuer must have given explicit approval on the cumulativePayout to the beneficiary
  */
  function _cashChequeInternal(
    address beneficiary,
    address recipient,
    uint cumulativePayout,
    uint callerPayout,
    bytes memory issuerSig
  ) internal {
    /* The issuer must have given explicit approval to the cumulativePayout, either by being the caller or by signature*/
    if (msg.sender != issuer) {
      require(issuer == recoverEIP712(chequeHash(address(this), beneficiary, cumulativePayout), issuerSig),
      "invalid issuer signature");
    }
    /* the requestPayout is the amount requested for payment processing */
    uint requestPayout = cumulativePayout.sub(paidOut[beneficiary]);
    /* calculates acutal payout */
    uint totalPayout = Math.min(requestPayout, liquidBalanceFor(beneficiary));
    /* calculates hard-deposit usage */
    uint hardDepositUsage = Math.min(totalPayout, hardDeposits[beneficiary].amount);
    require(totalPayout >= callerPayout, "SimpleSwap: cannot pay caller");
    /* if there are some of the hard deposit used, update hardDeposits*/
    if (hardDepositUsage != 0) {
      hardDeposits[beneficiary].amount = hardDeposits[beneficiary].amount.sub(hardDepositUsage);

      totalHardDeposit = totalHardDeposit.sub(hardDepositUsage);
    }
    /* increase the stored paidOut amount to avoid double payout */
    paidOut[beneficiary] = paidOut[beneficiary].add(totalPayout);
    totalPaidOut = totalPaidOut.add(totalPayout);

    /* let the world know that the issuer has over-promised on outstanding cheques */
    if (requestPayout != totalPayout) {
      bounced = true;
      emit ChequeBounced();
    }

    if (callerPayout != 0) {
    /* do a transfer to the caller if specified*/
      require(token.transfer(msg.sender, callerPayout), "transfer failed");
      /* do the actual payment */
      require(token.transfer(recipient, totalPayout.sub(callerPayout)), "transfer failed");
    } else {
      /* do the actual payment */
      require(token.transfer(recipient, totalPayout), "transfer failed");
    }

    emit ChequeCashed(beneficiary, recipient, msg.sender, totalPayout, cumulativePayout, callerPayout);
  }
```


### paidOut 方法

### withdraw 方法

### 重要的外部变量

- issuer

- totalPaidOut

- balance


参考：

[以太坊：什么是ERC20标准](https://www.jianshu.com/p/a5158fbfaeb9)



