# 智能合约学习指南

前言：在[精通以太坊](https://github.com/inoutcode/ethereum_book)中，已经详细讲解了智能合约的相关技术，下面选取其中几个章节作为解读。


## 智能合约

计算机程序：智能合约只是计算机程序。合约这个词在这方面没有法律意义。 不可变的：一旦部署，智能合约的代码不能改变。与传统软件不同，修改智能合约的唯一方法是部署新实例。 确定性的：智能合约的结果对于运行它的每个人来说都是一样的，包括调用它们的交易的上下文，以及执行时以太坊区块链的状态。 EVM上下文：智能合约以非常有限的执行上下文运行。他们可以访问自己的状态，调用它们的交易的上下文以及有关最新块的一些信息。 去中心化的世界计算机：EVM在每个以太坊节点上作为本地实例运行，但由于EVM的所有实例都在相同的初始状态下运行并产生相同的最终状态，因此整个系统作为单台世界计算机运行。

建议学习[精通以太坊之智能合约](https://github.com/inoutcode/ethereum_book/blob/master/%E7%AC%AC%E5%85%AB%E7%AB%A0.asciidoc)章节。

看完整篇后，对智能合约会有有一个整体的认知，然后介绍了怎么编写一个智能合约，并且部署智能合约。

[source,solidity,linenums]
```solidity
// Our first contract is a faucet!
contract Faucet {

    // Give out ether to anyone who asks
    function withdraw(uint withdraw_amount) public {

        // Limit withdrawal amount
        require(withdraw_amount <= 100000000000000000);

        // Send the amount to the address that requested it
        msg.sender.transfer(withdraw_amount);
    }

    // Accept any incoming amount
    function () public payable {}

}
```

然后，可以使用　remix　编译器，编译和部署智能合约。

## 开发工具，框架和库

### 开发工具

常见的智能合约开发工具如下：

- truffle
  
truffle　是一个使用非常多的开发工具。

truffle　官方文档：https://www.trufflesuite.com/

其中，ganache　工具可以部署一个图形界面的以太坊私链，方便开发环境使用。

- remix

remix 是一个solidity在线编译工具。

remix　官方文档：https://remix-ide.readthedocs.io/en/latest/

- hardhat

这个工具在本书里面没有介绍，但是使用得比较多的工具。

官方文档：https://hardhat.org/getting-started/#overview

## 第三方库

- OpenZeppelin

OpenZeppelin　是一个开源合约库，里面包括ERC20，合约安全工具等。

官方地址：https://github.com/OpenZeppelin/openzeppelin-contracts

## Token

### 定义

基于区块链的Token将这个词重新定义为基于区块链的抽象概念，可以被拥有，并代表资产，货币或访问权。

### ERC20 Token

ERC20 Token 标准:第一个标准由Fabian Vogelsteller于2015年11月引入，作为以太坊征求意见（ERC）。它被自动分配了GitHub发行号码20，从而获得了名字“ERC20 Token”。绝大多数Token目前都基于ERC20。ERC20征求意见最终成为以太坊改进建议EIP20，但大多仍以原名ERC20提及。你可以在这里阅读标准：

https://github.com/ethereum/EIPs/blob/master/EIPS/eip-20.md

[source,solidity,linenums]
```solidity
contract ERC20 {
   function totalSupply() constant returns (uint theTotalSupply);
   function balanceOf(address _owner) constant returns (uint balance);
   function transfer(address _to, uint _value) returns (bool success);
   function transferFrom(address _from, address _to, uint _value) returns (bool success);
   function approve(address _spender, uint _value) returns (bool success);
   function allowance(address _owner, address _spender) constant returns (uint remaining);
   event Transfer(address indexed _from, address indexed _to, uint _value);
   event Approval(address indexed _owner, address indexed _spender, uint _value);
}
```

### 发行 ERC20 Token

https://gitee.com/xingzjx/blockchain/blob/master/ethereum/publish%20toekn%20guide.md


### ERC721

不可替代的Token（契据）标准，也就是现在非常火的 NFT，在后面的系列会进一步深入。

ERC721标准：

[source,solidity,linenums]
```solidity
interface ERC721 /* is ERC165 */ {
    event Transfer(address indexed _from, address indexed _to, uint256 _deedId);
    event Approval(address indexed _owner, address indexed _approved, uint256 _deedId);
    event ApprovalForAll(address indexed _owner, address indexed _operator, bool _approved);

    function balanceOf(address _owner) external view returns (uint256 _balance);
    function ownerOf(uint256 _deedId) external view returns (address _owner);
    function transfer(address _to, uint256 _deedId) external payable;
    function transferFrom(address _from, address _to, uint256 _deedId) external payable;
    function approve(address _approved, uint256 _deedId) external payable;
    function setApprovalForAll(address _operateor, boolean _approved) payable;
    function supportsInterface(bytes4 interfaceID) external view returns (bool);
}
```

可选接口：元数据

[source,solidity,linenums]
```solidity
interface ERC721Metadata /* is ERC721 */ {
    function name() external pure returns (string _name);
    function symbol() external pure returns (string _symbol);
    function deedUri(uint256 _deedId) external view returns (string _deedUri);
}
```

可选接口：枚举

[source,solidity,linenums]
```solidity
interface ERC721Enumerable /* is ERC721 */ {
    function totalSupply() external view returns (uint256 _count);
    function deedByIndex(uint256 _index) external view returns (uint256 _deedId);
    function countOfOwners() external view returns (uint256 _count);
    function ownerByIndex(uint256 _index) external view returns (address _owner);
    function deedOfOwnerByIndex(address _owner, uint256 _index) external view returns (uint256 _deedId);
}
```

## 去中心应用

与传统应用程序不同，去中心化应用（DApp）不仅属于单个提供者或服务器，而是整个栈将在P2P网络上以分布式方式部署和操作。

典型的DApp栈包括前端，后端和数据存储。创建DApp有许多优点，典型集中式架构无法提供：

1）弹性：在智能合约上编写业务逻辑意味着DApp后端将在区块链上完全分发和管理。与在中央服务器上部署应用程序不同，DApp不会有停机时间，只要区块链仍在运行，它就会继续存在。

2）透明性：DApp的开源特性允许任何人分叉代码并在区块链上运行相同的应用程序。同样，任何与区块链的互动都将永久存储，任何拥有区块链副本的人都可以获得对它的访问权限。值得注意的是，可能无法将字节码反编译为源码并完全理解合约的代码。寻求提供合约行为完全透明的开发人员必须发布供用户阅读，编译和验证的源代码。

3）抗审查：只要用户可以访问以太坊节点，用户将始终能够与DApp交互而不受集中机构控制的干扰。一旦在网络上部署代码，任何服务提供商，甚至智能合约的所有者都不能更改代码。

使用　react 构建一个　dapp　案例：https://www.trufflesuite.com/boxes/react


## 总结

通过以上的学习，基本可以开发一个简单的智能合约了。随着更深入的学习，可以研究目前市面上知名的项目，比如，uniswap和swarm，xdai中的合约。另一方面，可以研究以太坊EVM的实现机制。

