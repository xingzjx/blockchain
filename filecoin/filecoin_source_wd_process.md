# Filecoin 源码解读：Wd-Post调用流程


源码分析环境： v1.12.0-rc2

Filecoin网络里，时空证明（POST），分为两种类型，一个是 window post(简称wdpost)，一个是 winning post(简称wnpost)。

接下来，分析wdpost的代码执行流程。


在前面的代码分析中知道，在node.go中会调用node模块中的builder.go的Repo方法。


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
		return Options(
			Override(new(repo.LockedRepo), modules.LockedRepo(lr)), // module handles closing

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

接下来看ConfigStorageMiner，在该方法中会调用modules下的StorageMiner

**node/builder_miner.go**

```go

func ConfigStorageMiner(c interface{}) Option {
    // 省略

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
    // 省略

}

```

StorageMiner的构造方法实现如下：

- 启动wdpost ： fps.Run(ctx)
- 启动wnpost ： sm.Run(ctx)

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

接下来看fps.Run，fps是WindowPoStScheduler类型。

**storage/wdpost_sched.go**

```go
func (s *WindowPoStScheduler) Run(ctx context.Context) {
	// Initialize change handler.

	// callbacks is a union of the fullNodeFilteredAPI and ourselves.
	callbacks := struct {
		fullNodeFilteredAPI
		*WindowPoStScheduler
	}{s.api, s}

	s.ch = newChangeHandler(callbacks, s.actor)
	defer s.ch.shutdown()
	s.ch.start()

	var (
		notifs <-chan []*api.HeadChange
		err    error
		gotCur bool
	)

	// not fine to panic after this point
	for {
		if notifs == nil {
			notifs, err = s.api.ChainNotify(ctx)
			if err != nil {
				log.Errorf("ChainNotify error: %+v", err)

				build.Clock.Sleep(10 * time.Second)
				continue
			}

			gotCur = false
		}

		select {
		case changes, ok := <-notifs:
			if !ok {
				log.Warn("window post scheduler notifs channel closed")
				notifs = nil
				continue
			}

			if !gotCur {
				if len(changes) != 1 {
					log.Errorf("expected first notif to have len = 1")
					continue
				}
				chg := changes[0]
				if chg.Type != store.HCCurrent {
					log.Errorf("expected first notif to tell current ts")
					continue
				}

				ctx, span := trace.StartSpan(ctx, "WindowPoStScheduler.headChange")

				s.update(ctx, nil, chg.Val)

				span.End()
				gotCur = true
				continue
			}

			ctx, span := trace.StartSpan(ctx, "WindowPoStScheduler.headChange")

			var lowest, highest *types.TipSet = nil, nil

			for _, change := range changes {
				if change.Val == nil {
					log.Errorf("change.Val was nil")
				}
				switch change.Type {
				case store.HCRevert:
					lowest = change.Val
				case store.HCApply:
					highest = change.Val
				}
			}

			s.update(ctx, lowest, highest)

			span.End()
		case <-ctx.Done():
			return
		}
	}
}
```

在Run方法里面，执行newChangeHandler，然后执行wdpost_changehandler.go的start方法。

- proveHdlr: 生成证明的参数
- submitHdlr: 提交证明参数到链上


**storage/wdpost_changehandler.go**

```go

func (ch *changeHandler) start() {
	go ch.proveHdlr.run()
	go ch.submitHdlr.run()
}

```

## 生成证明

**storage/wdpost_changehandler.go**

```go
func (p *proveHandler) run() {
	// Abort proving on shutdown
	defer func() {
		if p.current != nil {
			p.current.abort()
		}
	}()

	for p.shutdownCtx.Err() == nil {
		select {
		case <-p.shutdownCtx.Done():
			return

		case hc := <-p.hcs:
			// Head changed
			p.processHeadChange(hc.ctx, hc.advance, hc.di)
			if p.processedHeadChanges != nil {
				p.processedHeadChanges <- hc
			}

		case res := <-p.postResults:
			// Proof generation complete
			p.processPostResult(res)
			if p.processedPostResults != nil {
				p.processedPostResults <- res
			}
		}
	}
}
```

Head changed 的时候，会执行processHeadChange方法，在该方法中会调用生成证明参数的方法，startGeneratePoST。

**storage/wdpost_changehandler.go**

```go

func (p *proveHandler) processHeadChange(ctx context.Context, newTS *types.TipSet, di *dline.Info) {
	// If the post window has expired, abort the current proof
	if p.current != nil && newTS.Height() >= p.current.di.Close {
		// Cancel the context on the current proof
		p.current.abort()

		// Clear out the reference to the proof so that we can immediately
		// start generating a new proof, without having to worry about state
		// getting clobbered when the abort completes
		p.current = nil
	}

	// Only generate one proof at a time
	if p.current != nil {
		return
	}

	// If the proof for the current post window has been generated, check the
	// next post window
	_, complete := p.posts.get(di)
	for complete {
		di = nextDeadline(di)
		_, complete = p.posts.get(di)
	}

	// Check if the chain is above the Challenge height for the post window
	if newTS.Height() < di.Challenge+ChallengeConfidence {
		return
	}

	p.current = &currentPost{di: di}
	curr := p.current
	p.current.abort = p.api.startGeneratePoST(ctx, newTS, di, func(posts []miner.SubmitWindowedPoStParams, err error) {
		p.postResults <- &postResult{ts: newTS, currPost: curr, posts: posts, err: err}
	})
}

```

startGeneratePoST：生成证明参数，其实现在wdpost_run.go

**storage/wdpost_run.go**

```go

func (s *WindowPoStScheduler) startGeneratePoST(
	ctx context.Context,
	ts *types.TipSet,
	deadline *dline.Info,
	completeGeneratePoST CompleteGeneratePoSTCb,
) context.CancelFunc {
	ctx, abort := context.WithCancel(ctx)
	go func() {
		defer abort()

		s.journal.RecordEvent(s.evtTypes[evtTypeWdPoStScheduler], func() interface{} {
			return WdPoStSchedulerEvt{
				evtCommon: s.getEvtCommon(nil),
				State:     SchedulerStateStarted,
			}
		})

		posts, err := s.runGeneratePoST(ctx, ts, deadline)
		completeGeneratePoST(posts, err)
	}()

	return abort
}

```

**storage/wdpost_run.go**

```go

func (s *WindowPoStScheduler) runGeneratePoST(
	ctx context.Context,
	ts *types.TipSet,
	deadline *dline.Info,
) ([]miner.SubmitWindowedPoStParams, error) {
	ctx, span := trace.StartSpan(ctx, "WindowPoStScheduler.generatePoST")
	defer span.End()

	posts, err := s.runPoStCycle(ctx, *deadline, ts)
	if err != nil {
		log.Errorf("runPoStCycle failed: %+v", err)
		return nil, err
	}

	if len(posts) == 0 {
		s.recordProofsEvent(nil, cid.Undef)
	}

	return posts, nil
}

```

runPostCycle方法：执行wdpost的核心方法。

- 在下一个截止日期执行恢复声明。
- 为下一个截止日期执行错误声明。
- 计算并提交证明，批处理分区并确保它们不要超过消息容量。

```go
func (s *WindowPoStScheduler) runPoStCycle(ctx context.Context, di dline.Info, ts *types.TipSet) ([]miner.SubmitWindowedPoStParams, error) {
	ctx, span := trace.StartSpan(ctx, "storage.runPoStCycle")
	defer span.End()

	go func() {
		// TODO: extract from runPoStCycle, run on fault cutoff boundaries

		// check faults / recoveries for the *next* deadline. It's already too
		// late to declare them for this deadline
		declDeadline := (di.Index + 2) % di.WPoStPeriodDeadlines

		partitions, err := s.api.StateMinerPartitions(context.TODO(), s.actor, declDeadline, ts.Key())
		if err != nil {
			log.Errorf("getting partitions: %v", err)
			return
		}

		var (
			sigmsg     *types.SignedMessage
			recoveries []miner.RecoveryDeclaration
			faults     []miner.FaultDeclaration

			// optionalCid returns the CID of the message, or cid.Undef is the
			// message is nil. We don't need the argument (could capture the
			// pointer), but it's clearer and purer like that.
			optionalCid = func(sigmsg *types.SignedMessage) cid.Cid {
				if sigmsg == nil {
					return cid.Undef
				}
				return sigmsg.Cid()
			}
		)

		if recoveries, sigmsg, err = s.declareRecoveries(context.TODO(), declDeadline, partitions, ts.Key()); err != nil {
			// TODO: This is potentially quite bad, but not even trying to post when this fails is objectively worse
			log.Errorf("checking sector recoveries: %v", err)
		}

		s.journal.RecordEvent(s.evtTypes[evtTypeWdPoStRecoveries], func() interface{} {
			j := WdPoStRecoveriesProcessedEvt{
				evtCommon:    s.getEvtCommon(err),
				Declarations: recoveries,
				MessageCID:   optionalCid(sigmsg),
			}
			j.Error = err
			return j
		})

		if ts.Height() > build.UpgradeIgnitionHeight {
			return // FORK: declaring faults after ignition upgrade makes no sense
		}

		if faults, sigmsg, err = s.declareFaults(context.TODO(), declDeadline, partitions, ts.Key()); err != nil {
			// TODO: This is also potentially really bad, but we try to post anyways
			log.Errorf("checking sector faults: %v", err)
		}

		s.journal.RecordEvent(s.evtTypes[evtTypeWdPoStFaults], func() interface{} {
			return WdPoStFaultsProcessedEvt{
				evtCommon:    s.getEvtCommon(err),
				Declarations: faults,
				MessageCID:   optionalCid(sigmsg),
			}
		})
	}()

	buf := new(bytes.Buffer)
	if err := s.actor.MarshalCBOR(buf); err != nil {
		return nil, xerrors.Errorf("failed to marshal address to cbor: %w", err)
	}

	headTs, err := s.api.ChainHead(ctx)
	if err != nil {
		return nil, xerrors.Errorf("getting current head: %w", err)
	}

	rand, err := s.api.StateGetRandomnessFromBeacon(ctx, crypto.DomainSeparationTag_WindowedPoStChallengeSeed, di.Challenge, buf.Bytes(), headTs.Key())
	if err != nil {
		return nil, xerrors.Errorf("failed to get chain randomness from beacon for window post (ts=%d; deadline=%d): %w", ts.Height(), di, err)
	}

	// Get the partitions for the given deadline
	partitions, err := s.api.StateMinerPartitions(ctx, s.actor, di.Index, ts.Key())
	if err != nil {
		return nil, xerrors.Errorf("getting partitions: %w", err)
	}

	nv, err := s.api.StateNetworkVersion(ctx, ts.Key())
	if err != nil {
		return nil, xerrors.Errorf("getting network version: %w", err)
	}

	// Split partitions into batches, so as not to exceed the number of sectors
	// allowed in a single message
	partitionBatches, err := s.batchPartitions(partitions, nv)
	if err != nil {
		return nil, err
	}

	// Generate proofs in batches
	posts := make([]miner.SubmitWindowedPoStParams, 0, len(partitionBatches))
	for batchIdx, batch := range partitionBatches {
		batchPartitionStartIdx := 0
		for _, batch := range partitionBatches[:batchIdx] {
			batchPartitionStartIdx += len(batch)
		}

		params := miner.SubmitWindowedPoStParams{
			Deadline:   di.Index,
			Partitions: make([]miner.PoStPartition, 0, len(batch)),
			Proofs:     nil,
		}

		postSkipped := bitfield.New()
		somethingToProve := false

		// Retry until we run out of sectors to prove.
		for retries := 0; ; retries++ {
			skipCount := uint64(0)
			var partitions []miner.PoStPartition
			var sinfos []proof2.SectorInfo
			for partIdx, partition := range batch {
				// TODO: Can do this in parallel
				toProve, err := bitfield.SubtractBitField(partition.LiveSectors, partition.FaultySectors)
				if err != nil {
					return nil, xerrors.Errorf("removing faults from set of sectors to prove: %w", err)
				}
				toProve, err = bitfield.MergeBitFields(toProve, partition.RecoveringSectors)
				if err != nil {
					return nil, xerrors.Errorf("adding recoveries to set of sectors to prove: %w", err)
				}

				good, err := s.checkSectors(ctx, toProve, ts.Key())
				if err != nil {
					return nil, xerrors.Errorf("checking sectors to skip: %w", err)
				}

				good, err = bitfield.SubtractBitField(good, postSkipped)
				if err != nil {
					return nil, xerrors.Errorf("toProve - postSkipped: %w", err)
				}

				skipped, err := bitfield.SubtractBitField(toProve, good)
				if err != nil {
					return nil, xerrors.Errorf("toProve - good: %w", err)
				}

				sc, err := skipped.Count()
				if err != nil {
					return nil, xerrors.Errorf("getting skipped sector count: %w", err)
				}

				skipCount += sc

				ssi, err := s.sectorsForProof(ctx, good, partition.AllSectors, ts)
				if err != nil {
					return nil, xerrors.Errorf("getting sorted sector info: %w", err)
				}

				if len(ssi) == 0 {
					continue
				}

				sinfos = append(sinfos, ssi...)
				partitions = append(partitions, miner.PoStPartition{
					Index:   uint64(batchPartitionStartIdx + partIdx),
					Skipped: skipped,
				})
			}

			if len(sinfos) == 0 {
				// nothing to prove for this batch
				break
			}

			// Generate proof
			log.Infow("running window post",
				"chain-random", rand,
				"deadline", di,
				"height", ts.Height(),
				"skipped", skipCount)

			tsStart := build.Clock.Now()

			mid, err := address.IDFromAddress(s.actor)
			if err != nil {
				return nil, err
			}

			postOut, ps, err := s.prover.GenerateWindowPoSt(ctx, abi.ActorID(mid), sinfos, append(abi.PoStRandomness{}, rand...))
			elapsed := time.Since(tsStart)

			log.Infow("computing window post", "batch", batchIdx, "elapsed", elapsed)

			if err == nil {
				// If we proved nothing, something is very wrong.
				if len(postOut) == 0 {
					return nil, xerrors.Errorf("received no proofs back from generate window post")
				}

				headTs, err := s.api.ChainHead(ctx)
				if err != nil {
					return nil, xerrors.Errorf("getting current head: %w", err)
				}

				checkRand, err := s.api.StateGetRandomnessFromBeacon(ctx, crypto.DomainSeparationTag_WindowedPoStChallengeSeed, di.Challenge, buf.Bytes(), headTs.Key())
				if err != nil {
					return nil, xerrors.Errorf("failed to get chain randomness from beacon for window post (ts=%d; deadline=%d): %w", ts.Height(), di, err)
				}

				if !bytes.Equal(checkRand, rand) {
					log.Warnw("windowpost randomness changed", "old", rand, "new", checkRand, "ts-height", ts.Height(), "challenge-height", di.Challenge, "tsk", ts.Key())
					rand = checkRand
					continue
				}

				// If we generated an incorrect proof, try again.
				if correct, err := s.verifier.VerifyWindowPoSt(ctx, proof.WindowPoStVerifyInfo{
					Randomness:        abi.PoStRandomness(checkRand),
					Proofs:            postOut,
					ChallengedSectors: sinfos,
					Prover:            abi.ActorID(mid),
				}); err != nil {
					log.Errorw("window post verification failed", "post", postOut, "error", err)
					time.Sleep(5 * time.Second)
					continue
				} else if !correct {
					log.Errorw("generated incorrect window post proof", "post", postOut, "error", err)
					continue
				}

				// Proof generation successful, stop retrying
				somethingToProve = true
				params.Partitions = partitions
				params.Proofs = postOut
				break
			}

			// Proof generation failed, so retry

			if len(ps) == 0 {
				// If we didn't skip any new sectors, we failed
				// for some other reason and we need to abort.
				return nil, xerrors.Errorf("running window post failed: %w", err)
			}
			// TODO: maybe mark these as faulty somewhere?

			log.Warnw("generate window post skipped sectors", "sectors", ps, "error", err, "try", retries)

			// Explicitly make sure we haven't aborted this PoSt
			// (GenerateWindowPoSt may or may not check this).
			// Otherwise, we could try to continue proving a
			// deadline after the deadline has ended.
			if ctx.Err() != nil {
				log.Warnw("aborting PoSt due to context cancellation", "error", ctx.Err(), "deadline", di.Index)
				return nil, ctx.Err()
			}

			for _, sector := range ps {
				postSkipped.Set(uint64(sector.Number))
			}
		}

		// Nothing to prove for this batch, try the next batch
		if !somethingToProve {
			continue
		}

		posts = append(posts, params)
	}

	return posts, nil
}
```

其中，s.prover.GenerateWindowPoSt，会调用到extern中的sector-storage模块，最后会调用到rust部分完成证明参数的算法。

生成的证明数据会封装到posts（SubmitWindowedPoStParams类型）里面，该结构体定义如下：

**actors/builtin/miner/miner_actor.go**

```go
type SubmitWindowedPoStParams struct {
	// The deadline index which the submission targets.
	Deadline uint64
	// The partitions being proven.
	Partitions []PoStPartition
	// Array of proofs, one per distinct registered proof type present in the sectors being proven.
	// In the usual case of a single proof type, this array will always have a single element (independent of number of partitions).
	Proofs []proof.PoStProof
	// The epoch at which these proofs is being committed to a particular chain.
	ChainCommitEpoch abi.ChainEpoch
	// The ticket randomness on the chain at the ChainCommitEpoch on the chain this post is committed to
	ChainCommitRand abi.Randomness
}
```
## 提交上链

时空证明参数生成之后，会执行submitHandler的run方法，提交上链。


**storage/wdpost_changehandler.go**

```go

func (s *submitHandler) run() {
	// On shutdown, abort in-progress submits
	defer func() {
		for _, pw := range s.postWindows {
			if pw.abort != nil {
				pw.abort()
			}
		}
	}()

	for s.shutdownCtx.Err() == nil {
		select {
		case <-s.shutdownCtx.Done():
			return

		case hc := <-s.hcs:
			// Head change
			s.processHeadChange(hc.ctx, hc.revert, hc.advance, hc.di)
			if s.processedHeadChanges != nil {
				s.processedHeadChanges <- hc
			}

		case pi := <-s.posts.added:
			// Proof generated
			s.processPostReady(pi)
			if s.processedPostReady != nil {
				s.processedPostReady <- pi
			}

		case res := <-s.submitResults:
			// Submit complete
			s.processSubmitResult(res)
			if s.processedSubmitResults != nil {
				s.processedSubmitResults <- res
			}

		case pwreq := <-s.getPostWindowReqs:
			// used by getPostWindow() to sync with run loop
			pwreq.out <- s.postWindows[pwreq.di.Open]

		case out := <-s.getTSDIReq:
			// used by currentTSDI() to sync with run loop
			out <- &tsdi{ts: s.currentTS, di: s.currentDI}
		}
	}
}

```

当posts被添加生成的时候，会执行processPostReady方法，

**storage/wdpost_changehandler.go**

```go

func (s *submitHandler) processPostReady(pi *postInfo) {
	pw, ok := s.postWindows[pi.di.Open]
	if ok {
		s.submitIfReady(s.currentCtx, s.currentTS, pw)
	}
}

```
**storage/wdpost_changehandler.go**

```go
func (s *submitHandler) submitIfReady(ctx context.Context, advance *types.TipSet, pw *postWindow) {
	// If the window has expired, there's nothing more to do.
	if advance.Height() >= pw.di.Close {
		return
	}

	// Check if we're already submitting, or already completed submit
	if pw.submitState != SubmitStateStart {
		return
	}

	// Check if we've reached the confidence height to submit
	if advance.Height() < pw.di.Open+SubmitConfidence {
		return
	}

	// Check if the proofs have been generated for this deadline
	posts, ok := s.posts.get(pw.di)
	if !ok {
		return
	}

	// If there was nothing to prove, move straight to the complete state
	if len(posts) == 0 {
		pw.submitState = SubmitStateComplete
		return
	}

	// Start submitting post
	pw.submitState = SubmitStateSubmitting
	pw.abort = s.api.startSubmitPoST(ctx, advance, pw.di, posts, func(err error) {
		s.submitResults <- &submitResult{pw: pw, err: err}
	})
}
```

s.api.startSubmitPoST,其实现在WindowPoStScheduler

**storage/wdpost_run.go**

```go

func (s *WindowPoStScheduler) startSubmitPoST(
	ctx context.Context,
	ts *types.TipSet,
	deadline *dline.Info,
	posts []miner.SubmitWindowedPoStParams,
	completeSubmitPoST CompleteSubmitPoSTCb,
) context.CancelFunc {

	ctx, abort := context.WithCancel(ctx)
	go func() {
		defer abort()

		err := s.runSubmitPoST(ctx, ts, deadline, posts)
		if err == nil {
			s.journal.RecordEvent(s.evtTypes[evtTypeWdPoStScheduler], func() interface{} {
				return WdPoStSchedulerEvt{
					evtCommon: s.getEvtCommon(nil),
					State:     SchedulerStateSucceeded,
				}
			})
		}
		completeSubmitPoST(err)
	}()

	return abort
}

```

**storage/wdpost_run.go**

```go
func (s *WindowPoStScheduler) runSubmitPoST(
	ctx context.Context,
	ts *types.TipSet,
	deadline *dline.Info,
	posts []miner.SubmitWindowedPoStParams,
) error {
	if len(posts) == 0 {
		return nil
	}

	ctx, span := trace.StartSpan(ctx, "WindowPoStScheduler.submitPoST")
	defer span.End()

	// Get randomness from tickets
	// use the challenge epoch if we've upgraded to network version 4
	// (actors version 2). We want to go back as far as possible to be safe.
	commEpoch := deadline.Open
	if ver, err := s.api.StateNetworkVersion(ctx, types.EmptyTSK); err != nil {
		log.Errorw("failed to get network version to determine PoSt epoch randomness lookback", "error", err)
	} else if ver >= network.Version4 {
		commEpoch = deadline.Challenge
	}

	commRand, err := s.api.StateGetRandomnessFromTickets(ctx, crypto.DomainSeparationTag_PoStChainCommit, commEpoch, nil, ts.Key())
	if err != nil {
		err = xerrors.Errorf("failed to get chain randomness from tickets for windowPost (ts=%d; deadline=%d): %w", ts.Height(), commEpoch, err)
		log.Errorf("submitPoStMessage failed: %+v", err)

		return err
	}

	var submitErr error
	for i := range posts {
		// Add randomness to PoST
		post := &posts[i]
		post.ChainCommitEpoch = commEpoch
		post.ChainCommitRand = commRand

		// Submit PoST
		sm, submitErr := s.submitPoStMessage(ctx, post)
		if submitErr != nil {
			log.Errorf("submit window post failed: %+v", submitErr)
		} else {
			s.recordProofsEvent(post.Partitions, sm.Cid())
		}
	}

	return submitErr
}

```


