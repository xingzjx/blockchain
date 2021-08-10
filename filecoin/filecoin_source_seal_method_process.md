# Filecoin 源码解读：密封核心方法执行流程

- [Filecoin 源码解读：密封核心方法执行流程](#filecoin-源码解读密封核心方法执行流程)
	- [简介](#简介)
	- [矿工节点配置](#矿工节点配置)
	- [storage](#storage)
	- [extern](#extern)
		- [storage-sealing](#storage-sealing)
		- [selector-storage](#selector-storage)
		- [filecoin-ffi](#filecoin-ffi)

分析源码：lotus，filecoin的go语言实现，版本号: v1.11.0

## 简介

AddPiece 方法，是 lotus-worker 的核心方法之一，核心方法中还有 P1,P2, C1, C2 方法。

## 矿工节点配置

在前面的章节讲到，node模块下的 *builder.go* ，定义了节点启动流程中的核心逻辑和服务，也包括矿工相关的逻辑

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
				If(cfg.Splitstore.ColdStoreType == "universal",
					Override(new(dtypes.ColdBlockstore), From(new(dtypes.UniversalBlockstore)))),
				If(cfg.Splitstore.ColdStoreType == "discard",
					Override(new(dtypes.ColdBlockstore), modules.DiscardColdBlockstore)),
				If(cfg.Splitstore.HotStoreType == "badger",
					Override(new(dtypes.HotBlockstore), modules.BadgerHotBlockstore)),
				Override(new(dtypes.SplitBlockstore), modules.SplitBlockstore(cfg)),
				Override(new(dtypes.BasicChainBlockstore), modules.ChainSplitBlockstore),
				Override(new(dtypes.BasicStateBlockstore), modules.StateSplitBlockstore),
				Override(new(dtypes.BaseBlockstore), From(new(dtypes.SplitBlockstore))),
				Override(new(dtypes.ExposedBlockstore), modules.ExposedSplitBlockstore),
				Override(new(dtypes.GCReferenceProtector), modules.SplitBlockstoreGCReferenceProtector),
			),
			If(!cfg.EnableSplitstore,
				Override(new(dtypes.BasicChainBlockstore), modules.ChainFlatBlockstore),
				Override(new(dtypes.BasicStateBlockstore), modules.StateFlatBlockstore),
				Override(new(dtypes.BaseBlockstore), From(new(dtypes.UniversalBlockstore))),
				Override(new(dtypes.ExposedBlockstore), From(new(dtypes.UniversalBlockstore))),
				Override(new(dtypes.GCReferenceProtector), modules.NoopGCReferenceProtector),
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

			ApplyIf(IsType(repo.FullNode), ConfigFullNode(c)),
			ApplyIf(IsType(repo.StorageMiner), ConfigStorageMiner(c)),
		)(settings)
	}
}
```

ConfigStorageMiner ： 定义存储矿工的配置项

- Markets : 定义市场的基本逻辑
- Markets (retrieval) ： 定义检索市场的逻辑
- Markets (storage) ： 定义存储市场逻辑，其中 StorageProvider 会最后会调到 AddPiece 方法
- Config

**node/builder_miner.go**

```go
func ConfigStorageMiner(c interface{}) Option {
	cfg, ok := c.(*config.StorageMiner)
	if !ok {
		return Error(xerrors.Errorf("invalid config from repo, got: %T", c))
	}

	pricingConfig := cfg.Dealmaking.RetrievalPricing
	if pricingConfig.Strategy == config.RetrievalPricingExternalMode {
		if pricingConfig.External == nil {
			return Error(xerrors.New("retrieval pricing policy has been to set to external but external policy config is nil"))
		}

		if pricingConfig.External.Path == "" {
			return Error(xerrors.New("retrieval pricing policy has been to set to external but external script path is empty"))
		}
	} else if pricingConfig.Strategy != config.RetrievalPricingDefaultMode {
		return Error(xerrors.New("retrieval pricing policy must be either default or external"))
	}

	enableLibp2pNode := cfg.Subsystems.EnableMarkets // we enable libp2p nodes if the storage market subsystem is enabled, otherwise we don't

	return Options(
		ConfigCommon(&cfg.Common, enableLibp2pNode),

		Override(new(api.MinerSubsystems), modules.ExtractEnabledMinerSubsystems(cfg.Subsystems)),
		Override(new(stores.LocalStorage), From(new(repo.LockedRepo))),
		Override(new(*stores.Local), modules.LocalStorage),
		Override(new(*stores.Remote), modules.RemoteStorage),
		Override(new(dtypes.RetrievalPricingFunc), modules.RetrievalPricingFunc(cfg.Dealmaking)),

		If(!cfg.Subsystems.EnableMining,
			If(cfg.Subsystems.EnableSealing, Error(xerrors.Errorf("sealing can only be enabled on a mining node"))),
			If(cfg.Subsystems.EnableSectorStorage, Error(xerrors.Errorf("sealing can only be enabled on a mining node"))),
		),
		If(cfg.Subsystems.EnableMining,
			If(!cfg.Subsystems.EnableSealing, Error(xerrors.Errorf("sealing can't be disabled on a mining node yet"))),
			If(!cfg.Subsystems.EnableSectorStorage, Error(xerrors.Errorf("sealing can't be disabled on a mining node yet"))),

			// Sector storage: Proofs
			Override(new(ffiwrapper.Verifier), ffiwrapper.ProofVerifier),
			Override(new(ffiwrapper.Prover), ffiwrapper.ProofProver),
			Override(new(storage2.Prover), From(new(sectorstorage.SectorManager))),

			// Sealing (todo should be under EnableSealing, but storagefsm is currently bundled with storage.Miner)
			Override(new(sealing.SectorIDCounter), modules.SectorIDCounter),
			Override(GetParamsKey, modules.GetParams),

			Override(new(dtypes.SetSealingConfigFunc), modules.NewSetSealConfigFunc),
			Override(new(dtypes.GetSealingConfigFunc), modules.NewGetSealConfigFunc),

			// Mining / proving
			Override(new(*slashfilter.SlashFilter), modules.NewSlashFilter),
			Override(new(*storage.Miner), modules.StorageMiner(config.DefaultStorageMiner().Fees)),
			Override(new(*miner.Miner), modules.SetupBlockProducer),
			Override(new(gen.WinningPoStProver), storage.NewWinningPoStProver),
			Override(new(*storage.Miner), modules.StorageMiner(cfg.Fees)),
			Override(new(sectorblocks.SectorBuilder), From(new(*storage.Miner))),
		),

		If(cfg.Subsystems.EnableSectorStorage,
			// Sector storage
			Override(new(*stores.Index), stores.NewIndex),
			Override(new(stores.SectorIndex), From(new(*stores.Index))),
			Override(new(*sectorstorage.Manager), modules.SectorStorage),
			Override(new(sectorstorage.Unsealer), From(new(*sectorstorage.Manager))),
			Override(new(sectorstorage.SectorManager), From(new(*sectorstorage.Manager))),
			Override(new(storiface.WorkerReturn), From(new(sectorstorage.SectorManager))),
		),

		If(!cfg.Subsystems.EnableSectorStorage,
			Override(new(sectorstorage.StorageAuth), modules.StorageAuthWithURL(cfg.Subsystems.SectorIndexApiInfo)),
			Override(new(modules.MinerStorageService), modules.ConnectStorageService(cfg.Subsystems.SectorIndexApiInfo)),
			Override(new(sectorstorage.Unsealer), From(new(modules.MinerStorageService))),
			Override(new(sectorblocks.SectorBuilder), From(new(modules.MinerStorageService))),
		),
		If(!cfg.Subsystems.EnableSealing,
			Override(new(modules.MinerSealingService), modules.ConnectSealingService(cfg.Subsystems.SealerApiInfo)),
			Override(new(stores.SectorIndex), From(new(modules.MinerSealingService))),
		),

		If(cfg.Subsystems.EnableMarkets,
			// Markets
			Override(new(dtypes.StagingMultiDstore), modules.StagingMultiDatastore),
			Override(new(dtypes.StagingBlockstore), modules.StagingBlockstore),
			Override(new(dtypes.StagingDAG), modules.StagingDAG),
			Override(new(dtypes.StagingGraphsync), modules.StagingGraphsync(cfg.Dealmaking.SimultaneousTransfers)),
			Override(new(dtypes.ProviderPieceStore), modules.NewProviderPieceStore),
			Override(new(*sectorblocks.SectorBlocks), sectorblocks.NewSectorBlocks),

			// Markets (retrieval deps)
			Override(new(sectorstorage.PieceProvider), sectorstorage.NewPieceProvider),
			Override(new(dtypes.RetrievalPricingFunc), modules.RetrievalPricingFunc(config.DealmakingConfig{
				RetrievalPricing: &config.RetrievalPricing{
					Strategy: config.RetrievalPricingDefaultMode,
					Default:  &config.RetrievalPricingDefault{},
				},
			})),
			Override(new(dtypes.RetrievalPricingFunc), modules.RetrievalPricingFunc(cfg.Dealmaking)),

			// Markets (retrieval)
			Override(new(retrievalmarket.RetrievalProviderNode), retrievaladapter.NewRetrievalProviderNode),
			Override(new(rmnet.RetrievalMarketNetwork), modules.RetrievalNetwork),
			Override(new(retrievalmarket.RetrievalProvider), modules.RetrievalProvider),
			Override(new(dtypes.RetrievalDealFilter), modules.RetrievalDealFilter(nil)),
			Override(HandleRetrievalKey, modules.HandleRetrieval),

			// Markets (storage)
			Override(new(dtypes.ProviderDataTransfer), modules.NewProviderDAGServiceDataTransfer),
			Override(new(*storedask.StoredAsk), modules.NewStorageAsk),
			Override(new(dtypes.StorageDealFilter), modules.BasicDealFilter(nil)),
			Override(new(storagemarket.StorageProvider), modules.StorageProvider),
			Override(new(*storageadapter.DealPublisher), storageadapter.NewDealPublisher(nil, storageadapter.PublishMsgConfig{})),
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
			Override(new(dtypes.SetExpectedSealDurationFunc), modules.NewSetExpectedSealDurationFunc),
			Override(new(dtypes.GetExpectedSealDurationFunc), modules.NewGetExpectedSealDurationFunc),
			Override(new(dtypes.SetMaxDealStartDelayFunc), modules.NewSetMaxDealStartDelayFunc),
			Override(new(dtypes.GetMaxDealStartDelayFunc), modules.NewGetMaxDealStartDelayFunc),

			If(cfg.Dealmaking.Filter != "",
				Override(new(dtypes.StorageDealFilter), modules.BasicDealFilter(dealfilter.CliStorageDealFilter(cfg.Dealmaking.Filter))),
			),

			If(cfg.Dealmaking.RetrievalFilter != "",
				Override(new(dtypes.RetrievalDealFilter), modules.RetrievalDealFilter(dealfilter.CliRetrievalDealFilter(cfg.Dealmaking.RetrievalFilter))),
			),
			Override(new(*storageadapter.DealPublisher), storageadapter.NewDealPublisher(&cfg.Fees, storageadapter.PublishMsgConfig{
				Period:         time.Duration(cfg.Dealmaking.PublishMsgPeriod),
				MaxDealsPerMsg: cfg.Dealmaking.MaxDealsPerPublishMsg,
			})),
			Override(new(storagemarket.StorageProviderNode), storageadapter.NewProviderNodeAdapter(&cfg.Fees, &cfg.Dealmaking)),
		),

		Override(new(sectorstorage.SealerConfig), cfg.Storage),
		Override(new(*storage.AddressSelector), modules.AddressSelector(&cfg.Addresses)),
	)
}
```

继续看 modules.StorageMiner 方法，看最后一行代码

**node/modules/storageminer.go**

```go
func StorageMiner(fc config.MinerFeeConfig) func(params StorageMinerParams) (*storage.Miner, error) {
	return func(params StorageMinerParams) (*storage.Miner, error) {
		var (
			ds     = params.MetadataDS
			mctx   = params.MetricsCtx
			lc     = params.Lifecycle
			api    = params.API
			sealer = params.Sealer
			sc     = params.SectorIDCounter
			verif  = params.Verifier
			prover = params.Prover
			gsd    = params.GetSealingConfigFn
			j      = params.Journal
			as     = params.AddrSel
		)

		maddr, err := minerAddrFromDS(ds)
		if err != nil {
			return nil, err
		}

		ctx := helpers.LifecycleCtx(mctx, lc)

		fps, err := storage.NewWindowedPoStScheduler(api, fc, as, sealer, verif, sealer, j, maddr)
		if err != nil {
			return nil, err
		}

		sm, err := storage.NewMiner(api, maddr, ds, sealer, sc, verif, prover, gsd, fc, j, as)
		if err != nil {
			return nil, err
		}

		lc.Append(fx.Hook{
			OnStart: func(context.Context) error {
				go fps.Run(ctx)
				return sm.Run(ctx)
			},
			OnStop: sm.Stop,
		})

		return sm, nil
	}
}
```

storageminer 会调到　storage 模块的　miner.go

## storage 

miner.go run　方法：初始化密封状态机，并且运行状态机

关于状态机的库：https://github.com/filecoin-project/go-statemachine

关于存储状态的库：https://github.com/filecoin-project/go-statestore

**storage/miner.go**

```go
func (m *Miner) Run(ctx context.Context) error {
	if err := m.runPreflightChecks(ctx); err != nil {
		return xerrors.Errorf("miner preflight checks failed: %w", err)
	}

	md, err := m.api.StateMinerProvingDeadline(ctx, m.maddr, types.EmptyTSK)
	if err != nil {
		return xerrors.Errorf("getting miner info: %w", err)
	}

	var (
		// consumer of chain head changes.
		evts        = events.NewEvents(ctx, m.api)
		evtsAdapter = NewEventsAdapter(evts)

		// Create a shim to glue the API required by the sealing component
		// with the API that Lotus is capable of providing.
		// The shim translates between "tipset tokens" and tipset keys, and
		// provides extra methods.
		adaptedAPI = NewSealingAPIAdapter(m.api)

		// Instantiate a precommit policy.
		cfg             = sealing.GetSealingConfigFunc(m.getSealConfig)
		provingBoundary = md.PeriodStart % md.WPoStProvingPeriod
		provingBuffer   = md.WPoStProvingPeriod * 2

		// TODO: Maybe we update this policy after actor upgrades?
		pcp = sealing.NewBasicPreCommitPolicy(adaptedAPI, cfg, provingBoundary, provingBuffer)

		// address selector.
		as = func(ctx context.Context, mi miner.MinerInfo, use api.AddrUse, goodFunds, minFunds abi.TokenAmount) (address.Address, abi.TokenAmount, error) {
			return m.addrSel.AddressFor(ctx, m.api, mi, use, goodFunds, minFunds)
		}
	)

	// Instantiate the sealing FSM.
	m.sealing = sealing.New(ctx, adaptedAPI, m.feeCfg, evtsAdapter, m.maddr, m.ds, m.sealer, m.sc, m.verif, m.prover, &pcp, cfg, m.handleSealingNotifications, as)

	// Run the sealing FSM.
	go m.sealing.Run(ctx) //nolint:errcheck // logged intside the function

	return nil
}
```

## extern 

在 *extern* 目录下，有５个模块，包括　

- filecoin-ffi : 包括 worker 任务的核心逻辑，实现语言是rust
- sector-storage
- serialization-vectors
- storage-sealing　: 密封逻辑的入口
- test-vectors

### storage-sealing

sealing中的New方法：定义密封状态机

**extern/storage-sealing/sealing.go**

```go
func New(mctx context.Context, api SealingAPI, fc config.MinerFeeConfig, events Events, maddr address.Address, ds datastore.Batching, sealer sectorstorage.SectorManager, sc SectorIDCounter, verif ffiwrapper.Verifier, prov ffiwrapper.Prover, pcp PreCommitPolicy, gc GetSealingConfigFunc, notifee SectorStateNotifee, as AddrSel) *Sealing {
	s := &Sealing{
		api:    api,
		feeCfg: fc,
		events: events,

		maddr:  maddr,
		sealer: sealer,
		sc:     sc,
		verif:  verif,
		pcp:    pcp,

		openSectors:    map[abi.SectorID]*openSector{},
		sectorTimers:   map[abi.SectorID]*time.Timer{},
		pendingPieces:  map[cid.Cid]*pendingPiece{},
		assignedPieces: map[abi.SectorID][]cid.Cid{},
		toUpgrade:      map[abi.SectorNumber]struct{}{},

		notifee: notifee,
		addrSel: as,

		terminator:  NewTerminationBatcher(mctx, maddr, api, as, fc, gc),
		precommiter: NewPreCommitBatcher(mctx, maddr, api, as, fc, gc),
		commiter:    NewCommitBatcher(mctx, maddr, api, as, fc, gc, prov),

		getConfig: gc,
		dealInfo:  &CurrentDealInfoManager{api},

		stats: SectorStats{
			bySector: map[abi.SectorID]statSectorState{},
		},
	}
	s.startupWait.Add(1)

	s.sectors = statemachine.New(namespace.Wrap(ds, datastore.NewKey(SectorStorePrefix)), s, SectorInfo{})

	return s
}
```

有关状态机的详细分析后面在讲解，关于密封状态机行为的定义如下:

**extern/storage-sealing/fsm.go**

```go
func (m *Sealing) plan(events []statemachine.Event, state *SectorInfo) (func(statemachine.Context, SectorInfo) error, uint64, error) {
	/////
	// First process all events

	m.logEvents(events, state)

	if m.notifee != nil {
		defer func(before SectorInfo) {
			m.notifee(before, *state)
		}(*state) // take safe-ish copy of the before state (except for nested pointers)
	}

	p := fsmPlanners[state.State]
	if p == nil {
		if len(events) == 1 {
			if _, ok := events[0].User.(globalMutator); ok {
				p = planOne() // in case we're in a really weird state, allow restart / update state / remove
			}
		}

		if p == nil {
			return nil, 0, xerrors.Errorf("planner for state %s not found", state.State)
		}
	}

	processed, err := p(events, state)
	if err != nil {
		return nil, 0, xerrors.Errorf("running planner for state %s failed: %w", state.State, err)
	}

	/////
	// Now decide what to do next

	/*

				      UndefinedSectorState (start)
				       v                     |
				*<- WaitDeals <-> AddPiece   |
				|   |   /--------------------/
				|   v   v
				*<- Packing <- incoming committed capacity
				|   |
				|   v
				|   GetTicket
				|   |   ^
				|   v   |
				*<- PreCommit1 <--> SealPreCommit1Failed
				|   |       ^          ^^
				|   |       *----------++----\
				|   v       v          ||    |
				*<- PreCommit2 --------++--> SealPreCommit2Failed
				|   |                  ||
				|   v          /-------/|
				*   PreCommitting <-----+---> PreCommitFailed
				|   |                   |     ^
				|   v                   |     |
				*<- WaitSeed -----------+-----/
				|   |||  ^              |
				|   |||  \--------*-----/
				|   |||           |
				|   vvv      v----+----> ComputeProofFailed
				*<- Committing    |
				|   |        ^--> CommitFailed
				|   v             ^
		        |   SubmitCommit  |
		        |   |             |
		        |   v             |
				*<- CommitWait ---/
				|   |
				|   v
				|   FinalizeSector <--> FinalizeFailed
				|   |
				|   v
				*<- Proving
				|
				v
				FailedUnrecoverable

	*/

	if err := m.onUpdateSector(context.TODO(), state); err != nil {
		log.Errorw("update sector stats", "error", err)
	}

	switch state.State {
	// Happy path
	case Empty:
		fallthrough
	case WaitDeals:
		return m.handleWaitDeals, processed, nil
	case AddPiece:
		return m.handleAddPiece, processed, nil
	case Packing:
		return m.handlePacking, processed, nil
	case GetTicket:
		return m.handleGetTicket, processed, nil
	case PreCommit1:
		return m.handlePreCommit1, processed, nil
	case PreCommit2:
		return m.handlePreCommit2, processed, nil
	case PreCommitting:
		return m.handlePreCommitting, processed, nil
	case SubmitPreCommitBatch:
		return m.handleSubmitPreCommitBatch, processed, nil
	case PreCommitBatchWait:
		fallthrough
	case PreCommitWait:
		return m.handlePreCommitWait, processed, nil
	case WaitSeed:
		return m.handleWaitSeed, processed, nil
	case Committing:
		return m.handleCommitting, processed, nil
	case SubmitCommit:
		return m.handleSubmitCommit, processed, nil
	case SubmitCommitAggregate:
		return m.handleSubmitCommitAggregate, processed, nil
	case CommitAggregateWait:
		fallthrough
	case CommitWait:
		return m.handleCommitWait, processed, nil
	case CommitFinalize:
		fallthrough
	case FinalizeSector:
		return m.handleFinalizeSector, processed, nil

	// Handled failure modes
	case SealPreCommit1Failed:
		return m.handleSealPrecommit1Failed, processed, nil
	case SealPreCommit2Failed:
		return m.handleSealPrecommit2Failed, processed, nil
	case PreCommitFailed:
		return m.handlePreCommitFailed, processed, nil
	case ComputeProofFailed:
		return m.handleComputeProofFailed, processed, nil
	case CommitFailed:
		return m.handleCommitFailed, processed, nil
	case CommitFinalizeFailed:
		fallthrough
	case FinalizeFailed:
		return m.handleFinalizeFailed, processed, nil
	case PackingFailed: // DEPRECATED: remove this for the next reset
		state.State = DealsExpired
		fallthrough
	case DealsExpired:
		return m.handleDealsExpired, processed, nil
	case RecoverDealIDs:
		return m.handleRecoverDealIDs, processed, nil

	// Post-seal
	case Proving:
		return m.handleProvingSector, processed, nil
	case Terminating:
		return m.handleTerminating, processed, nil
	case TerminateWait:
		return m.handleTerminateWait, processed, nil
	case TerminateFinality:
		return m.handleTerminateFinality, processed, nil
	case TerminateFailed:
		return m.handleTerminateFailed, processed, nil
	case Removing:
		return m.handleRemoving, processed, nil
	case Removed:
		return nil, processed, nil

	case RemoveFailed:
		return m.handleRemoveFailed, processed, nil

		// Faults
	case Faulty:
		return m.handleFaulty, processed, nil
	case FaultReported:
		return m.handleFaultReported, processed, nil

	// Fatal errors
	case UndefinedSectorState:
		log.Error("sector update with undefined state!")
	case FailedUnrecoverable:
		log.Errorf("sector %d failed unrecoverably", state.SectorNumber)
	default:
		log.Errorf("unexpected sector update state: %s", state.State)
	}

	return nil, processed, nil
}
```

在plan方法中可以看到密封的核心接口，handleAddPiece，handlePreCommit1等等。

关于handler系列的核心方法中，有两个入口，分别是 **input.go** 和　**states_sealing.go**


**extern/storage-sealing/input.go**

```go
func (m *Sealing) handleAddPiece(ctx statemachine.Context, sector SectorInfo) error {
	ssize, err := sector.SectorType.SectorSize()
	if err != nil {
		return err
	}

	res := SectorPieceAdded{}

	m.inputLk.Lock()

	pending, ok := m.assignedPieces[m.minerSectorID(sector.SectorNumber)]
	if ok {
		delete(m.assignedPieces, m.minerSectorID(sector.SectorNumber))
	}
	m.inputLk.Unlock()
	if !ok {
		// nothing to do here (might happen after a restart in AddPiece)
		return ctx.Send(res)
	}

	var offset abi.UnpaddedPieceSize
	pieceSizes := make([]abi.UnpaddedPieceSize, len(sector.Pieces))
	for i, p := range sector.Pieces {
		pieceSizes[i] = p.Piece.Size.Unpadded()
		offset += p.Piece.Size.Unpadded()
	}

	maxDeals, err := getDealPerSectorLimit(ssize)
	if err != nil {
		return xerrors.Errorf("getting per-sector deal limit: %w", err)
	}

	for i, piece := range pending {
		m.inputLk.Lock()
		deal, ok := m.pendingPieces[piece]
		m.inputLk.Unlock()
		if !ok {
			return xerrors.Errorf("piece %s assigned to sector %d not found", piece, sector.SectorNumber)
		}

		if len(sector.dealIDs())+(i+1) > maxDeals {
			// todo: this is rather unlikely to happen, but in case it does, return the deal to waiting queue instead of failing it
			deal.accepted(sector.SectorNumber, offset, xerrors.Errorf("too many deals assigned to sector %d, dropping deal", sector.SectorNumber))
			continue
		}

		pads, padLength := ffiwrapper.GetRequiredPadding(offset.Padded(), deal.size.Padded())

		if offset.Padded()+padLength+deal.size.Padded() > abi.PaddedPieceSize(ssize) {
			// todo: this is rather unlikely to happen, but in case it does, return the deal to waiting queue instead of failing it
			deal.accepted(sector.SectorNumber, offset, xerrors.Errorf("piece %s assigned to sector %d with not enough space", piece, sector.SectorNumber))
			continue
		}

		offset += padLength.Unpadded()

		for _, p := range pads {
			ppi, err := m.sealer.AddPiece(sectorstorage.WithPriority(ctx.Context(), DealSectorPriority),
				m.minerSector(sector.SectorType, sector.SectorNumber),
				pieceSizes,
				p.Unpadded(),
				NewNullReader(p.Unpadded()))
			if err != nil {
				err = xerrors.Errorf("writing padding piece: %w", err)
				deal.accepted(sector.SectorNumber, offset, err)
				return ctx.Send(SectorAddPieceFailed{err})
			}

			pieceSizes = append(pieceSizes, p.Unpadded())
			res.NewPieces = append(res.NewPieces, Piece{
				Piece: ppi,
			})
		}

		ppi, err := m.sealer.AddPiece(sectorstorage.WithPriority(ctx.Context(), DealSectorPriority),
			m.minerSector(sector.SectorType, sector.SectorNumber),
			pieceSizes,
			deal.size,
			deal.data)
		if err != nil {
			err = xerrors.Errorf("writing piece: %w", err)
			deal.accepted(sector.SectorNumber, offset, err)
			return ctx.Send(SectorAddPieceFailed{err})
		}

		log.Infow("deal added to a sector", "deal", deal.deal.DealID, "sector", sector.SectorNumber, "piece", ppi.PieceCID)

		deal.accepted(sector.SectorNumber, offset, nil)

		offset += deal.size
		pieceSizes = append(pieceSizes, deal.size)

		res.NewPieces = append(res.NewPieces, Piece{
			Piece:    ppi,
			DealInfo: &deal.deal,
		})
	}

	return ctx.Send(res)
}
```

接下来，会调到 selector-storage 模块。

其中，C1、C2、P1、P2方法的执行流程差不多，只不过通过　states_sealing.go 调用到了　selector-storage　模块。

### selector-storage

**extern/selector-storage/sealer_cgo.go**

在　AddPiece 方法中，调用了　filecoin-ffi模块的proofs.go　的　GenerateUnsealedCID　方法

```go
func (sb *Sealer) AddPiece(ctx context.Context, sector storage.SectorRef, existingPieceSizes []abi.UnpaddedPieceSize, pieceSize abi.UnpaddedPieceSize, file storage.Data) (abi.PieceInfo, error) {
	// TODO: allow tuning those:
	chunk := abi.PaddedPieceSize(4 << 20)
	parallel := runtime.NumCPU()

	var offset abi.UnpaddedPieceSize
	for _, size := range existingPieceSizes {
		offset += size
	}

	ssize, err := sector.ProofType.SectorSize()
	if err != nil {
		return abi.PieceInfo{}, err
	}

	maxPieceSize := abi.PaddedPieceSize(ssize)

	if offset.Padded()+pieceSize.Padded() > maxPieceSize {
		return abi.PieceInfo{}, xerrors.Errorf("can't add %d byte piece to sector %v with %d bytes of existing pieces", pieceSize, sector, offset)
	}

	var done func()
	var stagedFile *partialfile.PartialFile

	defer func() {
		if done != nil {
			done()
		}

		if stagedFile != nil {
			if err := stagedFile.Close(); err != nil {
				log.Errorf("closing staged file: %+v", err)
			}
		}
	}()

	var stagedPath storiface.SectorPaths
	if len(existingPieceSizes) == 0 {
		stagedPath, done, err = sb.sectors.AcquireSector(ctx, sector, 0, storiface.FTUnsealed, storiface.PathSealing)
		if err != nil {
			return abi.PieceInfo{}, xerrors.Errorf("acquire unsealed sector: %w", err)
		}

		stagedFile, err = partialfile.CreatePartialFile(maxPieceSize, stagedPath.Unsealed)
		if err != nil {
			return abi.PieceInfo{}, xerrors.Errorf("creating unsealed sector file: %w", err)
		}
	} else {
		stagedPath, done, err = sb.sectors.AcquireSector(ctx, sector, storiface.FTUnsealed, 0, storiface.PathSealing)
		if err != nil {
			return abi.PieceInfo{}, xerrors.Errorf("acquire unsealed sector: %w", err)
		}

		stagedFile, err = partialfile.OpenPartialFile(maxPieceSize, stagedPath.Unsealed)
		if err != nil {
			return abi.PieceInfo{}, xerrors.Errorf("opening unsealed sector file: %w", err)
		}
	}

	w, err := stagedFile.Writer(storiface.UnpaddedByteIndex(offset).Padded(), pieceSize.Padded())
	if err != nil {
		return abi.PieceInfo{}, xerrors.Errorf("getting partial file writer: %w", err)
	}

	pw := fr32.NewPadWriter(w)

	pr := io.TeeReader(io.LimitReader(file, int64(pieceSize)), pw)

	throttle := make(chan []byte, parallel)
	piecePromises := make([]func() (abi.PieceInfo, error), 0)

	buf := make([]byte, chunk.Unpadded())
	for i := 0; i < parallel; i++ {
		if abi.UnpaddedPieceSize(i)*chunk.Unpadded() >= pieceSize {
			break // won't use this many buffers
		}
		throttle <- make([]byte, chunk.Unpadded())
	}

	for {
		var read int
		for rbuf := buf; len(rbuf) > 0; {
			n, err := pr.Read(rbuf)
			if err != nil && err != io.EOF {
				return abi.PieceInfo{}, xerrors.Errorf("pr read error: %w", err)
			}

			rbuf = rbuf[n:]
			read += n

			if err == io.EOF {
				break
			}
		}
		if read == 0 {
			break
		}

		done := make(chan struct {
			cid.Cid
			error
		}, 1)
		pbuf := <-throttle
		copy(pbuf, buf[:read])

		go func(read int) {
			defer func() {
				throttle <- pbuf
			}()

			c, err := sb.pieceCid(sector.ProofType, pbuf[:read])
			done <- struct {
				cid.Cid
				error
			}{c, err}
		}(read)

		piecePromises = append(piecePromises, func() (abi.PieceInfo, error) {
			select {
			case e := <-done:
				if e.error != nil {
					return abi.PieceInfo{}, e.error
				}

				return abi.PieceInfo{
					Size:     abi.UnpaddedPieceSize(len(buf[:read])).Padded(),
					PieceCID: e.Cid,
				}, nil
			case <-ctx.Done():
				return abi.PieceInfo{}, ctx.Err()
			}
		})
	}

	if err := pw.Close(); err != nil {
		return abi.PieceInfo{}, xerrors.Errorf("closing padded writer: %w", err)
	}

	if err := stagedFile.MarkAllocated(storiface.UnpaddedByteIndex(offset).Padded(), pieceSize.Padded()); err != nil {
		return abi.PieceInfo{}, xerrors.Errorf("marking data range as allocated: %w", err)
	}

	if err := stagedFile.Close(); err != nil {
		return abi.PieceInfo{}, err
	}
	stagedFile = nil

	if len(piecePromises) == 1 {
		return piecePromises[0]()
	}

	var payloadRoundedBytes abi.PaddedPieceSize
	pieceCids := make([]abi.PieceInfo, len(piecePromises))
	for i, promise := range piecePromises {
		pinfo, err := promise()
		if err != nil {
			return abi.PieceInfo{}, err
		}

		pieceCids[i] = pinfo
		payloadRoundedBytes += pinfo.Size
	}

	pieceCID, err := ffi.GenerateUnsealedCID(sector.ProofType, pieceCids)
	if err != nil {
		return abi.PieceInfo{}, xerrors.Errorf("generate unsealed CID: %w", err)
	}

	// validate that the pieceCID was properly formed
	if _, err := commcid.CIDToPieceCommitmentV1(pieceCID); err != nil {
		return abi.PieceInfo{}, err
	}

	if payloadRoundedBytes < pieceSize.Padded() {
		paddedCid, err := commpffi.ZeroPadPieceCommitment(pieceCID, payloadRoundedBytes.Unpadded(), pieceSize)
		if err != nil {
			return abi.PieceInfo{}, xerrors.Errorf("failed to pad data: %w", err)
		}

		pieceCID = paddedCid
	}

	return abi.PieceInfo{
		Size:     pieceSize.Padded(),
		PieceCID: pieceCID,
	}, nil
}
```

### filecoin-ffi

在该模块中，定义了密封核心方法：包括，GenerateUnsealedCID(AddPiece调用)，SealPreCommitPhase1，SealPreCommitPhase2，SealCommitPhase1，SealCommitPhase2等。接下来看　GenerateUnsealedCID　方法：

**filecoin-ffi/proofs.go**

```go
func GenerateUnsealedCID(proofType abi.RegisteredSealProof, pieces []abi.PieceInfo) (cid.Cid, error) {
	sp, err := toFilRegisteredSealProof(proofType)
	if err != nil {
		return cid.Undef, err
	}

	filPublicPieceInfos, filPublicPieceInfosLen, err := toFilPublicPieceInfos(pieces)
	if err != nil {
		return cid.Undef, err
	}

	resp := generated.FilGenerateDataCommitment(sp, filPublicPieceInfos, filPublicPieceInfosLen)
	resp.Deref()

	defer generated.FilDestroyGenerateDataCommitmentResponse(resp)

	if resp.StatusCode != generated.FCPResponseStatusFCPNoError {
		return cid.Undef, errors.New(generated.RawString(resp.ErrorMsg).Copy())
	}

	return commcid.DataCommitmentV1ToCID(resp.CommD[:])
}
```

**filecoin-ffi/generated/generated.go**

generated 中的　FilGenerateDataCommitment　方法，通过 *cgo* 调用到了本工程的rust目录下的　**api.rs**

```go
func FilGenerateDataCommitment(registeredProof FilRegisteredSealProof, piecesPtr []FilPublicPieceInfo, piecesLen uint) *FilGenerateDataCommitmentResponse {
	cregisteredProof, cregisteredProofAllocMap := (C.fil_RegisteredSealProof)(registeredProof), cgoAllocsUnknown
	cpiecesPtr, cpiecesPtrAllocMap := unpackArgSFilPublicPieceInfo(piecesPtr)
	cpiecesLen, cpiecesLenAllocMap := (C.size_t)(piecesLen), cgoAllocsUnknown
	__ret := C.fil_generate_data_commitment(cregisteredProof, cpiecesPtr, cpiecesLen)
	runtime.KeepAlive(cpiecesLenAllocMap)
	packSFilPublicPieceInfo(piecesPtr, cpiecesPtr)
	runtime.KeepAlive(cpiecesPtrAllocMap)
	runtime.KeepAlive(cregisteredProofAllocMap)
	__v := NewFilGenerateDataCommitmentResponseRef(unsafe.Pointer(__ret))
	return __v
}
```







