<h1>奇亚挖矿学习指南</>

# Chia挖矿入门教程

&emsp;&emsp;chia挖矿教程目前比较多，不重复再写。

&emsp;&emsp;参考如下教程：

[Chia挖矿教程(Hpool官方)](https://www.hpool.com/help/tutorial/Chia%E6%8C%96%E7%9F%BF%E6%95%99%E7%A8%8B)

[Chia挖矿体验（加入矿池版）](https://www.chainnode.com/post/542332)

[普通玩家用群晖NAS挖矿（Chia奇亚币）教程](https://wp.gxnas.com/10559.html)

[uupool矿池挖chia](https://www.uupool.com/)

&emsp;&emsp;注意事项：

- Hpool提供的p盘工具plotting.dat（目前只支持Windows下p盘），和chia官方钱包中的p盘功能是一致的。只不过，使用hpool提供的p盘工具p盘，就可以不用下载官方钱包了。
- Chia官方钱包的区块同步，是同步china主网的数据，同步的数据并不大，而p盘过程中是需要大量磁盘的。
- Chia进行p盘的时候，有不同的农田大小规格，比如101.4G，但是实际是需要更大的存储空间的，101.4G是最终生成plot文件的大小。
- 矿池和solo挖矿是可以相互切换的，使用矿池挖矿，只不过把时空证明交给矿池来处理，下面会从源码角度解读这个过程。
  
# Chia关键部分源码分析

&emsp;&emsp;Chia钱包客户端通过RPC接口和主网通信，接口文档如下：

[Chia RPC接口文档](https://github.com/Chia-Network/chia-blockchain/wiki/RPC-Interfaces)
  
## 同步区块
&emsp;&emsp;同步区块就是把当前节点和其它全节点的数据保持一致性的过程。当创建或者导入钱包，或者在钱包里面手动连接其它节点的时候，前端会调用RPC接口open_connection。

rpc_server.py
```python
    async def open_connection(self, request: Dict):
        host = request["host"]
        port = request["port"]
        target_node: PeerInfo = PeerInfo(host, uint16(int(port)))
        on_connect = None
        if hasattr(self.rpc_api.service, "on_connect"):
            on_connect = self.rpc_api.service.on_connect
        if getattr(self.rpc_api.service, "server", None) is None or not (
            await self.rpc_api.service.server.start_client(target_node, on_connect)
        ):
            raise ValueError("Start client failed, or server is not set")
        return {}
```

&emsp;&emsp;打开连接后，也就是新节点加入的时候，会触发如下方法
full_node_api.py
```python
    @execute_task
    @peer_required
    @api_request
    async def new_peak(self, request: full_node_protocol.NewPeak, peer: ws.WSChiaConnection) -> Optional[Message]:
    """
    A peer notifies us that they have added a new peak to their blockchain. If we don't have it,
    we can ask for it.
    """
    async with self.full_node.new_peak_lock:
    return await self.full_node.new_peak(request, peer)
```

full_node.py
```python
    async def new_peak(self, request: full_node_protocol.NewPeak, peer: ws.WSChiaConnection):
        """
        We have received a notification of a new peak from a peer. This happens either when we have just connected,
        or when the peer has updated their peak.

        Args:
            request: information about the new peak
            peer: peer that sent the message

        """

        # Store this peak/peer combination in case we want to sync to it, and to keep track of peers
        self.sync_store.peer_has_block(request.header_hash, peer.peer_node_id, request.weight, request.height, True)

        if self.blockchain.contains_block(request.header_hash):
            return None

        # Not interested in less heavy peaks
        peak: Optional[BlockRecord] = self.blockchain.get_peak()
        curr_peak_height = uint32(0) if peak is None else peak.height
        if peak is not None and peak.weight > request.weight:
            return None

        if self.sync_store.get_sync_mode():
            # If peer connects while we are syncing, check if they have the block we are syncing towards
            peak_sync_hash = self.sync_store.get_sync_target_hash()
            peak_sync_height = self.sync_store.get_sync_target_height()
            if peak_sync_hash is not None and request.header_hash != peak_sync_hash and peak_sync_height is not None:
                peak_peers: Set[bytes32] = self.sync_store.get_peers_that_have_peak([peak_sync_hash])
                # Don't ask if we already know this peer has the peak
                if peer.peer_node_id not in peak_peers:
                    target_peak_response: Optional[RespondBlock] = await peer.request_block(
                        full_node_protocol.RequestBlock(uint32(peak_sync_height), False), timeout=10
                    )
                    if target_peak_response is not None and isinstance(target_peak_response, RespondBlock):
                        self.sync_store.peer_has_block(
                            peak_sync_hash,
                            peer.peer_node_id,
                            target_peak_response.block.weight,
                            peak_sync_height,
                            False,
                        )
        else:
            if request.height <= curr_peak_height + self.config["short_sync_blocks_behind_threshold"]:
                # This is the normal case of receiving the next block
                if await self.short_sync_backtrack(
                    peer, curr_peak_height, request.height, request.unfinished_reward_block_hash
                ):
                    return None

            if request.height < self.constants.WEIGHT_PROOF_RECENT_BLOCKS:
                # This is the case of syncing up more than a few blocks, at the start of the chain
                # TODO(almog): fix weight proofs so they work at the beginning as well
                self.log.debug("Doing batch sync, no backup")
                await self.short_sync_batch(peer, uint32(0), request.height)
                return None

            if request.height < curr_peak_height + self.config["sync_blocks_behind_threshold"]:
                # This case of being behind but not by so much
                if await self.short_sync_batch(peer, uint32(max(curr_peak_height - 6, 0)), request.height):
                    return None

            # This is the either the case where we were not able to sync successfully (for example, due to the fork
            # point being in the past), or we are very far behind. Performs a long sync.
            self._sync_task = asyncio.create_task(self._sync())
```

full_node.py
```python
 async def _sync(self):
        """
        Performs a full sync of the blockchain up to the peak.
            - Wait a few seconds for peers to send us their peaks
            - Select the heaviest peak, and request a weight proof from a peer with that peak
            - Validate the weight proof, and disconnect from the peer if invalid
            - Find the fork point to see where to start downloading blocks
            - Download blocks in batch (and in parallel) and verify them one at a time
            - Disconnect peers that provide invalid blocks or don't have the blocks
        """
        if self.weight_proof_handler is None:
            return None
        # Ensure we are only syncing once and not double calling this method
        if self.sync_store.get_sync_mode():
            return None

        if self.sync_store.get_long_sync():
            self.log.debug("already in long sync")
            return None

        self.sync_store.set_long_sync(True)
        self.log.debug("long sync started")
        try:
            self.log.info("Starting to perform sync.")
            self.log.info("Waiting to receive peaks from peers.")

            # Wait until we have 3 peaks or up to a max of 30 seconds
            peaks = []
            for i in range(300):
                peaks = [tup[0] for tup in self.sync_store.get_peak_of_each_peer().values()]
                if len(self.sync_store.get_peers_that_have_peak(peaks)) < 3:
                    if self._shut_down:
                        return None
                    await asyncio.sleep(0.1)

            self.log.info(f"Collected a total of {len(peaks)} peaks.")
            self.sync_peers_handler = None

            # Based on responses from peers about the current peaks, see which peak is the heaviest
            # (similar to longest chain rule).
            target_peak = self.sync_store.get_heaviest_peak()

            if target_peak is None:
                raise RuntimeError("Not performing sync, no peaks collected")
            heaviest_peak_hash, heaviest_peak_height, heaviest_peak_weight = target_peak
            self.sync_store.set_peak_target(heaviest_peak_hash, heaviest_peak_height)

            self.log.info(f"Selected peak {heaviest_peak_height}, {heaviest_peak_hash}")
            # Check which peers are updated to this height

            peers = []
            coroutines = []
            for peer in self.server.all_connections.values():
                if peer.connection_type == NodeType.FULL_NODE:
                    peers.append(peer.peer_node_id)
                    coroutines.append(
                        peer.request_block(
                            full_node_protocol.RequestBlock(uint32(heaviest_peak_height), True), timeout=10
                        )
                    )
            for i, target_peak_response in enumerate(await asyncio.gather(*coroutines)):
                if target_peak_response is not None and isinstance(target_peak_response, RespondBlock):
                    self.sync_store.peer_has_block(
                        heaviest_peak_hash, peers[i], heaviest_peak_weight, heaviest_peak_height, False
                    )
            # TODO: disconnect from peer which gave us the heaviest_peak, if nobody has the peak

            peer_ids: Set[bytes32] = self.sync_store.get_peers_that_have_peak([heaviest_peak_hash])
            peers_with_peak: List = [c for c in self.server.all_connections.values() if c.peer_node_id in peer_ids]

            # Request weight proof from a random peer
            self.log.info(f"Total of {len(peers_with_peak)} peers with peak {heaviest_peak_height}")
            weight_proof_peer = random.choice(peers_with_peak)
            self.log.info(
                f"Requesting weight proof from peer {weight_proof_peer.peer_host} up to height"
                f" {heaviest_peak_height}"
            )

            if self.blockchain.get_peak() is not None and heaviest_peak_weight <= self.blockchain.get_peak().weight:
                raise ValueError("Not performing sync, already caught up.")

            wp_timeout = 360
            if "weight_proof_timeout" in self.config:
                wp_timeout = self.config["weight_proof_timeout"]
            self.log.debug(f"weight proof timeout is {wp_timeout} sec")
            request = full_node_protocol.RequestProofOfWeight(heaviest_peak_height, heaviest_peak_hash)
            response = await weight_proof_peer.request_proof_of_weight(request, timeout=wp_timeout)

            # Disconnect from this peer, because they have not behaved properly
            if response is None or not isinstance(response, full_node_protocol.RespondProofOfWeight):
                await weight_proof_peer.close(600)
                raise RuntimeError(f"Weight proof did not arrive in time from peer: {weight_proof_peer.peer_host}")
            if response.wp.recent_chain_data[-1].reward_chain_block.height != heaviest_peak_height:
                await weight_proof_peer.close(600)
                raise RuntimeError(f"Weight proof had the wrong height: {weight_proof_peer.peer_host}")
            if response.wp.recent_chain_data[-1].reward_chain_block.weight != heaviest_peak_weight:
                await weight_proof_peer.close(600)
                raise RuntimeError(f"Weight proof had the wrong weight: {weight_proof_peer.peer_host}")

            # dont sync to wp if local peak is heavier,
            # dont ban peer, we asked for this peak
            current_peak = self.blockchain.get_peak()
            if current_peak is not None:
                if response.wp.recent_chain_data[-1].reward_chain_block.weight <= current_peak.weight:
                    raise RuntimeError(f"current peak is heavier than Weight proof peek: {weight_proof_peer.peer_host}")

            try:
                validated, fork_point, summaries = await self.weight_proof_handler.validate_weight_proof(response.wp)
            except Exception as e:
                await weight_proof_peer.close(600)
                raise ValueError(f"Weight proof validation threw an error {e}")

            if not validated:
                await weight_proof_peer.close(600)
                raise ValueError("Weight proof validation failed")

            self.log.info(f"Re-checked peers: total of {len(peers_with_peak)} peers with peak {heaviest_peak_height}")
            self.sync_store.set_sync_mode(True)
            self._state_changed("sync_mode")
            # Ensures that the fork point does not change
            async with self.blockchain.lock:
                await self.blockchain.warmup(fork_point)
                await self.sync_from_fork_point(fork_point, heaviest_peak_height, heaviest_peak_hash, summaries)
        except asyncio.CancelledError:
            self.log.warning("Syncing failed, CancelledError")
        except Exception as e:
            tb = traceback.format_exc()
            self.log.error(f"Error with syncing: {type(e)}{tb}")
        finally:
            if self._shut_down:
                return None
            await self._finish_sync()
```
&emsp;&emsp;在_sync方法中，可以从注解中分析：

执行区块链到峰值的完全同步。

- 等几秒钟，让其它对等节点把它们的峰值发给当前节点

- 选择最重的峰值，并向拥有该峰值的同行请求重量证明

— 验证权重证明，无效时断开与对等体的连接

- 找到分叉点，看看从哪里开始下载块

- 批量下载块(并并行)，并一次验证一个

- 断开提供无效块或没有块的对等体

&emsp;&emsp;最后，同步的区块数据，会以sqlite文件存储在本地的磁盘中。其中，配置文件在initial-config.yaml中定义。目录默认在当前用户的.chia文件，需要显示隐藏文件才能看到(deepin系统)。

[80. 同步区块：多节点如何统一历史数据](https://zhuanlan.zhihu.com/p/101518970)

## p盘原理


## poc证明

参考：
[奇亚空间证明论文](https://www.chia.net/assets/Chia_Proof_of_Space_Construction_v1.1.pdf)


# 矿池开发

[Burstpool written in GO](https://github.com/PoC-Consortium/Nogrod)
[Scavenger - PoC miner in Rust](https://github.com/PoC-Consortium/scavenger)
[ChiaPool in C#](https://github.com/Playwo/ChiaPool)