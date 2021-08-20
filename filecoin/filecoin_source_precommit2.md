# Filecoin源码解读：PreCommit2

源码工程：rust-fil-proofs

版本号：９.０

## 简介

上一篇分析了P1的代码实现逻辑，也即DRG算法的第一部分，接下来分析P2的实现源码。

P2包含两个步骤：

- 构建Colum Hash：在SDR图上计算哈兮值，使用的哈兮算法是Poseidon(波塞冬)
- 构建Tree r last：和Colum Hash的构建过程一样，只不过参数不一样

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

## Building Colum Hash


## Building tree_r_last


参考：

[Filecoin - Precommit2计算介绍](https://blog.csdn.net/StarLi2020/article/details/116895492)