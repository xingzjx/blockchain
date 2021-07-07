# Bee源码解读：智能合约概要

- [Bee源码解读：智能合约概要](#bee源码解读智能合约概要)
  - [前言](#前言)
  - [Bzz和xBzz](#bzz和xbzz)
  - [Bandwidth price oracle](#bandwidth-price-oracle)
  - [Postage stamp](#postage-stamp)
  - [Bounding cure](#bounding-cure)

## 前言

Swarm 项目和其它的区块链存储项目有很大不同，比如　Filecoin，Swarm 是构建在以太坊二层网络的去中心化存储项目，Layer2　的核心实现用到了智能合约，下面以　Swarm 的官方实现　Bee　源码为例，概述用到的智能合约。

## Bzz和xBzz

Bzz　是部署在以太坊上的合约，xBzz部署在　xDai　网络的合约，其中，Bzz　桥接到了　xDai网络。

Bzz　桥接到　xDai　：https://omni.xdaichain.com/bridge

参考：

[Bzz 完整合约代码](https://etherscan.io/address/0x19062190b1925b5b6689d7073fdfc8c2976ef8cb#code)

## Bandwidth price oracle

带宽定价合约：

```javascript

// ... 省略　...

// File src/PriceOracle.sol

// : BSD-3-Clause
pragma solidity ^0.8.4;

/**
 * @title PriceOracle contract
 * @author The Swarm Authors
 * @dev The price oracle contract keeps track of the current prices for settlement in swap accounting.
 */
contract PriceOracle is Ownable {
    /**
     * @dev Emitted when the price is updated.
     */
    event PriceUpdate(uint256 price);
    /**
     * @dev Emitted when the cheque value deduction amount is updated.
     */
    event ChequeValueDeductionUpdate(uint256 chequeValueDeduction);

    // current price in PLUR per accounting unit
    uint256 public price;
    // value deducted from first received cheque from a peer in PLUR
    uint256 public chequeValueDeduction;

    constructor(uint256 _price, uint256 _chequeValueDeduction) {
        price = _price;
        chequeValueDeduction = _chequeValueDeduction;
    }

    /**
     * @notice Returns the current price in PLUR per accounting unit and the current cheque value deduction amount.
     */
    function getPrice() external view returns (uint256, uint256) {
        return (price, chequeValueDeduction);
    }

    /**
     * @notice Update the price. Can only be called by the owner.
     * @param newPrice the new price
     */
    function updatePrice(uint256 newPrice) external onlyOwner {
        price = newPrice;
        emit PriceUpdate(price);
    }

    /**
     * @notice Update the cheque value deduction amount. Can only be called by the owner.
     * @param newChequeValueDeduction the new cheque value deduction amount
     */
    function updateChequeValueDeduction(uint256 newChequeValueDeduction)
        external
        onlyOwner
    {
        chequeValueDeduction = newChequeValueDeduction;
        emit ChequeValueDeductionUpdate(chequeValueDeduction);
    }
}

```

参考：

[完整定价合约代码](https://blockscout.com/xdai/mainnet/address/0x0FDc5429C50e2a39066D8A94F3e2D2476fcc3b85/contracts)

## Postage stamp

邮票：是

```javascript

// ... 省略　...

/**
 * @title PostageStamp contract
 * @author The Swarm Authors
 * @dev The postage stamp contracts allows users to create and manage postage stamp batches.
 */
contract PostageStamp is AccessControl, Pausable {
    using SafeMath for uint256;
    /**
     * @dev Emitted when a new batch is created.
     */
    event BatchCreated(
        bytes32 indexed batchId,
        uint256 totalAmount,
        uint256 normalisedBalance,
        address owner,
        uint8 depth,
        uint8 bucketDepth,
        bool immutableFlag
    );

    /**
     * @dev Emitted when an existing batch is topped up.
     */
    event BatchTopUp(bytes32 indexed batchId, uint256 topupAmount, uint256 normalisedBalance);

    /**
     * @dev Emitted when the depth of an existing batch increases.
     */
    event BatchDepthIncrease(bytes32 indexed batchId, uint8 newDepth, uint256 normalisedBalance);

    /**
     *@dev Emitted on every price update.
     */
    event PriceUpdate(uint256 price);

    struct Batch {
        // Owner of this batch (0 if not valid).
        address owner;
        // Current depth of this batch.
        uint8 depth;
        // Whether this batch is immutable
        bool immutableFlag;
        // Normalised balance per chunk.
        uint256 normalisedBalance;
    }

    // The role allowed to increase totalOutPayment
    bytes32 public constant PRICE_ORACLE_ROLE = keccak256("PRICE_ORACLE");
    // The role allowed to pause
    bytes32 public constant PAUSER_ROLE = keccak256("PAUSER_ROLE");

    // Associate every batch id with batch data.
    mapping(bytes32 => Batch) public batches;

    // The address of the BZZ ERC20 token this contract references.
    address public bzzToken;
    // The total out payment per chunk
    uint256 public totalOutPayment;

    // the price from the last update
    uint256 public lastPrice;
    // the block at which the last update occured
    uint256 public lastUpdatedBlock;

    /**
     * @param _bzzToken The ERC20 token address to reference in this contract.
     */
    constructor(address _bzzToken) {
        bzzToken = _bzzToken;
        _setupRole(DEFAULT_ADMIN_ROLE, msg.sender);
        _setupRole(PAUSER_ROLE, msg.sender);
    }

    /**
     * @notice Create a new batch.
     * @dev At least `_initialBalancePerChunk*2^depth` number of tokens need to be preapproved for this contract.
     * @param _owner The owner of the new batch.
     * @param _initialBalancePerChunk The initial balance per chunk of the batch.
     * @param _depth The initial depth of the new batch.
     * @param _nonce A random value used in the batch id derivation to allow multiple batches per owner.
     * @param _immutable Whether the batch is mutable.
     */
    function createBatch(
        address _owner,
        uint256 _initialBalancePerChunk,
        uint8 _depth,
        uint8 _bucketDepth,
        bytes32 _nonce,
        bool _immutable
    ) external whenNotPaused {
        require(_owner != address(0), "owner cannot be the zero address");
        // bucket depth should be non-zero and smaller than the depth
        require(_bucketDepth != 0 && _bucketDepth < _depth, "invalid bucket depth");
        // Derive batchId from msg.sender to ensure another party cannot use the same batch id and frontrun us.
        bytes32 batchId = keccak256(abi.encode(msg.sender, _nonce));
        require(batches[batchId].owner == address(0), "batch already exists");

        // per chunk balance times the batch size is what we need to transfer in
        uint256 totalAmount = _initialBalancePerChunk.mul(1 << _depth);
        require(ERC20(bzzToken).transferFrom(msg.sender, address(this), totalAmount), "failed transfer");

        uint256 normalisedBalance = currentTotalOutPayment().add(_initialBalancePerChunk);

        batches[batchId] = Batch({
            owner: _owner,
            depth: _depth,
            immutableFlag: _immutable,
            normalisedBalance: normalisedBalance
        });

        emit BatchCreated(batchId, totalAmount, normalisedBalance, _owner, _depth, _bucketDepth, _immutable);
    }

    /**
     * @notice Top up an existing batch.
     * @dev At least `topupAmount*2^depth` number of tokens need to be preapproved for this contract.
     * @param _batchId The id of the existing batch.
     * @param _topupAmountPerChunk The amount of additional tokens to add per chunk.
     */
    function topUp(bytes32 _batchId, uint256 _topupAmountPerChunk) external whenNotPaused {
        Batch storage batch = batches[_batchId];
        require(batch.owner != address(0), "batch does not exist");
        require(batch.normalisedBalance > currentTotalOutPayment(), "batch already expired");

        // per chunk topup amount times the batch size is what we need to transfer in
        uint256 totalAmount = _topupAmountPerChunk.mul(1 << batch.depth);
        require(ERC20(bzzToken).transferFrom(msg.sender, address(this), totalAmount), "failed transfer");

        batch.normalisedBalance = batch.normalisedBalance.add(_topupAmountPerChunk);

        emit BatchTopUp(_batchId, totalAmount, batch.normalisedBalance);
    }

    /**
     * @notice Increase the depth of an existing batch.
     * @dev Can only be called by the owner of the batch.
     * @param _batchId the id of the existing batch
     * @param _newDepth the new (larger than the previous one) depth for this batch
     */
    function increaseDepth(bytes32 _batchId, uint8 _newDepth) external whenNotPaused {
        Batch storage batch = batches[_batchId];
        require(batch.owner == msg.sender, "not batch owner");
        require(!batch.immutableFlag, "batch is immutable");
        require(_newDepth > batch.depth, "depth not increasing");
        require(batch.normalisedBalance > currentTotalOutPayment(), "batch already expired");

        uint8 depthChange = _newDepth - batch.depth;
        // divide by the change in batch size (2^depthChange)
        uint256 newRemainingBalance = remainingBalance(_batchId).div(1 << depthChange);

        batch.depth = _newDepth;
        batch.normalisedBalance = currentTotalOutPayment().add(newRemainingBalance);

        emit BatchDepthIncrease(_batchId, _newDepth, batch.normalisedBalance);
    }

    /**
     * @notice Returns the per chunk balance not used up yet
     * @param _batchId the id of the existing batch
     */
    function remainingBalance(bytes32 _batchId) public view returns (uint256) {
        Batch storage batch = batches[_batchId];
        require(batch.owner != address(0), "batch does not exist");
        return batch.normalisedBalance.sub(currentTotalOutPayment());
    }

    /**
     * @notice set a new price
     * @dev can only be called by the price oracle
     * @param _price the new price
     */
    function setPrice(uint256 _price) external {
        require(hasRole(PRICE_ORACLE_ROLE, msg.sender), "only price oracle can set the price");

        // if there was a last price, charge for the time since the last update with the last price
        if (lastPrice != 0) {
            totalOutPayment = currentTotalOutPayment();
        }

        lastPrice = _price;
        lastUpdatedBlock = block.number;

        emit PriceUpdate(_price);
    }

    /**
     * @notice Returns the current total outpayment
     */
    function currentTotalOutPayment() public view returns (uint256) {
        uint256 blocks = block.number - lastUpdatedBlock;
        uint256 increaseSinceLastUpdate = lastPrice.mul(blocks);
        return totalOutPayment.add(increaseSinceLastUpdate);
    }

    /**
     * @notice Pause the contract. The contract is provably stopped by renouncing the pauser role and the admin role after pausing
     * @dev can only be called by the pauser when not paused
     */
    function pause() public {
        require(hasRole(PAUSER_ROLE, msg.sender), "only pauser can pause the contract");
        _pause();
    }

    /**
     * @notice Unpause the contract.
     * @dev can only be called by the pauser when paused
     */
    function unPause() public {
        require(hasRole(PAUSER_ROLE, msg.sender), "only pauser can unpause the contract");
        _unpause();
    }
}

```

参考：

[邮票合约完整代码](https://blockscout.com/xdai/mainnet/address/0x6a1A21ECA3aB28BE85C7Ba22b2d6eAE5907c900E/contracts)

## Bounding cure

联合曲线合约：联合曲线模型可以理解为描述“代币买卖价格”与“代币发行总量”之间的函数关系。可以由智能合约以去中心化的方式自动执行，这种函数关系确定了代币价格和代币供应之间的关系。

参考：

[Swarm Bond Curve分析](https://zhuanlan.zhihu.com/p/381605708?utm_source=wechat_timeline&utm_medium=social&s_r=0&wechatShare=1)