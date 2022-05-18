<!-- vscode-markdown-toc -->
* 1. [零知识证明简介](#)
	* 1.1. [zkSNARK](#zkSNARK)
	* 1.2. [zkSTARKs](#zkSTARKs)
	* 1.3. [Bulletproofs](#Bulletproofs)
* 2. [Filecoin 中的零知识证明](#Filecoin)
	* 2.1. [合成电路和生成证明](#-1)
	* 2.2. [验证证明](#-1)
* 3. [bellperson](#bellperson)
	* 3.1. [针对电路生成Vk（验证密钥）](#Vk)
	* 3.2. [生成证明](#-1)
		* 3.2.1. [create_random_proof_batch](#create_random_proof_batch)
		* 3.2.2. [execute_fft](#execute_fft)
		* 3.2.3. [multiexp](#multiexp)
	* 3.3. [验证证明](#-1)

<!-- vscode-markdown-toc-config
	numbering=true
	autoSave=true
	/vscode-markdown-toc-config -->
<!-- /vscode-markdown-toc -->


##  1. <a name=''></a>零知识证明简介

常见的零知识证明库有zkSnark、zkStars以及Bulletproofs，其中 Filecoin 使用的零知识证明就是 zkSnark 中的 groth16 算法。

###  1.1. <a name='zkSNARK'></a>zkSNARK

常见算法，Groth16, Marlin，Sonic ，Plonk。

###  1.2. <a name='zkSTARKs'></a>zkSTARKs

###  1.3. <a name='Bulletproofs'></a>Bulletproofs

参考：https://github.com/matter-labs/awesome-zero-knowledge-proofs

##  2. <a name='Filecoin'></a>Filecoin 中的零知识证明

Filecoin 网络中使用零知识证明的地方有三个，分别是密封过程中的C2,时空证明以及暴块证明。

零知识证明的入口调用在工程 rust-fil-proofs 的 compound_proof.rs 。

###  2.1. <a name='-1'></a>合成电路和生成证明

circuit_proofs 方法：

1. 合成电路

2. 通过电路和 groth_params 生成证明

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
                    pub_in,
                    C::ComponentPrivateInputs::default(),
                    &vanilla_proof,
                    pub_params,
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

###  2.2. <a name='-1'></a>验证证明

验证证明的方法有两个，verify 和 batch_verify

```rust

    fn batch_verify<'b>(
        public_params: &PublicParams<'a, S>,
        public_inputs: &[S::PublicInputs],
        multi_proofs: &[MultiProof<'b>],
        requirements: &S::Requirements,
    ) -> Result<bool> {
        ensure!(
            public_inputs.len() == multi_proofs.len(),
            "Inconsistent inputs"
        );
        for proof in multi_proofs {
            ensure!(
                proof.circuit_proofs.len() == Self::partition_count(public_params),
                "Inconsistent inputs"
            );
        }
        ensure!(!public_inputs.is_empty(), "Cannot verify empty proofs");

        let vanilla_public_params = &public_params.vanilla_params;
        // just use the first one, the must be equal any way
        let pvk = &multi_proofs[0].verifying_key;

        for multi_proof in multi_proofs.iter() {
            if !<S as ProofScheme>::satisfies_requirements(
                &public_params.vanilla_params,
                requirements,
                multi_proof.circuit_proofs.len(),
            ) {
                return Ok(false);
            }
        }

        let inputs: Vec<_> = multi_proofs
            .par_iter()
            .zip(public_inputs.par_iter())
            .flat_map(|(multi_proof, pub_inputs)| {
                (0..multi_proof.circuit_proofs.len())
                    .into_par_iter()
                    .map(|k| {
                        Self::generate_public_inputs(pub_inputs, vanilla_public_params, Some(k))
                    })
                    .collect::<Result<Vec<_>>>()
                    .expect("Invalid public inputs") // TODO: improve error handling
            })
            .collect::<Vec<_>>();
        let circuit_proofs: Vec<_> = multi_proofs
            .iter()
            .flat_map(|m| m.circuit_proofs.iter())
            .collect();

        let res = verify_proofs_batch(pvk, &mut OsRng, &circuit_proofs[..], &inputs)?;

        Ok(res)
    }

```

##  3. <a name='bellperson'></a>bellperson

  [bellperson](https://github.com/filecoin-project/bellperson)库是Filecoin官方fork自[bellman](https://github.com/zkcrypto/bellman)，bellman是Zcash团队用Rust语言开发的一个zk-SNARK软件库，实现了Groth16算法，而bellperson是Filecoin官方针对次项目做了一些优化和改进。

###  3.1. <a name='Vk'></a>针对电路生成Vk（验证密钥）

下面代码来自 bench, 首先合成电路 circuit， 然后生成 prams ， params 包括 vk（验证私钥），通过验证私钥生成pvk(证明私钥)


**verifier-bench/src/main.rs**

```rust

fn main() {
    let mut rng = rand::rngs::OsRng;
    pretty_env_logger::init_timed();

    let opts = Opts::from_args();
    if opts.gpu {
        std::env::set_var("BELLMAN_VERIFIER", "gpu");
    } else {
        std::env::set_var("BELLMAN_NO_GPU", "1");
    }

    let circuit = DummyDemo {
        public: opts.public,
        private: opts.private,
    };
    let circuits = vec![circuit.clone(); opts.proofs];

    let params = if opts.dummy {
        dummy_params::<Bls12, _>(opts.public, opts.private, &mut rng)
    } else {
        println!("Generating params... (You can skip this by passing `--dummy` flag)");
        generate_random_parameters(circuit, &mut rng).unwrap()
    };
    let pvk = prepare_verifying_key(&params.vk);

    let srs = if opts.aggregate {
        let x = setup_fake_srs(&mut rng, opts.proofs).specialize(opts.proofs);
        Some(x)
    } else {
        None
    };

    // **省略**
}

```

###  3.2. <a name='-1'></a>生成证明

####  3.2.1. <a name='create_random_proof_batch'></a>create_random_proof_batch

create_random_proof_batch 方法： 生成证明

**src/groth16/ext.rs**

```rust

pub fn create_random_proof_batch<E, C, R, P: ParameterSource<E>>(
    circuits: Vec<C>,
    params: P,
    rng: &mut R,
) -> Result<Vec<Proof<E>>, SynthesisError>
where
    E: gpu::GpuEngine + MultiMillerLoop,
    C: Circuit<E::Fr> + Send,
    R: RngCore,
{
    create_random_proof_batch_priority::<E, C, R, P>(circuits, params, rng, false)
}

```

然后，会调到 prove.rs 的 create_proof_batch_priority 方法：

1. 快速傅里叶变换算法(FFT)

2. 指数乘积算法(Multiexp,全名multi-exponentiation)

```rust
#[allow(clippy::clippy::needless_collect)]
pub fn create_proof_batch_priority<E, C, P: ParameterSource<E>>(
    circuits: Vec<C>,
    params: P,
    r_s: Vec<E::Fr>,
    s_s: Vec<E::Fr>,
    priority: bool,
) -> Result<Vec<Proof<E>>, SynthesisError>
where
    E: gpu::GpuEngine + MultiMillerLoop,
    C: Circuit<E::Fr> + Send,
{
    info!("Bellperson {} is being used!", BELLMAN_VERSION);

    let (start, mut provers, input_assignments, aux_assignments) =
        create_proof_batch_priority_inner(circuits)?;

    let worker = Worker::new();
    let input_len = input_assignments[0].len();
    let vk = params.get_vk(input_len)?.clone();
    let n = provers[0].a.len();
    let a_aux_density_total = provers[0].a_aux_density.get_total_density();
    let b_input_density_total = provers[0].b_input_density.get_total_density();
    let b_aux_density_total = provers[0].b_aux_density.get_total_density();
    let aux_assignment_len = provers[0].aux_assignment.len();
    let num_circuits = provers.len();
circuit_proofs
            "only equaly sized circuits are supported"
        );
        debug_assert_eq!(
            a_aux_density_total,
            prover.a_aux_density.get_total_density(),
            "only identical circuits are supported"
        );
        debug_assert_eq!(
            b_input_density_total,
            prover.b_input_density.get_total_density(),
            "only identical circuits are supported"
        );
        debug_assert_eq!(
            b_aux_density_total,
            prover.b_aux_density.get_total_density(),
            "only identical circuits are supported"
        );
    }

    let mut log_d = 0;
    while (1 << log_d) < n {
        log_d += 1;
    }

    #[cfg(any(feature = "cuda", feature = "opencl"))]
    let prio_lock = if priority {
        trace!("acquiring priority lock");
        Some(PriorityLock::lock())
    } else {
        None
    };

    let mut a_s = Vec::with_capacity(num_circuits);
    let mut params_h = None;
    let worker = &worker;
    let provers_ref = &mut provers;
    let params = &params;

    THREAD_POOL.scoped(|s| -> Result<(), SynthesisError> {
        let params_h = &mut params_h;
        s.execute(move || {
            debug!("get h");
            *params_h = Some(params.get_h(n));
        });

        let mut fft_kern = Some(LockedFFTKernel::<E>::new(log_d, priority));
        for prover in provers_ref {
            a_s.push(execute_fft(worker, prover, &mut fft_kern)?);
        }
        Ok(())
    })?;

    let mut multiexp_kern = Some(LockedMultiexpKernel::<E>::new(log_d, priority));
    let params_h = params_h.unwrap()?;

    let mut h_s = Vec::with_capacity(num_circuits);
    let mut params_l = None;

    THREAD_POOL.scoped(|s| {
        let params_l = &mut params_l;
        s.execute(move || {
            debug!("get l");
            *params_l = Some(params.get_l(aux_assignment_len));
        });

        debug!("multiexp h");
        for a in a_s.into_iter() {
            h_s.push(multiexp(
                &worker,
                params_h.clone(),
                FullDensity,
                a,
                &mut multiexp_kern,
            ));
        }
    });

    let params_l = params_l.unwrap()?;

    let mut l_s = Vec::with_capacity(num_circuits);
    let mut params_a = None;
    let mut params_b_g1 = None;
    let mut params_b_g2 = None;
    let a_aux_density_total = provers[0].a_aux_density.get_total_density();
    let b_input_density_total = provers[0].b_input_density.get_total_density();
    let b_aux_density_total = provers[0].b_aux_density.get_total_density();

    THREAD_POOL.scoped(|s| {
        let params_a = &mut params_a;
        let params_b_g1 = &mut params_b_g1;
        let params_b_g2 = &mut params_b_g2;
        s.execute(move || {
            debug!("get_a b_g1 b_g2");
            *params_a = Some(params.get_a(input_len, a_aux_density_total));
            *params_b_g1 = Some(params.get_b_g1(b_input_density_total, b_aux_density_total));
            *params_b_g2 = Some(params.get_b_g2(b_input_density_total, b_aux_density_total));
        });

        debug!("multiexp l");
        for aux in aux_assignments.iter() {
            l_s.push(multiexp(
                &worker,
                params_l.clone(),
                FullDensity,
                aux.clone(),
                &mut multiexp_kern,
            ));
        }
    });

    debug!("get_a b_g1 b_g2");
    let (a_inputs_source, a_aux_source) = params_a.unwrap()?;
    let (b_g1_inputs_source, b_g1_aux_source) = params_b_g1.unwrap()?;
    let (b_g2_inputs_source, b_g2_aux_source) = params_b_g2.unwrap()?;

    debug!("multiexp a b_g1 b_g2");
    let inputs = provers
        .into_iter()
        .zip(input_assignments.iter())
        .zip(aux_assignments.iter())
        .map(|((prover, input_assignment), aux_assignment)| {
            let a_inputs = multiexp(
                &worker,
                a_inputs_source.clone(),
                FullDensity,
                input_assignment.clone(),
                &mut multiexp_kern,
            );

            let a_aux = multiexp(
                &worker,
                a_aux_source.clone(),
                Arc::new(prover.a_aux_density),
                aux_assignment.clone(),
                &mut multiexp_kern,
            );

            let b_input_density = Arc::new(prover.b_input_density);
            let b_aux_density = Arc::new(prover.b_aux_density);

            let b_g1_inputs = multiexp(
                &worker,
                b_g1_inputs_source.clone(),
                b_input_density.clone(),
                input_assignment.clone(),
                &mut multiexp_kern,
            );

            let b_g1_aux = multiexp(
                &worker,
                b_g1_aux_source.clone(),
                b_aux_density.clone(),
                aux_assignment.clone(),
                &mut multiexp_kern,
            );

            let b_g2_inputs = multiexp(
                &worker,
                b_g2_inputs_source.clone(),
                b_input_density,
                input_assignment.clone(),
                &mut multiexp_kern,
            );
            let b_g2_aux = multiexp(
                &worker,
                b_g2_aux_source.clone(),
                b_aux_density,
                aux_assignment.clone(),
                &mut multiexp_kern,
            );

            (
                a_inputs,
                a_aux,
                b_g1_inputs,
                b_g1_aux,
                b_g2_inputs,
                b_g2_aux,
            )
        })
        .collect::<Vec<_>>();
    drop(multiexp_kern);
    drop(a_inputs_source);
    drop(a_aux_source);
    drop(b_g1_inputs_source);
    drop(b_g1_aux_source);
    drop(b_g2_inputs_source);
    drop(b_g2_aux_source);

    debug!("proofs");
    let proofs = h_s
        .into_iter()
        .zip(l_s.into_iter())
        .zip(inputs.into_iter())
        .zip(r_s.into_iter())
        .zip(s_s.into_iter())
        .map(
            |(
                (((h, l), (a_inputs, a_aux, b_g1_inputs, b_g1_aux, b_g2_inputs, b_g2_aux)), r),
                s,
            )| {
                if (vk.delta_g1.is_identity() | vk.delta_g2.is_identity()).into() {
                    // If this element is zero, someone is trying to perform a
                    // subversion-CRS attack.
                    return Err(SynthesisError::UnexpectedIdentity);
                }

                let mut g_a = vk.delta_g1.mul(r);
                g_a.add_assign(&vk.alpha_g1);
                let mut g_b = vk.delta_g2.mul(s);
                g_b.add_assign(&vk.beta_g2);
                let mut g_c;
                {
                    let mut rs = r;
                    rs.mul_assign(&s);

                    g_c = vk.delta_g1.mul(rs);
                    g_c.add_assign(&vk.alpha_g1.mul(s));
                    g_c.add_assign(&vk.beta_g1.mul(r));
                }
                let mut a_answer = a_inputs.wait()?;
                a_answer.add_assign(&a_aux.wait()?);
                g_a.add_assign(&a_answer);
                a_answer.mul_assign(s);
                g_c.add_assign(&a_answer);

                let mut b1_answer = b_g1_inputs.wait()?;
                b1_answer.add_assign(&b_g1_aux.wait()?);
                let mut b2_answer = b_g2_inputs.wait()?;
                b2_answer.add_assign(&b_g2_aux.wait()?);

                g_b.add_assign(&b2_answer);
                b1_answer.mul_assign(r);
                g_c.add_assign(&b1_answer);
                g_c.add_assign(&h.wait()?);
                g_c.add_assign(&l.wait()?);

                Ok(Proof {
                    a: g_a.to_affine(),
                    b: g_b.to_affine(),
                    c: g_c.to_affine(),
                })
            },
        )
        .collect::<Result<Vec<_>, SynthesisError>>()?;

    #[cfg(any(feature = "cuda", feature = "opencl"))]
    {
        trace!("dropping priority lock");
        drop(prio_lock);
    }

    let proof_time = start.elapsed();
    info!("prover time: {:?}", proof_time);

    Ok(proofs)
}

```

####  3.2.2. <a name='execute_fft'></a>execute_fft

execute_fft 方法：快速傅里叶变换算法的计算

**src/groth16/prover.rs**

```rust

fn execute_fft<E>(
    worker: &Worker,
    prover: &mut ProvingAssignment<E::Fr>,
    fft_kern: &mut Option<LockedFFTKernel<E>>,
) -> Result<Arc<Vec<<E::Fr as PrimeField>::Repr>>, SynthesisError>
where
    E: gpu::GpuEngine + MultiMillerLoop,
{
    let mut a = EvaluationDomain::from_coeffs(std::mem::replace(&mut prover.a, Vec::new()))?;
    let mut b = EvaluationDomain::from_coeffs(std::mem::replace(&mut prover.b, Vec::new()))?;
    let mut c = EvaluationDomain::from_coeffs(std::mem::replace(&mut prover.c, Vec::new()))?;

    EvaluationDomain::ifft_many(&mut [&mut a, &mut b, &mut c], &worker, fft_kern)?;
    EvaluationDomain::coset_fft_many(&mut [&mut a, &mut b, &mut c], &worker, fft_kern)?;

    a.mul_assign(&worker, &b);
    drop(b);
    a.sub_assign(&worker, &c);
    drop(c);

    a.divide_by_z_on_coset(&worker);
    a.icoset_fft(&worker, fft_kern)?;

    let a = a.into_coeffs();
    let a_len = a.len() - 1;
    let a = a
        .into_par_iter()
        .take(a_len)
        .map(|s| s.to_repr())
        .collect::<Vec<_>>();
    Ok(Arc::new(a))
}

```


####  3.2.3. <a name='multiexp'></a>multiexp

multiexp 方法： 也即， multi exponentiation，指数乘积算法的计算。

**src/multiexp.rs**

```rust

pub fn multiexp<Q, D, G, E, S>(
    pool: &Worker,
    bases: S,
    density_map: D,
    exponents: Arc<Vec<<G::Scalar as PrimeField>::Repr>>,
    kern: &mut Option<gpu::LockedMultiexpKernel<E>>,
) -> Waiter<Result<<G as PrimeCurveAffine>::Curve, SynthesisError>>
where
    for<'a> &'a Q: QueryDensity,
    D: Send + Sync + 'static + Clone + AsRef<Q>,
    G: PrimeCurveAffine,
    E: gpu::GpuEngine,
    E: Engine<Fr = G::Scalar>,
    S: SourceBuilder<G>,
{
    if let Some(ref mut kern) = kern {
        if let Ok(p) = kern.with(|k: &mut gpu::MultiexpKernel<E>| {
            let exps = density_map.as_ref().generate_exps::<E>(exponents.clone());
            let (bss, skip) = bases.clone().get();
            let n = exps.len();
            k.multiexp(pool, bss, exps, skip, n)
        }) {
            return Waiter::done(Ok(p));
        }
    }

    let c = if exponents.len() < 32 {
        3u32
    } else {
        (f64::from(exponents.len() as u32)).ln().ceil() as u32
    };

    if let Some(query_size) = density_map.as_ref().get_query_size() {
        // If the density map has a known query size, it should not be
        // inconsistent with the number of exponents.
        assert!(query_size == exponents.len());
    }

    #[allow(clippy::let_and_return)]
    let result = pool.compute(move || multiexp_inner(bases, density_map, exponents, c));
    #[cfg(any(feature = "cuda", feature = "opencl"))]
    {
        // Do not give the control back to the caller till the
        // multiexp is done. We may want to reacquire the GPU again
        // between the multiexps.
        let result = result.wait();
        Waiter::done(result)
    }
    #[cfg(not(any(feature = "cuda", feature = "opencl")))]
    result
}

```

最终，如果使用GPU，两个算法的计算都会使用GPU进行计算。比如，mutltiexp 的代码如下：

如果要优化GPU的计算，可以调整 num_groups（工作组） 和 num_windows（工作组） 这两个值的大小，从而增大GPU每次计算的吞吐量，减少堆内存的交换。

执行 kernel.run 之后，最后会执行 opencl 库的 clEnqueueNDRangeKernel 方法。

调用过程如下：bellperson -> rust-gpu-tools -> fil-ocl -> opencl

**src/gpu/multiexp.rs**

```rust

 pub fn multiexp<G>(
        &mut self,
        bases: &[G],
        exps: &[<G::Scalar as PrimeField>::Repr],
        n: usize,
    ) -> GPUResult<<G as PrimeCurveAffine>::Curve>
    where
        G: PrimeCurveAffine,
    {
        if locks::PriorityLock::should_break(self.priority) {
            return Err(GPUError::GPUTaken);
        }

        let exp_bits = exp_size::<E>() * 8;
        let window_size = calc_window_size(n as usize, exp_bits, self.core_count);
        let num_windows = ((exp_bits as f64) / (window_size as f64)).ceil() as usize;
        let num_groups = calc_num_groups(self.core_count, num_windows);
        let bucket_len = 1 << window_size;

        // Each group will have `num_windows` threads and as there are `num_groups` groups, there will
        // be `num_groups` * `num_windows` threads in total.
        // Each thread will use `num_groups` * `num_windows` * `bucket_len` buckets.

        let closures = program_closures!(
            |program, _arg| -> GPUResult<Vec<<G as PrimeCurveAffine>::Curve>> {
                let base_buffer = program.create_buffer_from_slice(bases)?;
                let exp_buffer = program.create_buffer_from_slice(exps)?;

                // It is safe as the GPU will initialize that buffer
                let bucket_buffer = unsafe {
                    program.create_buffer::<<G as PrimeCurveAffine>::Curve>(
                        2 * self.core_count * bucket_len,
                    )?
                };
                // It is safe as the GPU will initialize that buffer
                let result_buffer = unsafe {
                    program.create_buffer::<<G as PrimeCurveAffine>::Curve>(2 * self.core_count)?
                };

                // The global work size follows CUDA's definition and is the number of
                // `LOCAL_WORK_SIZE` sized thread groups.
                let global_work_size =
                    (num_windows * num_groups + LOCAL_WORK_SIZE - 1) / LOCAL_WORK_SIZE;

                let kernel = program.create_kernel(
                    if TypeId::of::<G>() == TypeId::of::<E::G1Affine>() {
                        "G1_bellman_multiexp"
                    } else if TypeId::of::<G>() == TypeId::of::<E::G2Affine>() {
                        "G2_bellman_multiexp"
                    } else {
                        return Err(GPUError::Simple("Only E::G1 and E::G2 are supported!"));
                    },
                    global_work_size,
                    LOCAL_WORK_SIZE,
                )?;

                kernel
                    .arg(&base_buffer)
                    .arg(&bucket_buffer)
                    .arg(&result_buffer)
                    .arg(&exp_buffer)
                    .arg(&(n as u32))
                    .arg(&(num_groups as u32))
                    .arg(&(num_windows as u32))
                    .arg(&(window_size as u32))
                    .run()?;

                let mut results =
                    vec![<G as PrimeCurveAffine>::Curve::identity(); 2 * self.core_count];
                program.read_into_buffer(&result_buffer, &mut results)?;

                Ok(results)
            }
        );

        let results = self.program.run(closures, ())?;

        // Using the algorithm below, we can calculate the final result by accumulating the results
        // of those `NUM_GROUPS` * `NUM_WINDOWS` threads.
        let mut acc = <G as PrimeCurveAffine>::Curve::identity();
        let mut bits = 0;
        for i in 0..num_windows {
            let w = std::cmp::min(window_size, exp_bits - bits);
            for _ in 0..w {
                acc = acc.double();
            }
            for g in 0..num_groups {
                acc.add_assign(&results[g * num_windows + i]);
            }
            bits += w; // Process the next window
        }

        Ok(acc)
    }
}

```


###  3.3. <a name='-1'></a>验证证明

**src/groth16/verifier.rs**

verify_proofs_batch 方法： 拿到 pvk(验证密钥)和证明的结果进行验证。

```rust

pub fn verify_proofs_batch<'a, E, R>(
    pvk: &'a PreparedVerifyingKey<E>,
    rng: &mut R,
    proofs: &[&Proof<E>],
    public_inputs: &[Vec<E::Fr>],
) -> Result<bool, SynthesisError>
where
    E: MultiMillerLoop,
    <E::Fr as PrimeField>::Repr: Sync + Copy,
    R: rand::RngCore,
{
    debug_assert_eq!(proofs.len(), public_inputs.len());

    for pub_input in public_inputs {
        if (pub_input.len() + 1) != pvk.ic.len() {
            return Err(SynthesisError::MalformedVerifyingKey);
        }
    }

    let num_inputs = public_inputs[0].len();
    let num_proofs = proofs.len();

    if num_proofs < 2 {
        return verify_proof(pvk, proofs[0], &public_inputs[0]);
    }

    let proof_num = proofs.len();

    // Choose random coefficients for combining the proofs.
    let mut rand_z_repr: Vec<_> = Vec::with_capacity(proof_num);
    let mut rand_z: Vec<_> = Vec::with_capacity(proof_num);
    let mut accum_y = E::Fr::zero();

    for _ in 0..proof_num {
        use rand::Rng;

        let t: u128 = rng.gen();

        let mut repr = E::Fr::zero().to_repr();
        let mut repr_u64s = le_bytes_to_u64s(&repr.as_ref());
        assert!(repr_u64s.len() > 1);

        repr_u64s[0] = (t & (-1i64 as u128) >> 64) as u64;
        repr_u64s[1] = (t >> 64) as u64;

        for (i, limb) in repr_u64s.iter().enumerate() {
            let start = i * 8;
            let stop = start + 8;
            repr.as_mut()[start..stop].copy_from_slice(&limb.to_le_bytes());
        }

        let fr = E::Fr::from_repr(repr).unwrap();
        let repr = fr.to_repr();

        // calculate sum
        accum_y.add_assign(&fr);
        // store FrRepr
        rand_z_repr.push(repr);
        // store Fr
        rand_z.push(fr);
    }

    // MillerLoop(\sum Accum_Gamma)
    let mut ml_g = <E as MultiMillerLoop>::Result::default();
    // MillerLoop(Accum_Delta)
    let mut ml_d = <E as MultiMillerLoop>::Result::default();
    // MillerLoop(Accum_AB)
    let mut acc_ab = <E as MultiMillerLoop>::Result::default();
    // Y^-Accum_Y
    let mut y = <E as Engine>::Gt::identity();

    let accum_y = &accum_y;
    let rand_z_repr = &rand_z_repr;

    rayon::in_place_scope(|s| {
        // - Thread 1: Calculate MillerLoop(\sum Accum_Gamma)
        let ml_g = &mut ml_g;
        s.spawn(move |_| {
            let scalar_getter = |idx: usize| -> <E::Fr as ff::PrimeField>::Repr {
                if idx == 0 {
                    return accum_y.to_repr();
                }
                let idx = idx - 1;

                // \sum(z_j * aj,i)
                let mut cur_sum = rand_z[0];
                cur_sum.mul_assign(&public_inputs[0][idx]);

                for (pi_mont, mut rand_mont) in
                    public_inputs.iter().zip(rand_z.iter().copied()).skip(1)
                {
                    // z_j * a_j,i
                    let pi_mont = &pi_mont[idx];
                    rand_mont.mul_assign(pi_mont);
                    cur_sum.add_assign(&rand_mont);
                }

                cur_sum.to_repr()
            };

            // \sum Accum_Gamma
            let acc_g_psi = multiscalar::par_multiscalar::<_, E::G1Affine>(
                &multiscalar::ScalarList::Getter(scalar_getter, num_inputs + 1),
                &pvk.multiscalar,
                256,
            );

            // MillerLoop(acc_g_psi, vk.gamma)
            *ml_g = E::multi_miller_loop(&[(&acc_g_psi.to_affine(), &pvk.gamma_g2)]);
        });

        // - Thread 2: Calculate MillerLoop(Accum_Delta)
        let ml_d = &mut ml_d;
        s.spawn(move |_| {
            let points: Vec<_> = proofs.iter().map(|p| p.c).collect();

            // Accum_Delta
            let acc_d: E::G1 = {
                let pre = multiscalar::precompute_fixed_window::<E::G1Affine>(&points, 1);
                multiscalar::multiscalar::<E::G1Affine>(
                    &rand_z_repr,
                    &pre,
                    std::mem::size_of::<<E::Fr as PrimeField>::Repr>() * 8,
                )
            };

            *ml_d = E::multi_miller_loop(&[(&acc_d.to_affine(), &pvk.delta_g2)]);
        });

        // - Thread 3: Calculate MillerLoop(Accum_AB)
        let acc_ab = &mut acc_ab;
        s.spawn(move |_| {
            let accum_ab_mls: Vec<_> = proofs
                .par_iter()
                .zip(rand_z_repr.par_iter())
                .map(|(proof, rand)| {
                    // [z_j] pi_j,A
                    let mul_a = proof.a.mul(E::Fr::from_repr(*rand).unwrap());

                    // -pi_j,B
                    let cur_neg_b = -proof.b.to_curve();

                    E::multi_miller_loop(&[(&mul_a.to_affine(), &cur_neg_b.to_affine().into())])
                })
                .collect();

            // Accum_AB = mul_j(ml((zj*proof_aj), -proof_bj))
            *acc_ab = accum_ab_mls[0];
            for accum in accum_ab_mls.iter().skip(1).take(num_proofs) {
                *acc_ab += accum;
            }
        });

        // Thread 4(current): Calculate Y^-Accum_Y
        // -Accum_Y
        let accum_y_neg = -*accum_y;

        // Y^-Accum_Y
        y = pvk.alpha_g1_beta_g2 * accum_y_neg;
    });

    let mut ml_all = acc_ab;
    ml_all += ml_d;
    ml_all += ml_g;

    let actual = ml_all.final_exponentiation();
    Ok(actual == y)
}

```

参考：

[Awesome zero knowledge proofs (zkp) 对比](https://github.com/matter-labs/awesome-zero-knowledge-proofs)

[硬核 | 技术解析热门零知识证明方案 Groth16](https://www.ylfx.com/Show/index/cid/37/id/283117.html)


