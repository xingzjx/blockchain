# Filecoin 源码解读：虚拟机执行原理

环境：lotus v1.11.0

Filecoin的*VM*相当于以太坊虚拟机，用来解析和执行指令的核心模块。而　*actor* 相当于以太坊中的智能合约，有大概11种内置的actor。接下来，以转账为例，分析*VM*的执行流程。

当执行转账或者创建矿工以及创世区块等操作的时候，都会执行交易，这和以太坊类似。其中，这个过程可以划分为３个流程：

- 消息池：添加交易到 mpool，在以太坊里面叫*tx_pool*，Filecoin 中的 message　相当　以太坊的　trasaction
- 矿工：矿工负责打包交易
- 虚拟机：Filecoin VM 解析指令，并且执行具体的 actor 代码（智能合约）

执行命令：

```bash

lotus send t3qbvcjx52r7426qiqphxzjtpktnzi5f2v4kc6bptipbwyp3pok6stijtgi63huobvw6l2ugqzqprihrhef6xa 1

```

然后，虚拟机会执行如下方法：

**chain/vm/vm.go**

```go
func (vm *VM) ApplyImplicitMessage(ctx context.Context, msg *types.Message) (*ApplyRet, error) {
	start := build.Clock.Now()
	defer atomic.AddUint64(&StatApplied, 1)
	ret, actorErr, rt := vm.send(ctx, msg, nil, nil, start)
	rt.finilizeGasTracing()
	return &ApplyRet{
		MessageReceipt: types.MessageReceipt{
			ExitCode: aerrors.RetCode(actorErr),
			Return:   ret,
			GasUsed:  0,
		},
		ActorErr:       actorErr,
		ExecutionTrace: rt.executionTrace,
		GasCosts:       nil,
		Duration:       time.Since(start),
	}, actorErr
}
```

ApplyImplicitMessage方法中调用了 *vm.go* 的send方法，其返回结果会封装到MessageReceipt里面。


send 方法逻辑：

- vm.makeRuntime : 创建runtime
- chargeGasSafe ：计算gas费，gas费不够，则无法执行交易
- vm.Invoke　: 执行 Invoke　方法

**chain/vm/vm.go**

```go
func (vm *VM) send(ctx context.Context, msg *types.Message, parent *Runtime,
	gasCharge *GasCharge, start time.Time) ([]byte, aerrors.ActorError, *Runtime) {
	defer atomic.AddUint64(&StatSends, 1)

	st := vm.cstate

	rt := vm.makeRuntime(ctx, msg, parent)
	if EnableGasTracing {
		rt.lastGasChargeTime = start
		if parent != nil {
			rt.lastGasChargeTime = parent.lastGasChargeTime
			rt.lastGasCharge = parent.lastGasCharge
			defer func() {
				parent.lastGasChargeTime = rt.lastGasChargeTime
				parent.lastGasCharge = rt.lastGasCharge
			}()
		}
	}

	if parent != nil {
		defer func() {
			parent.gasUsed = rt.gasUsed
		}()
	}
	if gasCharge != nil {
		if err := rt.chargeGasSafe(*gasCharge); err != nil {
			// this should never happen
			return nil, aerrors.Wrap(err, "not enough gas for initial message charge, this should not happen"), rt
		}
	}

	ret, err := func() ([]byte, aerrors.ActorError) {
		_ = rt.chargeGasSafe(newGasCharge("OnGetActor", 0, 0))
		toActor, err := st.GetActor(msg.To)
		if err != nil {
			if xerrors.Is(err, types.ErrActorNotFound) {
				a, aid, err := TryCreateAccountActor(rt, msg.To)
				if err != nil {
					return nil, aerrors.Wrapf(err, "could not create account")
				}
				toActor = a
				if vm.ntwkVersion(ctx, vm.blockHeight) <= network.Version3 {
					// Leave the rt.Message as is
				} else {
					nmsg := Message{
						msg: types.Message{
							To:    aid,
							From:  rt.Message.Caller(),
							Value: rt.Message.ValueReceived(),
						},
					}

					rt.Message = &nmsg
				}
			} else {
				return nil, aerrors.Escalate(err, "getting actor")
			}
		}

		if aerr := rt.chargeGasSafe(rt.Pricelist().OnMethodInvocation(msg.Value, msg.Method)); aerr != nil {
			return nil, aerrors.Wrap(aerr, "not enough gas for method invocation")
		}

		// not charging any gas, just logging
		//nolint:errcheck
		defer rt.chargeGasSafe(newGasCharge("OnMethodInvocationDone", 0, 0))

		if types.BigCmp(msg.Value, types.NewInt(0)) != 0 {
			if err := vm.transfer(msg.From, msg.To, msg.Value); err != nil {
				return nil, aerrors.Wrap(err, "failed to transfer funds")
			}
		}

		if msg.Method != 0 {
			var ret []byte
			_ = rt.chargeGasSafe(gasOnActorExec)
			ret, err := vm.Invoke(toActor, rt, msg.Method, msg.Params)
			return ret, err
		}
		return nil, nil
	}()

	mr := types.MessageReceipt{
		ExitCode: aerrors.RetCode(err),
		Return:   ret,
		GasUsed:  rt.gasUsed,
	}
	rt.executionTrace.MsgRct = &mr
	rt.executionTrace.Duration = time.Since(start)
	if err != nil {
		rt.executionTrace.Error = err.Error()
	}

	return ret, err, rt
}
```

Invoke 方法：会调用ActorRegistry的Invoke函数

**blockchain/vm/vm.go**

```go
func (vm *VM) Invoke(act *types.Actor, rt *Runtime, method abi.MethodNum, params []byte) ([]byte, aerrors.ActorError) {
	ctx, span := trace.StartSpan(rt.ctx, "vm.Invoke")
	defer span.End()
	if span.IsRecordingEvents() {
		span.AddAttributes(
			trace.StringAttribute("to", rt.Receiver().String()),
			trace.Int64Attribute("method", int64(method)),
			trace.StringAttribute("value", rt.ValueReceived().String()),
		)
	}

	var oldCtx context.Context
	oldCtx, rt.ctx = rt.ctx, ctx
	defer func() {
		rt.ctx = oldCtx
	}()
	ret, err := vm.areg.Invoke(act.Code, rt, method, params)
	if err != nil {
		return nil, err
	}
	return ret, nil
}
```

Invoke方法：会找到actor的方法，然后执行该方法（反射机制）。

**blockchain/vm/invoker.go**

```go
func (ar *ActorRegistry) Invoke(codeCid cid.Cid, rt vmr.Runtime, method abi.MethodNum, params []byte) ([]byte, aerrors.ActorError) {
	act, ok := ar.actors[codeCid]
	if !ok {
		log.Errorf("no code for actor %s (Addr: %s)", codeCid, rt.Receiver())
		return nil, aerrors.Newf(exitcode.SysErrorIllegalActor, "no code for actor %s(%d)(%s)", codeCid, method, hex.EncodeToString(params))
	}
	if err := act.predicate(rt, act.vmActor); err != nil {
		return nil, aerrors.Newf(exitcode.SysErrorIllegalActor, "unsupported actor: %s", err)
	}
	if method >= abi.MethodNum(len(act.methods)) || act.methods[method] == nil {
		return nil, aerrors.Newf(exitcode.SysErrInvalidMethod, "no method %d on actor", method)
	}
	return act.methods[method](rt, params)

}
```

关于actor的规范在：https://github.com/filecoin-project/specs-actors