# Filecoin 源码解读：Commit2

源码工程：rust-fil-proofs

版本号：９.０

Commit2是密封过程中比较耗时的方法，该方法主要做存储证明（PoRep）过程。

入口函数核心逻辑：

- 获取P1返回的参数：包括comm_r，comm_d等。
- circuit_proofs:启动零知识证明，调用电路证明方法
- verify_seal：验证密封

**filecoin-proofs/src/api/seal.rs**

```rust
#[allow(clippy::too_many_arguments)]
pub fn seal_commit_phase2<Tree: 'static + MerkleTreeTrait>(
    porep_config: PoRepConfig,
    phase1_output: SealCommitPhase1Output<Tree>,
    prover_id: ProverId,
    sector_id: SectorId,
) -> Result<SealCommitOutput> {
    info!("seal_commit_phase2:start: {:?}", sector_id);

    let SealCommitPhase1Output {
        vanilla_proofs,
        comm_d,
        comm_r,
        replica_id,
        seed,
        ticket,
    } = phase1_output;

    ensure!(comm_d != [0; 32], "Invalid all zero commitment (comm_d)");
    ensure!(comm_r != [0; 32], "Invalid all zero commitment (comm_r)");

    let comm_r_safe = as_safe_commitment(&comm_r, "comm_r")?;
    let comm_d_safe = DefaultPieceDomain::try_from_bytes(&comm_d)?;

    let public_inputs = stacked::PublicInputs {
        replica_id,
        tau: Some(stacked::Tau {
            comm_d: comm_d_safe,
            comm_r: comm_r_safe,
        }),
        k: None,
        seed,
    };

    let groth_params = get_stacked_params::<Tree>(porep_config)?;

    info!(
        "got groth params ({}) while sealing",
        u64::from(PaddedBytesAmount::from(porep_config))
    );

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

    info!("snark_proof:start");
    let groth_proofs = StackedCompound::<Tree, DefaultPieceHasher>::circuit_proofs(
        &public_inputs,
        vanilla_proofs,
        &compound_public_params.vanilla_params,
        &groth_params,
        compound_public_params.priority,
    )?;
    info!("snark_proof:finish");

    let proof = MultiProof::new(groth_proofs, &groth_params.pvk);

    let mut buf = Vec::with_capacity(
        SINGLE_PARTITION_PROOF_LEN * usize::from(PoRepProofPartitions::from(porep_config)),
    );

    proof.write(&mut buf)?;

    // Verification is cheap when parameters are cached,
    // and it is never correct to return a proof which does not verify.
    verify_seal::<Tree>(
        porep_config,
        comm_r,
        comm_d,
        prover_id,
        sector_id,
        ticket,
        seed,
        &buf,
    )
    .context("post-seal verification sanity check failed")?;

    let out = SealCommitOutput { proof: buf };

    info!("seal_commit_phase2:finish: {:?}", sector_id);
    Ok(out)
}
```

- 关于电路证明方法

通过输入参赛，创建和合成电路，然后生成并且返回 *groth proof*，其中，create_random_proof_batch方法会调用bellperson库。

bellperson地址：https://github.com/filecoin-project/bellperson

*bellman* 是用来建造zk-SNARK电路的crate(rust仓库)。它提供了电路特性和基本结构，以及基本的小工具实现，如布尔值和数字抽象。
  
**storage-proofs-core/src/compound_proof.rs**

```rust
 fn circuit_proofs(
        pub_in: &S::PublicInputs,
        vanilla_proofs: Vec<S::Proof>,
        pub_params: &S::PublicParams,
        groth_params: &groth16::MappedParameters<Bls12>,
        priority: bool,
    ) -> Result<Vec<groth16::Proof<Bls12>>> {
        let mut rng = OsRng;
        ensure!(
            !vanilla_proofs.is_empty(),
            "cannot create a circuit proof over missing vanilla proofs"
        );

        let circuits = vanilla_proofs
            .into_par_iter()
            .enumerate()
            .map(|(k, vanilla_proof)| {
                Self::circuit(
                    &pub_in,
                    C::ComponentPrivateInputs::default(),
                    &vanilla_proof,
                    &pub_params,
                    Some(k),
                )
            })
            .collect::<Result<Vec<_>>>()?;

        let groth_proofs = if priority {
            create_random_proof_batch_in_priority(circuits, groth_params, &mut rng)?
        } else {
            create_random_proof_batch(circuits, groth_params, &mut rng)?
        };

        groth_proofs
            .into_iter()
            .map(|groth_proof| {
                let mut proof_vec = Vec::new();
                groth_proof.write(&mut proof_vec)?;
                let gp = groth16::Proof::<Bls12>::read(&proof_vec[..])?;
                Ok(gp)
            })
            .collect()
    }
```

- 关于验证密封方法

验证一些以前运行过的密封操作的输出。

**filecoin-proofs/src/api/seal.rs**

```rust
pub fn verify_seal<Tree: 'static + MerkleTreeTrait>(
    porep_config: PoRepConfig,
    comm_r_in: Commitment,
    comm_d_in: Commitment,
    prover_id: ProverId,
    sector_id: SectorId,
    ticket: Ticket,
    seed: Ticket,
    proof_vec: &[u8],
) -> Result<bool> {
    info!("verify_seal:start: {:?}", sector_id);
    ensure!(comm_d_in != [0; 32], "Invalid all zero commitment (comm_d)");
    ensure!(comm_r_in != [0; 32], "Invalid all zero commitment (comm_r)");

    let comm_r: <Tree::Hasher as Hasher>::Domain = as_safe_commitment(&comm_r_in, "comm_r")?;
    let comm_d: DefaultPieceDomain = as_safe_commitment(&comm_d_in, "comm_d")?;

    let replica_id = generate_replica_id::<Tree::Hasher, _>(
        &prover_id,
        sector_id.into(),
        &ticket,
        comm_d,
        &porep_config.porep_id,
    );

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

    let compound_public_params: compound_proof::PublicParams<
        '_,
        StackedDrg<'_, Tree, DefaultPieceHasher>,
    > = StackedCompound::setup(&compound_setup_params)?;

    let public_inputs =
        stacked::PublicInputs::<<Tree::Hasher as Hasher>::Domain, DefaultPieceDomain> {
            replica_id,
            tau: Some(Tau { comm_r, comm_d }),
            seed,
            k: None,
        };

    let result = {
        let sector_bytes = PaddedBytesAmount::from(porep_config);
        let verifying_key = get_stacked_verifying_key::<Tree>(porep_config)?;

        info!(
            "got verifying key ({}) while verifying seal",
            u64::from(sector_bytes)
        );

        let proof = MultiProof::new_from_reader(
            Some(usize::from(PoRepProofPartitions::from(porep_config))),
            proof_vec,
            &verifying_key,
        )?;

        StackedCompound::verify(
            &compound_public_params,
            &public_inputs,
            &proof,
            &ChallengeRequirements {
                minimum_challenges: *POREP_MINIMUM_CHALLENGES
                    .read()
                    .expect("POREP_MINIMUM_CHALLENGES poisoned")
                    .get(&u64::from(SectorSize::from(porep_config)))
                    .expect("unknown sector size") as usize,
            },
        )
    };

    info!("verify_seal:finish: {:?}", sector_id);
    result
}
```

然后，调用到了storage-proofs-core目录的verify方法，

其中，verify_proofs_batch方法在bellperson库里面实现。

**storage-proofs-core/src/compound_proof.rs**

```rust
fn verify<'b>(
        public_params: &PublicParams<'a, S>,
        public_inputs: &S::PublicInputs,
        multi_proof: &MultiProof<'b>,
        requirements: &S::Requirements,
    ) -> Result<bool> {
        ensure!(
            multi_proof.circuit_proofs.len() == Self::partition_count(public_params),
            "Inconsistent inputs"
        );

        let vanilla_public_params = &public_params.vanilla_params;
        let pvk = &multi_proof.verifying_key;

        if !<S as ProofScheme>::satisfies_requirements(
            &public_params.vanilla_params,
            requirements,
            multi_proof.circuit_proofs.len(),
        ) {
            return Ok(false);
        }

        let inputs: Vec<_> = (0..multi_proof.circuit_proofs.len())
            .into_par_iter()
            .map(|k| Self::generate_public_inputs(public_inputs, vanilla_public_params, Some(k)))
            .collect::<Result<_>>()?;

        let proofs: Vec<_> = multi_proof.circuit_proofs.iter().collect();
        let res = verify_proofs_batch(&pvk, &mut OsRng, &proofs, &inputs)?;
        Ok(res)
    }
```