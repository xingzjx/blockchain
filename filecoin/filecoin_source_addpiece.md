# Filecoin 源码解读：Workder-AddPiece


分析源码：lotus，filecoin的go语言实现，版本号: v1.11.0

## 简介

AddPiece 方法，是 lotus-worker 的核心方法之一，核心方法中还有 P1,P2, C1, C2 方法。

## 启动入口

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

在 ConfigStorageMiner 方法中