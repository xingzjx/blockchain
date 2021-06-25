# POSDAO 共识算法学习指南

## 简介

POSDAO 是一种作为分散自治组织（DAO）实施的股权证明（POS）算法。它旨在为公共链提供分散，公平和节能的共识。该算法用作一组用Solidity编写的智能合约。 POSDAO使用通用

BFT共识协议实现，例如具有提议者节点和概率最终性的授权回合（AuRa），或具有无领导性和即时终结性的Honey Badger BFT（HBBFT）。激励验证者通过可配置的奖励结构来表现出

网络的最佳利益。该算法提供了Sybil控制机制，用于管理一组验证器，分发奖励，以及报告和惩罚恶意验证器。作者提供了参考POSDAO实现xDai DPOS，它使用xDai作为稳定的交易硬

币和代表性的ERC20令牌（DPOS）作为赌注令牌。参考实现在以太坊1.0侧链上起作用并利用AuRa共识协议。

## 代码实现

POSDAO合约已实现AuRa机制，并没有实现HBBFT。

[POSDAO 合约实现](https://github.com/poanetwork/posdao-contracts)

[HBBFT 协议论文](https://eprint.iacr.org/2016/199.pdf)


## 算法白皮书

[POSDAO white paper](https://forum.poa.network/t/posdao-white-paper/2208)


## 应用场景

xDai 采用了该共识机制。

xDai 客户端实现：

- OpenEthereum (previously Parity): Rust Client
- Nethermind: .NET Client 

## 桥接合约

https://github.com/poanetwork/tokenbridge-contracts