# 以太坊源码解读：交易执行流程

## 前言

在以太坊网络里面，转账、创建合约以及调用合约都是执行一笔交易的过程。其流程划分：

- api接口：前端接口调用，然后把交易添加到交易池里面
- 矿工：矿工负责打包交易
- 区块上链：这个过程会启用以太坊虚拟机解析执行指令
- 获取数据：用户查询交易数据

## api接口

当用户执行一笔交易的时候，会调用api里面的SendTransaction方法：

**internal/ethapi/api.go**

```go

func (s *PublicTransactionPoolAPI) SendTransaction(ctx context.Context, args TransactionArgs) (common.Hash, error) {
	// Look up the wallet containing the requested signer
	account := accounts.Account{Address: args.from()}

	wallet, err := s.b.AccountManager().Find(account)
	if err != nil {
		return common.Hash{}, err
	}

	if args.Nonce == nil {
		// Hold the addresse's mutex around signing to prevent concurrent assignment of
		// the same nonce to multiple accounts.
		s.nonceLock.LockAddr(args.from())
		defer s.nonceLock.UnlockAddr(args.from())
	}

	// Set some sanity defaults and terminate on failure
	if err := args.setDefaults(ctx, s.b); err != nil {
		return common.Hash{}, err
	}
	// Assemble the transaction and sign with the wallet
	tx := args.toTransaction()

	signed, err := wallet.SignTx(account, tx, s.b.ChainConfig().ChainID)
	if err != nil {
		return common.Hash{}, err
	}
	return SubmitTransaction(ctx, s.b, signed)
}

```

在SendTransaction里面调用了SubmitTransaction方法，其核心逻辑：

- b.SendTx： b只的是api_backend.go，接口实现者

**internal/ethapi/api.go**

```go

func SubmitTransaction(ctx context.Context, b Backend, tx *types.Transaction) (common.Hash, error) {
	// If the transaction fee cap is already specified, ensure the
	// fee of the given transaction is _reasonable_.
	if err := checkTxFee(tx.GasPrice(), tx.Gas(), b.RPCTxFeeCap()); err != nil {
		return common.Hash{}, err
	}
	if !b.UnprotectedAllowed() && !tx.Protected() {
		// Ensure only eip155 signed transactions are submitted if EIP155Required is set.
		return common.Hash{}, errors.New("only replay-protected (EIP-155) transactions allowed over RPC")
	}
	if err := b.SendTx(ctx, tx); err != nil {
		return common.Hash{}, err
	}
	// Print a log with full tx details for manual investigations and interventions
	signer := types.MakeSigner(b.ChainConfig(), b.CurrentBlock().Number())
	from, err := types.Sender(signer, tx)
	if err != nil {
		return common.Hash{}, err
	}

	if tx.To() == nil {
		addr := crypto.CreateAddress(from, tx.Nonce())
		log.Info("Submitted contract creation", "hash", tx.Hash().Hex(), "from", from, "nonce", tx.Nonce(), "contract", addr.Hex(), "value", tx.Value())
	} else {
		log.Info("Submitted transaction", "hash", tx.Hash().Hex(), "from", from, "nonce", tx.Nonce(), "recipient", tx.To(), "value", tx.Value())
	}
	return tx.Hash(), nil
}

```

api_backend中会执行SendTx方法：可以看到，交易会添到txPool中，即交易池里面。

**eth/api_backend.go**

```bash

func (b *EthAPIBackend) SendTx(ctx context.Context, signedTx *types.Transaction) error {
	return b.eth.txPool.AddLocal(signedTx)
}

```

TxPool的结构体如下：其中，交易会被添加到结构体里面的 *locals* 成员

**core/tx_pool.go**

```go
type TxPool struct {
	config      TxPoolConfig
	chainconfig *params.ChainConfig
	chain       blockChain
	gasPrice    *big.Int
	txFeed      event.Feed
	scope       event.SubscriptionScope
	signer      types.Signer
	mu          sync.RWMutex

	istanbul bool // Fork indicator whether we are in the istanbul stage.
	eip2718  bool // Fork indicator whether we are using EIP-2718 type transactions.
	eip1559  bool // Fork indicator whether we are using EIP-1559 type transactions.

	currentState  *state.StateDB // Current state in the blockchain head
	pendingNonces *txNoncer      // Pending state tracking virtual nonces
	currentMaxGas uint64         // Current gas limit for transaction caps

	locals  *accountSet // Set of local transaction to exempt from eviction rules
	journal *txJournal  // Journal of local transaction to back up to disk

	pending map[common.Address]*txList   // All currently processable transactions
	queue   map[common.Address]*txList   // Queued but non-processable transactions
	beats   map[common.Address]time.Time // Last heartbeat from each known account
	all     *txLookup                    // All transactions to allow lookups
	priced  *txPricedList                // All transactions sorted by price

	chainHeadCh     chan ChainHeadEvent
	chainHeadSub    event.Subscription
	reqResetCh      chan *txpoolResetRequest
	reqPromoteCh    chan *accountSet
	queueTxEventCh  chan *types.Transaction
	reorgDoneCh     chan chan struct{}
	reorgShutdownCh chan struct{}  // requests shutdown of scheduleReorgLoop
	wg              sync.WaitGroup // tracks loop, scheduleReorgLoop
}
```

## 矿工打包



[Ethereum 以太坊智能合约调用源码分析](https://www.jianshu.com/p/97f7ca54c4a3)

[以太坊源代码 - 挖矿打包流程以及速度](https://blog.csdn.net/Metal1/article/details/80501519)

[死磕以太坊源码分析之挖矿流程](https://blog.csdn.net/pulong0748/article/details/113864246)