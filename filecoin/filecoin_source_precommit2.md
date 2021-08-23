# Filecoin源码解读：PreCommit2

源码工程：rust-fil-proofs

版本号：９.０

## 简介

上一篇分析了P1的代码实现逻辑，也即DRG算法的第一部分，接下来分析P2的实现源码。

P2阶段包含两个步骤：

**generating tree c**： 构建Colum Hash，在SDR图上计算哈兮值，使用的哈兮算法是Poseidon(波塞冬)

**generating tree r last**：构建Tree r last，和Colum Hash的构建过程一样，只不过参数不一样


seal_pre_commit_phase2执行逻辑：

- data_tree : 从硬盘里面加载数据
- replicate_phase2：　调用StackedDrg的replicate_phase2方法

其中，P2 返回的参数comm_r和comm_d会用于之后的C1,C2阶段。


**filecoin-proofs/src/api/seal.rs**

```rust
#[allow(clippy::too_many_arguments)]
pub fn seal_pre_commit_phase2<R, S, Tree: 'static + MerkleTreeTrait>(
    porep_config: PoRepConfig,
    phase1_output: SealPreCommitPhase1Output<Tree>,
    cache_path: S,
    replica_path: R,
) -> Result<SealPreCommitOutput>
where
    R: AsRef<Path>,
    S: AsRef<Path>,
{
    info!("seal_pre_commit_phase2:start");

    // Sanity check all input path types.
    ensure!(
        metadata(cache_path.as_ref())?.is_dir(),
        "cache_path must be a directory"
    );
    ensure!(
        metadata(replica_path.as_ref())?.is_file(),
        "replica_path must be a file"
    );

    let SealPreCommitPhase1Output {
        mut labels,
        mut config,
        comm_d,
        ..
    } = phase1_output;

    labels.update_root(cache_path.as_ref());
    config.path = cache_path.as_ref().into();

    let f_data = OpenOptions::new()
        .read(true)
        .write(true)
        .open(&replica_path)
        .with_context(|| {
            format!(
                "could not open replica_path={:?}",
                replica_path.as_ref().display()
            )
        })?;
    let data = unsafe {
        MmapOptions::new().map_mut(&f_data).with_context(|| {
            format!(
                "could not mmap replica_path={:?}",
                replica_path.as_ref().display()
            )
        })?
    };
    let data: Data<'_> = (data, PathBuf::from(replica_path.as_ref())).into();

    // Load data tree from disk
    let data_tree = {
        let base_tree_size = get_base_tree_size::<DefaultBinaryTree>(porep_config.sector_size)?;
        let base_tree_leafs = get_base_tree_leafs::<DefaultBinaryTree>(base_tree_size)?;

        trace!(
            "seal phase 2: base tree size {}, base tree leafs {}, rows to discard {}",
            base_tree_size,
            base_tree_leafs,
            default_rows_to_discard(base_tree_leafs, BINARY_ARITY)
        );
        ensure!(
            config.rows_to_discard == default_rows_to_discard(base_tree_leafs, BINARY_ARITY),
            "Invalid cache size specified"
        );

        let store: DiskStore<DefaultPieceDomain> =
            DiskStore::new_from_disk(base_tree_size, BINARY_ARITY, &config)?;
        BinaryMerkleTree::<DefaultPieceHasher>::from_data_store(store, base_tree_leafs)?
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

    let (tau, (p_aux, t_aux)) = StackedDrg::<Tree, DefaultPieceHasher>::replicate_phase2(
        &compound_public_params.vanilla_params,
        labels,
        data,
        data_tree,
        config,
        replica_path.as_ref().to_path_buf(),
    )?;

    let comm_r = commitment_from_fr(tau.comm_r.into());

    // Persist p_aux and t_aux here
    let p_aux_path = cache_path.as_ref().join(CacheKey::PAux.to_string());
    let mut f_p_aux = File::create(&p_aux_path)
        .with_context(|| format!("could not create file p_aux={:?}", p_aux_path))?;
    let p_aux_bytes = serialize(&p_aux)?;
    f_p_aux
        .write_all(&p_aux_bytes)
        .with_context(|| format!("could not write to file p_aux={:?}", p_aux_path))?;

    let t_aux_path = cache_path.as_ref().join(CacheKey::TAux.to_string());
    let mut f_t_aux = File::create(&t_aux_path)
        .with_context(|| format!("could not create file t_aux={:?}", t_aux_path))?;
    let t_aux_bytes = serialize(&t_aux)?;
    f_t_aux
        .write_all(&t_aux_bytes)
        .with_context(|| format!("could not write to file t_aux={:?}", t_aux_path))?;

    let out = SealPreCommitOutput { comm_r, comm_d };

    info!("seal_pre_commit_phase2:finish");
    Ok(out)
}
```

接下来，看proof里面的replicate_phase2方法，

**storage-proofs-porep/src/stacked/vanilla/proof.rs**

```rust
 #[allow(clippy::type_complexity)]
    pub fn replicate_phase2(
        pp: &'a PublicParams<Tree>,
        labels: Labels<Tree>,
        data: Data<'a>,
        data_tree: BinaryMerkleTree<G>,
        config: StoreConfig,
        replica_path: PathBuf,
    ) -> Result<(
        <Self as PoRep<'a, Tree::Hasher, G>>::Tau,
        <Self as PoRep<'a, Tree::Hasher, G>>::ProverAux,
    )> {
        info!("replicate_phase2");

        let (tau, paux, taux) = Self::transform_and_replicate_layers_inner(
            &pp.graph,
            &pp.layer_challenges,
            data,
            Some(data_tree),
            config,
            replica_path,
            labels,
        )?;

        Ok((tau, (paux, taux)))
    }
```

transform_and_replicate_layers_inner方法：

- 生成tree_c：调用方法generate_tree_c
- 生成tree_r_last：调用方法generate_tree_r_last

**storage-proofs-porep/src/stacked/vanilla/proof.rs**

```rust
pub(crate) fn transform_and_replicate_layers_inner(
        graph: &StackedBucketGraph<Tree::Hasher>,
        layer_challenges: &LayerChallenges,
        mut data: Data<'_>,
        data_tree: Option<BinaryMerkleTree<G>>,
        config: StoreConfig,
        replica_path: PathBuf,
        label_configs: Labels<Tree>,
    ) -> Result<TransformedLayers<Tree, G>> {
        trace!("transform_and_replicate_layers");
        let nodes_count = graph.size();

        assert_eq!(data.len(), nodes_count * NODE_SIZE);
        trace!("nodes count {}, data len {}", nodes_count, data.len());

        let tree_count = get_base_tree_count::<Tree>();
        let nodes_count = graph.size() / tree_count;

        // Ensure that the node count will work for binary and oct arities.
        let binary_arity_valid = is_merkle_tree_size_valid(nodes_count, BINARY_ARITY);
        let other_arity_valid = is_merkle_tree_size_valid(nodes_count, Tree::Arity::to_usize());
        trace!(
            "is_merkle_tree_size_valid({}, BINARY_ARITY) = {}",
            nodes_count,
            binary_arity_valid
        );
        trace!(
            "is_merkle_tree_size_valid({}, {}) = {}",
            nodes_count,
            Tree::Arity::to_usize(),
            other_arity_valid
        );
        assert!(binary_arity_valid);
        assert!(other_arity_valid);

        let layers = layer_challenges.layers();
        assert!(layers > 0);

        // Generate all store configs that we need based on the
        // cache_path in the specified config.
        let mut tree_d_config = StoreConfig::from_config(
            &config,
            CacheKey::CommDTree.to_string(),
            Some(get_merkle_tree_len(nodes_count, BINARY_ARITY)?),
        );
        tree_d_config.rows_to_discard = default_rows_to_discard(nodes_count, BINARY_ARITY);

        let mut tree_r_last_config = StoreConfig::from_config(
            &config,
            CacheKey::CommRLastTree.to_string(),
            Some(get_merkle_tree_len(nodes_count, Tree::Arity::to_usize())?),
        );

        // A default 'rows_to_discard' value will be chosen for tree_r_last, unless the user overrides this value via the
        // environment setting (FIL_PROOFS_ROWS_TO_DISCARD).  If this value is specified, no checking is done on it and it may
        // result in a broken configuration.  Use with caution.  It must be noted that if/when this unchecked value is passed
        // through merkle_light, merkle_light now does a check that does not allow us to discard more rows than is possible
        // to discard.
        tree_r_last_config.rows_to_discard =
            default_rows_to_discard(nodes_count, Tree::Arity::to_usize());
        trace!(
            "tree_r_last using rows_to_discard={}",
            tree_r_last_config.rows_to_discard
        );

        let mut tree_c_config = StoreConfig::from_config(
            &config,
            CacheKey::CommCTree.to_string(),
            Some(get_merkle_tree_len(nodes_count, Tree::Arity::to_usize())?),
        );
        tree_c_config.rows_to_discard =
            default_rows_to_discard(nodes_count, Tree::Arity::to_usize());

        let labels =
            LabelsCache::<Tree>::new(&label_configs).context("failed to create labels cache")?;
        let configs = split_config(tree_c_config.clone(), tree_count)?;

        match raise_fd_limit() {
            Some(res) => {
                info!("Building trees [{} descriptors max available]", res);
            }
            None => error!("Failed to raise the fd limit"),
        };

        let tree_c_root = match layers {
            2 => {
                let tree_c = Self::generate_tree_c::<U2, Tree::Arity>(
                    layers,
                    nodes_count,
                    tree_count,
                    configs,
                    &labels,
                )?;
                tree_c.root()
            }
            8 => {
                let tree_c = Self::generate_tree_c::<U8, Tree::Arity>(
                    layers,
                    nodes_count,
                    tree_count,
                    configs,
                    &labels,
                )?;
                tree_c.root()
            }
            11 => {
                let tree_c = Self::generate_tree_c::<U11, Tree::Arity>(
                    layers,
                    nodes_count,
                    tree_count,
                    configs,
                    &labels,
                )?;
                tree_c.root()
            }
            _ => panic_any("Unsupported column arity"),
        };
        info!("tree_c done");

        // Build the MerkleTree over the original data (if needed).
        let tree_d = match data_tree {
            Some(t) => {
                trace!("using existing original data merkle tree");
                assert_eq!(t.len(), 2 * (data.len() / NODE_SIZE) - 1);

                t
            }
            None => {
                trace!("building merkle tree for the original data");
                data.ensure_data()?;
                measure_op(Operation::CommD, || {
                    Self::build_binary_tree::<G>(data.as_ref(), tree_d_config.clone())
                })?
            }
        };
        tree_d_config.size = Some(tree_d.len());
        assert_eq!(
            tree_d_config.size.expect("config size failure"),
            tree_d.len()
        );
        let tree_d_root = tree_d.root();
        drop(tree_d);

        // Encode original data into the last layer.
        info!("building tree_r_last");
        let tree_r_last = measure_op(Operation::GenerateTreeRLast, || {
            Self::generate_tree_r_last::<Tree::Arity>(
                &mut data,
                nodes_count,
                tree_count,
                tree_r_last_config.clone(),
                replica_path.clone(),
                &labels,
            )
            .context("failed to generate tree_r_last")
        })?;
        info!("tree_r_last done");

        let tree_r_last_root = tree_r_last.root();
        drop(tree_r_last);

        data.drop_data();

        // comm_r = H(comm_c || comm_r_last)
        let comm_r: <Tree::Hasher as Hasher>::Domain =
            <Tree::Hasher as Hasher>::Function::hash2(&tree_c_root, &tree_r_last_root);

        Ok((
            Tau {
                comm_d: tree_d_root,
                comm_r,
            },
            PersistentAux {
                comm_c: tree_c_root,
                comm_r_last: tree_r_last_root,
            },
            TemporaryAux {
                labels: label_configs,
                tree_d_config,
                tree_r_last_config,
                tree_c_config,
                _g: PhantomData,
            },
        ))
    }
```

## Building tree_c

构建tree_c的方法，也在proof里面，在该方法里面，有CPU和GPU的方式来生成哈希值。

**storage-proofs-porep/src/stacked/vanilla/proof.rs**

```rust
#[cfg(any(feature = "gpu"))]
    fn generate_tree_c<ColumnArity, TreeArity>(
        layers: usize,
        nodes_count: usize,
        tree_count: usize,
        configs: Vec<StoreConfig>,
        labels: &LabelsCache<Tree>,
    ) -> Result<DiskTree<Tree::Hasher, Tree::Arity, Tree::SubTreeArity, Tree::TopTreeArity>>
    where
        ColumnArity: 'static + PoseidonArity,
        TreeArity: PoseidonArity,
    {
        if SETTINGS.use_gpu_column_builder {
            Self::generate_tree_c_gpu::<ColumnArity, TreeArity>(
                layers,
                nodes_count,
                tree_count,
                configs,
                labels,
            )
        } else {
            Self::generate_tree_c_cpu::<ColumnArity, TreeArity>(
                layers,
                nodes_count,
                tree_count,
                configs,
                labels,
            )
        }
    }
```


## Building tree_r_last

在generate_tree_r_last方法里面，也可以看到可以使用GPU加速。

**storage-proofs-porep/src/stacked/vanilla/proof.rs**

```rust
  #[cfg(any(feature = "gpu"))]
    fn generate_tree_r_last<TreeArity>(
        data: &mut Data<'_>,
        nodes_count: usize,
        tree_count: usize,
        tree_r_last_config: StoreConfig,
        replica_path: PathBuf,
        labels: &LabelsCache<Tree>,
    ) -> Result<LCTree<Tree::Hasher, Tree::Arity, Tree::SubTreeArity, Tree::TopTreeArity>>
    where
        TreeArity: PoseidonArity,
    {
        if SETTINGS.use_gpu_tree_builder {
            Self::generate_tree_r_last_gpu::<TreeArity>(
                data,
                nodes_count,
                tree_count,
                tree_r_last_config,
                replica_path,
                labels,
            )
        } else {
            Self::generate_tree_r_last_cpu::<TreeArity>(
                data,
                nodes_count,
                tree_count,
                tree_r_last_config,
                replica_path,
                labels,
            )
        }
    }
```


参考：

[Filecoin - Precommit2计算介绍](https://blog.csdn.net/StarLi2020/article/details/116895492)