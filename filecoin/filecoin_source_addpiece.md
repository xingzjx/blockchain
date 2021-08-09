# Filecoin 源码解读：Workder-AddPiece

- [Filecoin 源码解读：Workder-AddPiece](#filecoin-源码解读workder-addpiece)
	- [简介](#简介)
	- [矿工节点配置](#矿工节点配置)
	- [store 状态机](#store-状态机)
	- [markets模块](#markets模块)
	- [storege模块](#storege模块)
	- [extern](#extern)

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

继续看 StorageProvider 方法，看最后一行代码

storageimpl.NewProvider ： 返回 *storage provider* ， 接下来的实现在 [go-fil-markets](https://github.com/filecoin-project/go-fil-markets) 库。

其中， *go-fil-markets* 是  [存储和检索市场子系统](https://spec.filecoin.io/#section-systems.filecoin_markets) 的实现。

**nodes/modules/storageminer**

```go
func StorageProvider(minerAddress dtypes.MinerAddress,
	storedAsk *storedask.StoredAsk,
	h host.Host, ds dtypes.MetadataDS,
	mds dtypes.StagingMultiDstore,
	r repo.LockedRepo,
	pieceStore dtypes.ProviderPieceStore,
	dataTransfer dtypes.ProviderDataTransfer,
	spn storagemarket.StorageProviderNode,
	df dtypes.StorageDealFilter,
) (storagemarket.StorageProvider, error) {
	net := smnet.NewFromLibp2pHost(h)
	store, err := piecefilestore.NewLocalFileStore(piecefilestore.OsPath(r.Path()))
	if err != nil {
		return nil, err
	}

	opt := storageimpl.CustomDealDecisionLogic(storageimpl.DealDeciderFunc(df))

	return storageimpl.NewProvider(net, namespace.Wrap(ds, datastore.NewKey("/deals/provider")), store, mds, pieceStore, dataTransfer, spn, address.Address(minerAddress), storedAsk, opt)
}
```

## store 状态机

下面代码在 go-fil-markets 工程，在创建 *store provider* 的时候，会创建一个状态机 *newProviderStateMachine*

**go-fil-markets@v1.6.2/storagemarket/impl/provider.go**

```go
func NewProvider(net network.StorageMarketNetwork,
	ds datastore.Batching,
	fs filestore.FileStore,
	multiStore *multistore.MultiStore,
	pieceStore piecestore.PieceStore,
	dataTransfer datatransfer.Manager,
	spn storagemarket.StorageProviderNode,
	minerAddress address.Address,
	storedAsk StoredAsk,
	options ...StorageProviderOption,
) (storagemarket.StorageProvider, error) {
	carIO := cario.NewCarIO()
	pio := pieceio.NewPieceIO(carIO, nil, multiStore)

	h := &Provider{
		net:          net,
		spn:          spn,
		fs:           fs,
		multiStore:   multiStore,
		pio:          pio,
		pieceStore:   pieceStore,
		conns:        connmanager.NewConnManager(),
		storedAsk:    storedAsk,
		actor:        minerAddress,
		dataTransfer: dataTransfer,
		pubSub:       pubsub.New(providerDispatcher),
		readyMgr:     shared.NewReadyManager(),
	}
	storageMigrations, err := migrations.ProviderMigrations.Build()
	if err != nil {
		return nil, err
	}
	h.deals, h.migrateDeals, err = newProviderStateMachine(
		ds,
		&providerDealEnvironment{h},
		h.dispatch,
		storageMigrations,
		versioning.VersionKey("1"),
	)
	if err != nil {
		return nil, err
	}
	h.Configure(options...)

	// register a data transfer event handler -- this will send events to the state machines based on DT events
	h.unsubDataTransfer = dataTransfer.SubscribeToEvents(dtutils.ProviderDataTransferSubscriber(h.deals))

	err = dataTransfer.RegisterVoucherType(&requestvalidation.StorageDataTransferVoucher{}, requestvalidation.NewUnifiedRequestValidator(&providerPushDeals{h}, nil))
	if err != nil {
		return nil, err
	}

	err = dataTransfer.RegisterTransportConfigurer(&requestvalidation.StorageDataTransferVoucher{}, dtutils.TransportConfigurer(&providerStoreGetter{h}))
	if err != nil {
		return nil, err
	}

	return h, nil
}
```

**go-fil-markets@v1.6.2/storagemarket/impl/provider.go**

```go
func newProviderStateMachine(ds datastore.Batching, env fsm.Environment, notifier fsm.Notifier, storageMigrations versioning.VersionedMigrationList, target versioning.VersionKey) (fsm.Group, func(context.Context) error, error) {
	return versionedfsm.NewVersionedFSM(ds, fsm.Parameters{
		Environment:     env,
		StateType:       storagemarket.MinerDeal{},
		StateKeyField:   "State",
		Events:          providerstates.ProviderEvents,
		StateEntryFuncs: providerstates.ProviderStateEntryFuncs,
		FinalityStates:  providerstates.ProviderFinalityStates,
		Notifier:        notifier,
	}, storageMigrations, target)
}
```

**go-fil-markets@v1.6.2/storagemarket/impl/providerstates/provider_fsm.go**

```go
var ProviderStateEntryFuncs = fsm.StateEntryFuncs{
	storagemarket.StorageDealValidating:           ValidateDealProposal,
	storagemarket.StorageDealAcceptWait:           DecideOnProposal,
	storagemarket.StorageDealVerifyData:           VerifyData,
	storagemarket.StorageDealReserveProviderFunds: ReserveProviderFunds,
	storagemarket.StorageDealProviderFunding:      WaitForFunding,
	storagemarket.StorageDealPublish:              PublishDeal,
	storagemarket.StorageDealPublishing:           WaitForPublish,
	storagemarket.StorageDealStaged:               HandoffDeal,
	storagemarket.StorageDealAwaitingPreCommit:    VerifyDealPreCommitted,
	storagemarket.StorageDealSealing:              VerifyDealActivated,
	storagemarket.StorageDealRejecting:            RejectDeal,
	storagemarket.StorageDealFinalizing:           CleanupDeal,
	storagemarket.StorageDealActive:               WaitForDealCompletion,
	storagemarket.StorageDealFailing:              FailDeal,
}
```

ProviderStateEntryFuncs 定义了状态机的行为函数，其中 *HandoffDeal* 函数，会处理被发布的订单。

**go-fil-markets@v1.6.2/storagemarket/impl/providerstates/provider_states.go**

```go
func HandoffDeal(ctx fsm.Context, environment ProviderDealEnvironment, deal storagemarket.MinerDeal) error {
	triggerHandoffFailed := func(err error, packingErr error) error {
		if packingErr == nil {
			return ctx.Trigger(storagemarket.ProviderEventDealHandoffFailed, err)
		}
		packingErr = xerrors.Errorf("packing error: %w", packingErr)
		err = xerrors.Errorf("%s: %w", err, packingErr)
		return ctx.Trigger(storagemarket.ProviderEventDealHandoffFailed, err)
	}

	var packingInfo *storagemarket.PackingResult
	if deal.PiecePath != "" {
		// Data for offline deals is stored on disk, so if PiecePath is set,
		// create a Reader from the file path
		file, err := environment.FileStore().Open(deal.PiecePath)
		if err != nil {
			return ctx.Trigger(storagemarket.ProviderEventFileStoreErrored,
				xerrors.Errorf("reading piece at path %s: %w", deal.PiecePath, err))
		}

		// Hand the deal off to the process that adds it to a sector
		packingInfo, err = handoffDeal(ctx.Context(), environment, deal, file, uint64(file.Size()), deal.Proposal.PieceSize)
		if err != nil {
			err = xerrors.Errorf("packing piece at path %s: %w", deal.PiecePath, err)
			return ctx.Trigger(storagemarket.ProviderEventDealHandoffFailed, err)
		}
	} else {
		// Create a reader to read the piece from the blockstore
		pieceReader, pieceSize, err, writeErrChan := environment.GeneratePieceReader(deal.StoreID, deal.Ref.Root, shared.AllSelector())
		if err != nil {
			err := xerrors.Errorf("reading piece %s from store %d: %w", deal.Ref.PieceCid, deal.StoreID, err)
			return ctx.Trigger(storagemarket.ProviderEventDealHandoffFailed, err)
		}

		// Hand the deal off to the process that adds it to a sector
		var packingErr error
		packingInfo, packingErr = handoffDeal(ctx.Context(), environment, deal, pieceReader, pieceSize, deal.Proposal.PieceSize)

		// Close the read side of the pipe
		err = pieceReader.Close()
		if err != nil {
			err = xerrors.Errorf("closing reader for piece %s from store %d: %w", deal.Ref.PieceCid, deal.StoreID, err)
			return triggerHandoffFailed(err, packingErr)
		}

		// Wait for the write to complete
		select {
		case <-ctx.Context().Done():
			return ctx.Trigger(storagemarket.ProviderEventDealHandoffFailed,
				xerrors.Errorf("writing piece %s never finished: %w", deal.Ref.PieceCid, ctx.Context().Err()))
		case err = <-writeErrChan:
			if err != nil {
				err = xerrors.Errorf("writing piece %s: %w", deal.Ref.PieceCid, err)
				return triggerHandoffFailed(err, packingErr)
			}
		}

		if packingErr != nil {
			err = xerrors.Errorf("packing piece %s: %w", deal.Ref.PieceCid, packingErr)
			return ctx.Trigger(storagemarket.ProviderEventDealHandoffFailed, err)
		}
	}

	if err := recordPiece(environment, deal, packingInfo.SectorNumber, packingInfo.Offset, packingInfo.Size); err != nil {
		err = xerrors.Errorf("failed to register deal data for piece %s for retrieval: %w", deal.Ref.PieceCid, err)
		log.Error(err.Error())
		_ = ctx.Trigger(storagemarket.ProviderEventPieceStoreErrored, err)
	}

	return ctx.Trigger(storagemarket.ProviderEventDealHandedOff)
}
```

在 HandoffDeal 函数里面，会调用 handoffDeal，

**go-fil-markets@v1.6.2/storagemarket/impl/providerstates/provider_states.go**

```go
func handoffDeal(ctx context.Context, environment ProviderDealEnvironment, deal storagemarket.MinerDeal, reader io.Reader, payloadSize uint64, pieceSize abi.PaddedPieceSize) (*storagemarket.PackingResult, error) {
	// because we use the PadReader directly during AP we need to produce the
	// correct amount of zeroes
	// (alternative would be to keep precise track of sector offsets for each
	// piece which is just too much work for a seldom used feature)
	paddedReader, err := padreader.NewInflator(reader, payloadSize, pieceSize.Unpadded())
	if err != nil {
		return nil, err
	}
	return environment.Node().OnDealComplete(
		ctx,
		storagemarket.MinerDeal{
			Client:             deal.Client,
			ClientDealProposal: deal.ClientDealProposal,
			ProposalCid:        deal.ProposalCid,
			State:              deal.State,
			Ref:                deal.Ref,
			PublishCid:         deal.PublishCid,
			DealID:             deal.DealID,
			FastRetrieval:      deal.FastRetrieval,
		},
		pieceSize.Unpadded(),
		paddedReader,
	)
}
```

其中 environment.Node()， 指定的是 storagemarket.StorageProviderNode ，而 *OnDealComplete* 的实现在 lotus 工程中。

其源码位于工程目录的 markets/storageadapter/provider.go ，在 *OnDealComplete* 中可以看到，会调用 *AddPiece* 方法。

## markets模块

**markets/storageadapter/provider.go**

```go
func (n *ProviderNodeAdapter) OnDealComplete(ctx context.Context, deal storagemarket.MinerDeal, pieceSize abi.UnpaddedPieceSize, pieceData io.Reader) (*storagemarket.PackingResult, error) {
	if deal.PublishCid == nil {
		return nil, xerrors.Errorf("deal.PublishCid can't be nil")
	}

	sdInfo := api.PieceDealInfo{
		DealID:       deal.DealID,
		DealProposal: &deal.Proposal,
		PublishCid:   deal.PublishCid,
		DealSchedule: api.DealSchedule{
			StartEpoch: deal.ClientDealProposal.Proposal.StartEpoch,
			EndEpoch:   deal.ClientDealProposal.Proposal.EndEpoch,
		},
		KeepUnsealed: deal.FastRetrieval,
	}

	p, offset, err := n.secb.AddPiece(ctx, pieceSize, pieceData, sdInfo)
	curTime := time.Now()
	for time.Since(curTime) < addPieceRetryTimeout {
		if !xerrors.Is(err, sealing.ErrTooManySectorsSealing) {
			if err != nil {
				log.Errorf("failed to addPiece for deal %d, err: %v", deal.DealID, err)
			}
			break
		}
		select {
		case <-time.After(addPieceRetryWait):
			p, offset, err = n.secb.AddPiece(ctx, pieceSize, pieceData, sdInfo)
		case <-ctx.Done():
			return nil, xerrors.New("context expired while waiting to retry AddPiece")
		}
	}

	if err != nil {
		return nil, xerrors.Errorf("AddPiece failed: %s", err)
	}
	log.Warnf("New Deal: deal %d", deal.DealID)

	return &storagemarket.PackingResult{
		SectorNumber: p,
		Offset:       offset,
		Size:         pieceSize.Padded(),
	}, nil
}
```

## storege模块

**storege/selectorblocks/blocks.go**

```go
func (st *SectorBlocks) AddPiece(ctx context.Context, size abi.UnpaddedPieceSize, r io.Reader, d api.PieceDealInfo) (abi.SectorNumber, abi.PaddedPieceSize, error) {
	so, err := st.SectorBuilder.SectorAddPieceToAny(ctx, size, r, d)
	if err != nil {
		return 0, 0, err
	}

	// TODO: DealID has very low finality here
	err = st.writeRef(d.DealID, so.Sector, so.Offset, size)
	if err != nil {
		return 0, 0, xerrors.Errorf("writeRef: %w", err)
	}

	return so.Sector, so.Offset, nil
}
```

**storege/mine_sealing.go**

```go
func (m *Miner) SectorAddPieceToAny(ctx context.Context, size abi.UnpaddedPieceSize, r storage.Data, d api.PieceDealInfo) (api.SectorOffset, error) {
	return m.sealing.SectorAddPieceToAny(ctx, size, r, d)
}
```

接下来　*SectorAddPieceToAny* 会调到　extern 的 storage-sealing 模块。

## extern 

在 *extern* 目录下，有５个模块，包括　

- filecoin-ffi : 包括 worker 任务的核心逻辑，实现语言是rust
- sector-storage
- serialization-vectors
- storage-sealing
- test-vectors