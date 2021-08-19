# Filecoin 源码解读：PreCommit1

## 简介

PreCommit1是密封过程中的核心方法，是官方 *Stacked DRG* 算法的具体实现步骤部分，这个过程会用到CPU计算生成节点和哈兮值。

Stacked DRG 算法：https://spec.filecoin.io/#section-algorithms.sdr

其代码的调用过程，在前面的文章有讲到，执行路径：lotus -> extern(storage-sealing,sector-storage,filecoin-ffi) -> rust-filecoin-proofs-api -> rust-fil-proofs。

接下来重点分析 rust 部分的实现逻辑。

## 接口部分

rust-filecoin-proofs-api包行了密封过程函数以及两个时空证明(post和winpost)的接口，也即证明子系统接口。

工程源码：https://github.com/filecoin-project/rust-filecoin-proofs-api

**seal.rs**

```rust
pub fn seal_pre_commit_phase1<R, S, T>(
    registered_proof: RegisteredSealProof,
    cache_path: R,
    in_path: S,
    out_path: T,
    prover_id: ProverId,
    sector_id: SectorId,
    ticket: Ticket,
    piece_infos: &[PieceInfo],
) -> Result<SealPreCommitPhase1Output>
where
    R: AsRef<Path>,
    S: AsRef<Path>,
    T: AsRef<Path>,
{
    ensure!(
        registered_proof.major_version() == 1,
        "unusupported version"
    );

    with_shape!(
        u64::from(registered_proof.sector_size()),
        seal_pre_commit_phase1_inner,
        registered_proof,
        cache_path.as_ref(),
        in_path.as_ref(),
        out_path.as_ref(),
        prover_id,
        sector_id,
        ticket,
        piece_infos
    )
}
```

在inner方法中，filecoin_proofs_v1::seal_pre_commit_phase1，会调用证明子系统的实现部分。

**seal.rs**

```rust
fn seal_pre_commit_phase1_inner<Tree: 'static + MerkleTreeTrait>(
    registered_proof: RegisteredSealProof,
    cache_path: &Path,
    in_path: &Path,
    out_path: &Path,
    prover_id: ProverId,
    sector_id: SectorId,
    ticket: Ticket,
    piece_infos: &[PieceInfo],
) -> Result<SealPreCommitPhase1Output> {
    let config = registered_proof.as_v1_config();

    let output = filecoin_proofs_v1::seal_pre_commit_phase1::<_, _, _, Tree>(
        config,
        cache_path,
        in_path,
        out_path,
        prover_id,
        sector_id,
        ticket,
        piece_infos,
    )?;

    let filecoin_proofs_v1::types::SealPreCommitPhase1Output::<Tree> {
        labels,
        config,
        comm_d,
    } = output;

    Ok(SealPreCommitPhase1Output {
        registered_proof,
        labels: Labels::from_raw::<Tree>(registered_proof, &labels)?,
        config,
        comm_d,
    })
}
```

## 实现crates组成

工程源码：https://github.com/filecoin-project/rust-fil-proofs

关于证明子系统(FPS)源码的crates(rust语言的代码仓库)组成：

- 证明入口库 (filecoin-proofs) ：存储证明的包装器，提供了ffi导出的api，lotus通过cgo的方式调用到。
- 时空证明库 (storage-proofs-post)：包括时空证明的实现部分。
- 复制证明库 (storage-proofs-porep) ：复制证明的实现部分。核心组建如下：
  + PoR (Proof-of-Retrievability: 默克尔包含证明)
  + DrgPoRep (深度鲁棒图复制证明)
  + StackedDrgPoRep
- 存储证明核心库　(storage-proofs-core) 

## 实现源码

上面有讲到，实现部分的入口在filecoin-proofs目录，其中P1过程在api目录下的 seal.rs

seal_pre_commit_phase1：该方法会生成labes(Labels<Tree>的数据类型)，并且返回作为P2阶段的输入，其核心逻辑如下：

- setup_params:构建启动参数，返回变量vanilla_params，其graph字段，就是构建出来的图的数据结构。
- create_base_merkle_tree：创建默克尔树，根据其树根得到comm_d
- verify_pieces：根据comm_d验证碎片
- generate_replica_id：通过comm_d生成副本id
- replicate_phase1：生成labes，作为P1的输出，也即P2的输入参数

**filecoin-proofs/src/api/seal.rs**

```rust
#[allow(clippy::too_many_arguments)]
pub fn seal_pre_commit_phase1<R, S, T, Tree: 'static + MerkleTreeTrait>(
    porep_config: PoRepConfig,
    cache_path: R,
    in_path: S,
    out_path: T,
    prover_id: ProverId,
    sector_id: SectorId,
    ticket: Ticket,
    piece_infos: &[PieceInfo],
) -> Result<SealPreCommitPhase1Output<Tree>>
where
    R: AsRef<Path>,
    S: AsRef<Path>,
    T: AsRef<Path>,
{
    info!("seal_pre_commit_phase1:start: {:?}", sector_id);

    // Sanity check all input path types.
    ensure!(
        metadata(in_path.as_ref())?.is_file(),
        "in_path must be a file"
    );
    ensure!(
        metadata(out_path.as_ref())?.is_file(),
        "out_path must be a file"
    );
    ensure!(
        metadata(cache_path.as_ref())?.is_dir(),
        "cache_path must be a directory"
    );

    let sector_bytes = usize::from(PaddedBytesAmount::from(porep_config));
    fs::metadata(&in_path)
        .with_context(|| format!("could not read in_path={:?})", in_path.as_ref().display()))?;

    fs::metadata(&out_path)
        .with_context(|| format!("could not read out_path={:?}", out_path.as_ref().display()))?;

    // Copy unsealed data to output location, where it will be sealed in place.
    fs::copy(&in_path, &out_path).with_context(|| {
        format!(
            "could not copy in_path={:?} to out_path={:?}",
            in_path.as_ref().display(),
            out_path.as_ref().display()
        )
    })?;

    let f_data = OpenOptions::new()
        .read(true)
        .write(true)
        .open(&out_path)
        .with_context(|| format!("could not open out_path={:?}", out_path.as_ref().display()))?;

    // Zero-pad the data to the requested size by extending the underlying file if needed.
    f_data.set_len(sector_bytes as u64)?;

    let data = unsafe {
        MmapOptions::new()
            .map_mut(&f_data)
            .with_context(|| format!("could not mmap out_path={:?}", out_path.as_ref().display()))?
    };

    let compound_setup_params = compound_proof::SetupParams {
        vanilla_params: setup_params(
            PaddedBytesAmount::from(porep_config),
            usize::from(PoRepProofPartitions::from(porep_config)),
            porep_config.porep_id,
            porep_config.api_version,
        )?,
        partitions: Some(usize::from(PoRepProofPartitions::from(porep_config))),
        priority: false,
    };

    let compound_public_params = <StackedCompound<Tree, DefaultPieceHasher> as CompoundProof<
        StackedDrg<'_, Tree, DefaultPieceHasher>,
        _,
    >>::setup(&compound_setup_params)?;

    info!("building merkle tree for the original data");
    let (config, comm_d) = measure_op(Operation::CommD, || -> Result<_> {
        let base_tree_size = get_base_tree_size::<DefaultBinaryTree>(porep_config.sector_size)?;
        let base_tree_leafs = get_base_tree_leafs::<DefaultBinaryTree>(base_tree_size)?;
        ensure!(
            compound_public_params.vanilla_params.graph.size() == base_tree_leafs,
            "graph size and leaf size don't match"
        );

        trace!(
            "seal phase 1: sector_size {}, base tree size {}, base tree leafs {}",
            u64::from(porep_config.sector_size),
            base_tree_size,
            base_tree_leafs,
        );

        let mut config = StoreConfig::new(
            cache_path.as_ref(),
            CacheKey::CommDTree.to_string(),
            default_rows_to_discard(base_tree_leafs, BINARY_ARITY),
        );

        let data_tree = create_base_merkle_tree::<BinaryMerkleTree<DefaultPieceHasher>>(
            Some(config.clone()),
            base_tree_leafs,
            &data,
        )?;
        drop(data);

        config.size = Some(data_tree.len());
        let comm_d_root: Fr = data_tree.root().into();
        let comm_d = commitment_from_fr(comm_d_root);

        drop(data_tree);

        Ok((config, comm_d))
    })?;

    info!("verifying pieces");

    ensure!(
        verify_pieces(&comm_d, piece_infos, porep_config.into())?,
        "pieces and comm_d do not match"
    );

    let replica_id = generate_replica_id::<Tree::Hasher, _>(
        &prover_id,
        sector_id.into(),
        &ticket,
        comm_d,
        &porep_config.porep_id,
    );

    let labels = StackedDrg::<Tree, DefaultPieceHasher>::replicate_phase1(
        &compound_public_params.vanilla_params,
        &replica_id,
        config.clone(),
    )?;

    let out = SealPreCommitPhase1Output {
        labels,
        config,
        comm_d,
    };

    info!("seal_pre_commit_phase1:finish: {:?}", sector_id);
    Ok(out)
}
```

setup_params方法如下：

- nodes：表示SDR图中的节点，SDR共有11层，每一层的节点数量相当于１GiB的字节数量。
- degree:大小是６，当前层中抽查的节点数量
- expansion_degree：大小是８，上一层中抽取的节点数量，用来计算当前层的节点数据

**filecoin-proofs/src/parameters.rs**

```rust
pub fn setup_params(
    sector_bytes: PaddedBytesAmount,
    partitions: usize,
    porep_id: [u8; 32],
    api_version: ApiVersion,
) -> Result<stacked::SetupParams> {
    let layer_challenges = select_challenges(
        partitions,
        *POREP_MINIMUM_CHALLENGES
            .read()
            .expect("POREP_MINIMUM_CHALLENGES poisoned")
            .get(&u64::from(sector_bytes))
            .expect("unknown sector size") as usize,
        *LAYERS
            .read()
            .expect("LAYERS poisoned")
            .get(&u64::from(sector_bytes))
            .expect("unknown sector size"),
    );
    let sector_bytes = u64::from(sector_bytes);

    ensure!(
        sector_bytes % 32 == 0,
        "sector_bytes ({}) must be a multiple of 32",
        sector_bytes,
    );

    let nodes = (sector_bytes / 32) as usize;
    let degree = DRG_DEGREE;
    let expansion_degree = EXP_DEGREE;

    Ok(stacked::SetupParams {
        nodes,
        degree,
        expansion_degree,
        porep_id,
        layer_challenges,
        api_version,
    })
}
```

接下来分析StackedDrg的replicate_phase1方法

**storage-proofs-porep/src/stacked/vanilla/proof.rs**

```rust
 pub fn replicate_phase1(
        pp: &'a PublicParams<Tree>,
        replica_id: &<Tree::Hasher as Hasher>::Domain,
        config: StoreConfig,
    ) -> Result<Labels<Tree>> {
        info!("replicate_phase1");

        let labels = measure_op(Operation::EncodeWindowTimeAll, || {
            Self::generate_labels_for_encoding(&pp.graph, &pp.layer_challenges, replica_id, config)
        })?
        .0;

        Ok(labels)
    }
```

在generate_labels_for_encoding方法中：根据是否启用SDR，调用不同的逻辑

注意：在开启SDR后，会使用多核CPU生成labels，可以加快P1的执行速度。

**storage-proofs-porep/src/stacked/vanilla/proof.rs**

```rust
pub fn generate_labels_for_encoding(
        graph: &StackedBucketGraph<Tree::Hasher>,
        layer_challenges: &LayerChallenges,
        replica_id: &<Tree::Hasher as Hasher>::Domain,
        config: StoreConfig,
    ) -> Result<(Labels<Tree>, Vec<LayerState>)> {
        let mut parent_cache = graph.parent_cache()?;

        #[cfg(feature = "multicore-sdr")]
        {
            if SETTINGS.use_multicore_sdr {
                info!("multi core replication");
                create_label::multi::create_labels_for_encoding(
                    graph,
                    &parent_cache,
                    layer_challenges.layers(),
                    replica_id,
                    config,
                )
            } else {
                info!("single core replication");
                create_label::single::create_labels_for_encoding(
                    graph,
                    &mut parent_cache,
                    layer_challenges.layers(),
                    replica_id,
                    config,
                )
            }
        }

        #[cfg(not(feature = "multicore-sdr"))]
        {
            info!("single core replication");
            create_label::single::create_labels_for_encoding(
                graph,
                &mut parent_cache,
                layer_challenges.layers(),
                replica_id,
                config,
            )
        }
    }
```

如果开启SDR特性，代码执行如下：

遍历所有的layers，然后一层一层创建label，注意第一层没有依赖上一层。

**storage-proofs-porep/src/stacked/vanilla/create_label/multi.rs**

```rust
#[allow(clippy::type_complexity)]
pub fn create_labels_for_encoding<Tree: 'static + MerkleTreeTrait, T: AsRef<[u8]>>(
    graph: &StackedBucketGraph<Tree::Hasher>,
    parents_cache: &ParentCache,
    layers: usize,
    replica_id: T,
    config: StoreConfig,
) -> Result<(Labels<Tree>, Vec<LayerState>)> {
    info!("create labels");

    let layer_states = prepare_layers::<Tree>(graph, &config, layers);

    let sector_size = graph.size() * NODE_SIZE;
    let node_count = graph.size() as u64;
    let cache_window_nodes = SETTINGS.sdr_parents_cache_size as usize;

    let default_cache_size = DEGREE * 4 * cache_window_nodes;

    let core_group = Arc::new(checkout_core_group());

    // When `_cleanup_handle` is dropped, the previous binding of thread will be restored.
    let _cleanup_handle = (*core_group).as_ref().map(|group| {
        // This could fail, but we will ignore the error if so.
        // It will be logged as a warning by `bind_core`.
        debug!("binding core in main thread");
        group.get(0).map(|core_index| bind_core(*core_index))
    });

    // NOTE: this means we currently keep 2x sector size around, to improve speed
    let (parents_cache, mut layer_labels, mut exp_labels) = setup_create_label_memory(
        sector_size,
        DEGREE,
        Some(default_cache_size as usize),
        &parents_cache.path,
    )?;

    for (layer, layer_state) in (1..=layers).zip(layer_states.iter()) {
        info!("Layer {}", layer);

        if layer_state.generated {
            info!("skipping layer {}, already generated", layer);

            // load the already generated layer into exp_labels
            read_layer(&layer_state.config, &mut exp_labels)?;
            continue;
        }

        // Cache reset happens in two parts.
        // The second part (the finish) happens before each layer but the first.
        if layers != 1 {
            parents_cache.finish_reset()?;
        }

        create_layer_labels(
            &parents_cache,
            &replica_id.as_ref(),
            &mut layer_labels,
            if layer == 1 {
                None
            } else {
                Some(&mut exp_labels)
            },
            node_count,
            layer as u32,
            core_group.clone(),
        );

        // Cache reset happens in two parts.
        // The first part (the start) happens after each layer but the last.
        if layer != layers {
            parents_cache.start_reset()?;
        }

        mem::swap(&mut layer_labels, &mut exp_labels);
        {
            let layer_config = &layer_state.config;

            info!("  storing labels on disk");
            write_layer(&exp_labels, layer_config).context("failed to store labels")?;

            info!(
                "  generated layer {} store with id {}",
                layer, layer_config.id
            );
        }
    }

    Ok((
        Labels::<Tree> {
            labels: layer_states.iter().map(|s| s.config.clone()).collect(),
            _h: PhantomData,
        },
        layer_states,
    ))
}
```

create_layer_labels方法：创建当前层的labels，每一层的节点是１GiB。

**storage-proofs-porep/src/stacked/vanilla/create_label/multi.rs**

```rust
fn create_layer_labels(
    parents_cache: &CacheReader<u32>,
    replica_id: &[u8],
    layer_labels: &mut MmapMut,
    exp_labels: Option<&mut MmapMut>,
    num_nodes: u64,
    cur_layer: u32,
    core_group: Arc<Option<MutexGuard<'_, Vec<CoreIndex>>>>,
) {
    info!("Creating labels for layer {}", cur_layer);
    // num_producers is the number of producer threads
    let (lookahead, num_producers, producer_stride) = {
        let settings = &SETTINGS;
        let lookahead = settings.multicore_sdr_lookahead;
        let num_producers = settings.multicore_sdr_producers;
        // NOTE: Stride must not exceed the number of nodes in parents_cache's window. If it does, the process will deadlock
        // with producers and consumers waiting for each other.
        let producer_stride = settings
            .multicore_sdr_producer_stride
            .min(parents_cache.window_nodes() as u64);

        (lookahead, num_producers, producer_stride)
    };

    const BYTES_PER_NODE: usize = (NODE_SIZE * DEGREE) + SHA_BLOCK_SIZE;

    let mut ring_buf = RingBuf::new(BYTES_PER_NODE, lookahead);
    let mut base_parent_missing = vec![BitMask::default(); lookahead];

    // Fill in the fixed portion of all buffers
    for buf in ring_buf.iter_slot_mut() {
        prepare_block(replica_id, cur_layer, buf);
    }

    // Highest node that is ready from the producer
    let cur_producer = AtomicU64::new(0);
    // Next node to be filled
    let cur_awaiting = AtomicU64::new(1);

    // These UnsafeSlices are managed through the 3 Atomics above, to minimize any locking overhead.
    let layer_labels = UnsafeSlice::from_slice(
        layer_labels
            .as_mut_slice_of::<u32>()
            .expect("failed as mut slice of"),
    );
    let exp_labels = exp_labels.map(|m| {
        UnsafeSlice::from_slice(m.as_mut_slice_of::<u32>().expect("failed as mut slice of"))
    });
    let base_parent_missing = UnsafeSlice::from_slice(&mut base_parent_missing);

    crossbeam::thread::scope(|s| {
        let mut runners = Vec::with_capacity(num_producers);

        for i in 0..num_producers {
            let layer_labels = &layer_labels;
            let exp_labels = exp_labels.as_ref();
            let cur_producer = &cur_producer;
            let cur_awaiting = &cur_awaiting;
            let ring_buf = &ring_buf;
            let base_parent_missing = &base_parent_missing;

            let core_index = if let Some(cg) = &*core_group {
                cg.get(i + 1)
            } else {
                None
            };
            runners.push(s.spawn(move |_| {
                // This could fail, but we will ignore the error if so.
                // It will be logged as a warning by `bind_core`.
                debug!("binding core in producer thread {}", i);
                // When `_cleanup_handle` is dropped, the previous binding of thread will be restored.
                let _cleanup_handle = core_index.map(|c| bind_core(*c));

                create_label_runner(
                    parents_cache,
                    layer_labels,
                    exp_labels,
                    num_nodes,
                    cur_producer,
                    cur_awaiting,
                    producer_stride,
                    lookahead as u64,
                    ring_buf,
                    base_parent_missing,
                )
            }));
        }

        let mut cur_node_ptr = unsafe { layer_labels.as_mut_slice() };
        let mut cur_parent_ptr = unsafe { parents_cache.consumer_slice_at(DEGREE) };
        let mut cur_parent_ptr_offset = DEGREE;

        // Calculate node 0 (special case with no parents)
        // Which is replica_id || cur_layer || 0
        // TODO - Hash and save intermediate result: replica_id || cur_layer
        let mut buf = [0u8; (NODE_SIZE * DEGREE) + 64];
        prepare_block(replica_id, cur_layer, &mut buf);

        cur_node_ptr[..8].copy_from_slice(&SHA256_INITIAL_DIGEST);
        compress256!(cur_node_ptr, buf, 2);

        // Fix endianess
        cur_node_ptr[..8].iter_mut().for_each(|x| *x = x.to_be());

        cur_node_ptr[7] &= 0x3FFF_FFFF; // Strip last two bits to ensure in Fr

        // Keep track of which node slot in the ring_buffer to use
        let mut cur_slot = 0;
        let mut count_not_ready = 0;

        // Calculate nodes 1 to n

        // Skip first node.
        parents_cache.store_consumer(1);
        let mut i = 1;
        while i < num_nodes {
            // Ensure next buffer is ready
            let mut counted = false;
            let mut producer_val = cur_producer.load(SeqCst);

            while producer_val < i {
                if !counted {
                    counted = true;
                    count_not_ready += 1;
                }
                thread::sleep(Duration::from_micros(10));
                producer_val = cur_producer.load(SeqCst);
            }

            // Process as many nodes as are ready
            let ready_count = producer_val - i + 1;
            for _count in 0..ready_count {
                // If we have used up the last cache window's parent data, get some more.
                if cur_parent_ptr.is_empty() {
                    // Safety: values read from `cur_parent_ptr` before calling `increment_consumer`
                    // must not be read again after.
                    unsafe {
                        cur_parent_ptr = parents_cache.consumer_slice_at(cur_parent_ptr_offset);
                    }
                }

                cur_node_ptr = &mut cur_node_ptr[8..];
                // Grab the current slot of the ring_buf
                let buf = unsafe { ring_buf.slot_mut(cur_slot) };
                // Fill in the base parents
                for k in 0..BASE_DEGREE {
                    let bpm = unsafe { base_parent_missing.get(cur_slot) };
                    if bpm.get(k) {
                        let source = unsafe {
                            let start = cur_parent_ptr[0] as usize * NODE_WORDS;
                            let end = start + NODE_WORDS;
                            &layer_labels.as_slice()[start..end]
                        };

                        buf[64 + (NODE_SIZE * k)..64 + (NODE_SIZE * (k + 1))]
                            .copy_from_slice(source.as_byte_slice());
                    }
                    cur_parent_ptr = &cur_parent_ptr[1..];
                    cur_parent_ptr_offset += 1;
                }

                // Expanders are already all filled in (layer 1 doesn't use expanders)
                cur_parent_ptr = &cur_parent_ptr[EXP_DEGREE..];
                cur_parent_ptr_offset += EXP_DEGREE;

                if cur_layer == 1 {
                    // Six rounds of all base parents
                    for _j in 0..6 {
                        compress256!(cur_node_ptr, &buf[64..], 3);
                    }

                    // round 7 is only first parent
                    memset(&mut buf[96..128], 0); // Zero out upper half of last block
                    buf[96] = 0x80; // Padding
                    buf[126] = 0x27; // Length (0x2700 = 9984 bits -> 1248 bytes)
                    compress256!(cur_node_ptr, &buf[64..], 1);
                } else {
                    // Two rounds of all parents
                    let blocks = [
                        *GenericArray::<u8, U64>::from_slice(&buf[64..128]),
                        *GenericArray::<u8, U64>::from_slice(&buf[128..192]),
                        *GenericArray::<u8, U64>::from_slice(&buf[192..256]),
                        *GenericArray::<u8, U64>::from_slice(&buf[256..320]),
                        *GenericArray::<u8, U64>::from_slice(&buf[320..384]),
                        *GenericArray::<u8, U64>::from_slice(&buf[384..448]),
                        *GenericArray::<u8, U64>::from_slice(&buf[448..512]),
                    ];
                    sha2::compress256(
                        (&mut cur_node_ptr[..8])
                            .try_into()
                            .expect("compress failed"),
                        &blocks,
                    );
                    sha2::compress256(
                        (&mut cur_node_ptr[..8])
                            .try_into()
                            .expect("compress failed"),
                        &blocks,
                    );

                    // Final round is only nine parents
                    memset(&mut buf[352..384], 0); // Zero out upper half of last block
                    buf[352] = 0x80; // Padding
                    buf[382] = 0x27; // Length (0x2700 = 9984 bits -> 1248 bytes)
                    compress256!(cur_node_ptr, &buf[64..], 5);
                }

                // Fix endianess
                cur_node_ptr[..8].iter_mut().for_each(|x| *x = x.to_be());

                cur_node_ptr[7] &= 0x3FFF_FFFF; // Strip last two bits to fit in Fr

                // Safety:
                // It's possible that this increment will trigger moving the cache window.
                // In that case, we must not access `parents_cache` again but instead replace it.
                // This will happen above because `parents_cache` will now be empty, if we have
                // correctly advanced it so far.
                unsafe {
                    parents_cache.increment_consumer();
                }
                i += 1;
                cur_slot = (cur_slot + 1) % lookahead;
            }
        }

        debug!("PRODUCER NOT READY: {} times", count_not_ready);

        for runner in runners {
            runner.join().expect("join failed");
        }
    })
    .expect("crossbeam scope failure");
}
```

参考：

[密封（Seal）流程简介](https://github.com/filecoin-project/community-china/blob/master/documents/tutorial/lotus_seal_process/seal_process.md)

[Filecoin - 深入理解SDR算法](https://blog.csdn.net/StarLi2020/article/details/107576768)

[Filecoin 扇区生命周期及状态管理](https://github.com/filecoin-project/community-china/discussions/14)

[零知识证明 - Filecoin存储证明了什么？](https://blog.csdn.net/StarLi2020/article/details/104567316)

[Filecoin - PoRep和PoSt算法源代码导读](https://blog.csdn.net/StarLi2020/article/details/116895231)


