# 翻译：ERC721 官方文档

- [翻译：ERC721 官方文档](#翻译erc721-官方文档)
  - [简介](#简介)
  - [抽象](#抽象)
  - [动机](#动机)
  - [规格](#规格)
    - [注意事项](#注意事项)
  - [基本原理](#基本原理)
  - [向后兼容](#向后兼容)
  - [测试案例](#测试案例)
  - [实现](#实现)
  - [参考](#参考)
  - [版本](#版本)

```

以太坊提案: 721
标题: ERC-721 非同质化代币标准
作者: William Entriken <github.com@phor.net>, Dieter Shirley <dete@axiomzen.co>, Jacob Evans <jacob@dekz.net>, Nastassia Sachs <nastassia.sachs@protonmail.com>
讨论: https://github.com/ethereum/eips/issues/721
类型: 标准跟踪
类别: ERC
状态: Final
创建时间: 2018-01-24
需求: 165

```

## 简介

不可替代代币的标准接口，也称为契约。

## 抽象

以下标准允许在智能合约中为 NFT 实施标准 API。该标准提供了跟踪和传输 NFT 的基本功能。

我们考虑了 NFT 由个人拥有和交易以及委托给第三方经纪人/钱包/拍卖商（“运营商”）的用例。NFT 可以代表对数字或物理资产的所有权。我们考虑了多元化的资产领域，我们知道您会梦想更多：

- 有形财产——房屋、独特的艺术品
- 虚拟收藏品 — 小猫的独特照片、收藏卡
- “负值”资产——贷款、负担和其他责任

一般来说，所有的房子都是不同的，没有两只小猫是一样的。NFT 是可区分的，您必须分别跟踪每个的所有权。

## 动机

标准接口允许钱包/经纪人/拍卖应用程序与以太坊上的任何 NFT 一起使用。我们提供简单的 ERC-721 智能合约以及跟踪任意大量NFT 的合约。下面讨论其他应用。

该标准的灵感来自 ERC-20 代币标准，并建立在自 EIP-20 创建以来两年的经验之上。EIP-20 不足以跟踪 NFT，因为每个资产都是不同的（不可替代的），而每个数量的代币都是相同的（可替代的）。

本标准与 EIP-20 之间的差异在下面进行了检查。

## 规格

本文档中的关键词“必须”、“不得”、“需要”、“应该”、“不应”、“应该”、“不应该”、“推荐”、“可以”和“可选”是按照 RFC 2119 中的描述进行解释。

**每个符合 ERC-721 的合约都必须实现ERC721和ERC165接口**（受以下“警告”的约束）：

```solidity

pragma solidity ^0.4.20;

/// @title ERC-721 Non-Fungible Token Standard
/// @dev See https://eips.ethereum.org/EIPS/eip-721
///  Note: the ERC-165 identifier for this interface is 0x80ac58cd.
interface ERC721 /* is ERC165 */ {
    /// @dev This emits when ownership of any NFT changes by any mechanism.
    ///  This event emits when NFTs are created (`from` == 0) and destroyed
    ///  (`to` == 0). Exception: during contract creation, any number of NFTs
    ///  may be created and assigned without emitting Transfer. At the time of
    ///  any transfer, the approved address for that NFT (if any) is reset to none.
    event Transfer(address indexed _from, address indexed _to, uint256 indexed _tokenId);

    /// @dev This emits when the approved address for an NFT is changed or
    ///  reaffirmed. The zero address indicates there is no approved address.
    ///  When a Transfer event emits, this also indicates that the approved
    ///  address for that NFT (if any) is reset to none.
    event Approval(address indexed _owner, address indexed _approved, uint256 indexed _tokenId);

    /// @dev This emits when an operator is enabled or disabled for an owner.
    ///  The operator can manage all NFTs of the owner.
    event ApprovalForAll(address indexed _owner, address indexed _operator, bool _approved);

    /// @notice Count all NFTs assigned to an owner
    /// @dev NFTs assigned to the zero address are considered invalid, and this
    ///  function throws for queries about the zero address.
    /// @param _owner An address for whom to query the balance
    /// @return The number of NFTs owned by `_owner`, possibly zero
    function balanceOf(address _owner) external view returns (uint256);

    /// @notice Find the owner of an NFT
    /// @dev NFTs assigned to zero address are considered invalid, and queries
    ///  about them do throw.
    /// @param _tokenId The identifier for an NFT
    /// @return The address of the owner of the NFT
    function ownerOf(uint256 _tokenId) external view returns (address);

    /// @notice Transfers the ownership of an NFT from one address to another address
    /// @dev Throws unless `msg.sender` is the current owner, an authorized
    ///  operator, or the approved address for this NFT. Throws if `_from` is
    ///  not the current owner. Throws if `_to` is the zero address. Throws if
    ///  `_tokenId` is not a valid NFT. When transfer is complete, this function
    ///  checks if `_to` is a smart contract (code size > 0). If so, it calls
    ///  `onERC721Received` on `_to` and throws if the return value is not
    ///  `bytes4(keccak256("onERC721Received(address,address,uint256,bytes)"))`.
    /// @param _from The current owner of the NFT
    /// @param _to The new owner
    /// @param _tokenId The NFT to transfer
    /// @param data Additional data with no specified format, sent in call to `_to`
    function safeTransferFrom(address _from, address _to, uint256 _tokenId, bytes data) external payable;

    /// @notice Transfers the ownership of an NFT from one address to another address
    /// @dev This works identically to the other function with an extra data parameter,
    ///  except this function just sets data to "".
    /// @param _from The current owner of the NFT
    /// @param _to The new owner
    /// @param _tokenId The NFT to transfer
    function safeTransferFrom(address _from, address _to, uint256 _tokenId) external payable;

    /// @notice Transfer ownership of an NFT -- THE CALLER IS RESPONSIBLE
    ///  TO CONFIRM THAT `_to` IS CAPABLE OF RECEIVING NFTS OR ELSE
    ///  THEY MAY BE PERMANENTLY LOST
    /// @dev Throws unless `msg.sender` is the current owner, an authorized
    ///  operator, or the approved address for this NFT. Throws if `_from` is
    ///  not the current owner. Throws if `_to` is the zero address. Throws if
    ///  `_tokenId` is not a valid NFT.
    /// @param _from The current owner of the NFT
    /// @param _to The new owner
    /// @param _tokenId The NFT to transfer
    function transferFrom(address _from, address _to, uint256 _tokenId) external payable;

    /// @notice Change or reaffirm the approved address for an NFT
    /// @dev The zero address indicates there is no approved address.
    ///  Throws unless `msg.sender` is the current NFT owner, or an authorized
    ///  operator of the current owner.
    /// @param _approved The new approved NFT controller
    /// @param _tokenId The NFT to approve
    function approve(address _approved, uint256 _tokenId) external payable;

    /// @notice Enable or disable approval for a third party ("operator") to manage
    ///  all of `msg.sender`'s assets
    /// @dev Emits the ApprovalForAll event. The contract MUST allow
    ///  multiple operators per owner.
    /// @param _operator Address to add to the set of authorized operators
    /// @param _approved True if the operator is approved, false to revoke approval
    function setApprovalForAll(address _operator, bool _approved) external;

    /// @notice Get the approved address for a single NFT
    /// @dev Throws if `_tokenId` is not a valid NFT.
    /// @param _tokenId The NFT to find the approved address for
    /// @return The approved address for this NFT, or the zero address if there is none
    function getApproved(uint256 _tokenId) external view returns (address);

    /// @notice Query if an address is an authorized operator for another address
    /// @param _owner The address that owns the NFTs
    /// @param _operator The address that acts on behalf of the owner
    /// @return True if `_operator` is an approved operator for `_owner`, false otherwise
    function isApprovedForAll(address _owner, address _operator) external view returns (bool);
}

interface ERC165 {
    /// @notice Query if a contract implements an interface
    /// @param interfaceID The interface identifier, as specified in ERC-165
    /// @dev Interface identification is specified in ERC-165. This function
    ///  uses less than 30,000 gas.
    /// @return `true` if the contract implements `interfaceID` and
    ///  `interfaceID` is not 0xffffffff, `false` otherwise
    function supportsInterface(bytes4 interfaceID) external view returns (bool);
}
```

如果钱包/经纪人/拍卖应用程序接受安全转账，它必须实现**钱包接口**。

```solidity
/// @dev Note: the ERC-165 identifier for this interface is 0x150b7a02.
interface ERC721TokenReceiver {
    /// @notice Handle the receipt of an NFT
    /// @dev The ERC721 smart contract calls this function on the recipient
    ///  after a `transfer`. This function MAY throw to revert and reject the
    ///  transfer. Return of other than the magic value MUST result in the
    ///  transaction being reverted.
    ///  Note: the contract address is always the message sender.
    /// @param _operator The address which called `safeTransferFrom` function
    /// @param _from The address which previously owned the token
    /// @param _tokenId The NFT identifier which is being transferred
    /// @param _data Additional data with no specified format
    /// @return `bytes4(keccak256("onERC721Received(address,address,uint256,bytes)"))`
    ///  unless throwing
    function onERC721Received(address _operator, address _from, uint256 _tokenId, bytes _data) external returns(bytes4);
}

```

该**元数据**扩展是可选的ERC-721智能合同（见“警告”，下同）。这允许询问您的智能合约的名称以及有关您的 NFT 代表的资产的详细信息。

```solidity
/// @title ERC-721 Non-Fungible Token Standard, optional metadata extension
/// @dev See https://eips.ethereum.org/EIPS/eip-721
///  Note: the ERC-165 identifier for this interface is 0x5b5e139f.
interface ERC721Metadata /* is ERC721 */ {
    /// @notice A descriptive name for a collection of NFTs in this contract
    function name() external view returns (string _name);

    /// @notice An abbreviated name for NFTs in this contract
    function symbol() external view returns (string _symbol);

    /// @notice A distinct Uniform Resource Identifier (URI) for a given asset.
    /// @dev Throws if `_tokenId` is not a valid NFT. URIs are defined in RFC
    ///  3986. The URI may point to a JSON file that conforms to the "ERC721
    ///  Metadata JSON Schema".
    function tokenURI(uint256 _tokenId) external view returns (string);
}
```

这是上面引用的 “ERC721 Metadata JSON Schema” 。

```json
{
    "title": "Asset Metadata",
    "type": "object",
    "properties": {
        "name": {
            "type": "string",
            "description": "Identifies the asset to which this NFT represents"
        },
        "description": {
            "type": "string",
            "description": "Describes the asset to which this NFT represents"
        },
        "image": {
            "type": "string",
            "description": "A URI pointing to a resource with mime type image/* representing the asset to which this NFT represents. Consider making any images at a width between 320 and 1080 pixels and aspect ratio between 1.91:1 and 4:5 inclusive."
        }
    }
}
```

ERC-721 智能合约的**枚举扩展**是可选的（请参阅下面的“注意事项”）。这允许您的合约发布其完整的 NFT 列表并使它们可被发现。

```solidity
/// @title ERC-721 Non-Fungible Token Standard, optional enumeration extension
/// @dev See https://eips.ethereum.org/EIPS/eip-721
///  Note: the ERC-165 identifier for this interface is 0x780e9d63.
interface ERC721Enumerable /* is ERC721 */ {
    /// @notice Count NFTs tracked by this contract
    /// @return A count of valid NFTs tracked by this contract, where each one of
    ///  them has an assigned and queryable owner not equal to the zero address
    function totalSupply() external view returns (uint256);

    /// @notice Enumerate valid NFTs
    /// @dev Throws if `_index` >= `totalSupply()`.
    /// @param _index A counter less than `totalSupply()`
    /// @return The token identifier for the `_index`th NFT,
    ///  (sort order not specified)
    function tokenByIndex(uint256 _index) external view returns (uint256);

    /// @notice Enumerate NFTs assigned to an owner
    /// @dev Throws if `_index` >= `balanceOf(_owner)` or if
    ///  `_owner` is the zero address, representing invalid NFTs.
    /// @param _owner An address where we are interested in NFTs owned by them
    /// @param _index A counter less than `balanceOf(_owner)`
    /// @return The token identifier for the `_index`th NFT assigned to `_owner`,
    ///   (sort order not specified)
    function tokenOfOwnerByIndex(address _owner, uint256 _index) external view returns (uint256);
}
```

### 注意事项

0.4.20 Solidity 接口语法的表达能力不足以记录 ERC-721 标准。符合 ERC-721 的合约还必须遵守以下内容：

- Solidity issue #3412: 述接口包括每个函数的显式可变性保证。可变性保证从弱到强依次为：payable、隐式不可支付view、 和pure。您的实现必须满足此接口中的可变性保证，并且您可以满足更强的保证。例如，payable此接口中的函数可能在您的合约中实现为 nonpayble（未指定状态可变性）。我们预计以后的 Solidity 版本将允许您从该接口继承更严格的合约，但 0.4.20 版本的解决方法是您可以在从您的合约继承之前编辑此接口以添加更严格的可变性。
- Solidity issue #3419: 实施ERC721Metadata或ERC721Enumerable应也实施ERC721. ERC-721 实现了接口 ERC-165 的要求。
- Solidity issue #2330: 如果在本规范中显示了一个函数，external那么如果它使用public可见性，则合约将是合规的。作为 0.4.20 版本的解决方法，您可以public在从合同继承之前编辑此界面以切换到。
- Solidity issues #3494, #3544: this.*.selectorSolidity 将使用标记为警告，Solidity 的未来版本不会将此标记为错误。

*如果新版本的 Solidity 允许在代码中表达警告，那么这个 EIP 可能会更新并删除警告，这将等同于原始规范。*

## 基本原理

以太坊智能合约的许多拟议用途依赖于跟踪可区分的资产。现有或计划中的 NFT 示例包括 Decentraland 中的 LAND、CryptoPunks 中的同名朋克，以及使用 DMarket 或 EnjinCoin 等系统的游戏内物品。未来的用途包括跟踪现实世界的资产，如房地产（如 Ubitquity 或 Propy 等公司所设想的那样）。在这些情况中的每一种情况下，这些项目都不会作为分类帐中的数字“集中在一起”，而是每个资产都必须单独和原子地跟踪其所有权，这一点至关重要。不管这些资产的性质如何，如果我们有一个标准化的接口，允许跨职能的资产管理和销售平台，生态系统就会更加强大。

**“NFT”选词**

“NFT”几乎让所有接受调查的人都感到满意，并且广泛适用于可区分的数字资产的广泛领域。我们认识到“契约”对于本标准的某些应用（特别是物理属性）非常具有描述性。

*考虑的替代方案：可区分资产、所有权、代币、资产、股权、票据*

**NFT 标识符**

每个 NFT 都由`uint256ERC-721` 智能合约内的唯一ID 标识。该识别号码在合同有效期内不得更改。(`contract address`, `uint256 tokenId`)然后，该对将成为以太坊链上特定资产的全球唯一且完全合格的标识符。虽然一些 ERC-721 智能合约可能会发现从 ID 0 开始并为每个新的 NFT 简单地增加 1 很方便，但调用者不应假设 ID 号码对他们有任何特定模式，并且必须将 ID 视为“黑匣子”。另请注意，NFT 可能会失效（被销毁）。请参阅枚举函数以了解支持的枚举接口。

`uint256`的允许多种应用程序，因为 UUID 和 sha3 哈希可直接转换为`uint256`。

**转账机制**

ERC-721 standardizes a safe transfer function `safeTransferFrom` (overloaded with and without a `bytes` parameter) and an unsafe function `transferFrom`. Transfers may be initiated by:

ERC-721 标准化了安全转账函数`safeTransferFrom`（带`bytes`参数和不带参数的重载）和不安全函数transferFrom。转账可以由以下人员发起：

- NFT 的所有者
- NFT 的批准地址
- NFT当前所有者的授权运营商

此外，授权运营商可以为 NFT 设置批准的地址。这为钱包、经纪人和拍卖应用程序提供了一套强大的工具，以快速使用大量NFT。

transfer 和 accept 函数的文档只指定事务必须抛出的条件。您的实现也可能会导致其他情况。这允许实现获得有趣的结果：

- **如果合约暂停，则不允许转账** — 现有技术，CryptoKitties 部署的合约，第 611 行
- **将接收 NFT 的某些地址列入黑名单** — 现有技术，CryptoKitties 部署合约，第 565、566 行
- **禁止不安全的转账** — `transferFrom`抛出除非`_to`等于`msg.sender`或`countOf(_to)`非零或之前非零（因为这种情况是安全的）
- **收取费用的交易双方** — 需要支付调用`approve`用非零`_approved`如果它以前的零个地址，退回货款，如果在调用`approve`与零个地址，如果它以前是一个非零地址，调用任何时候要求支付传输函数，要求传输参数`_to`等于`msg.sender`，要求传输参数`_to`为 NFT 的批准地址

- **只读NFT注册表** — 总是抛出 `unsafeTransfer`, `transferFrom`, `approve` and `setApprovalForAll`

失败的交易将抛出，这是 ERC-223、ERC-677、ERC-827 和 OpenZeppelin 的 SafeERC20.sol 实现中确定的最佳实践。ERC-20 定义了一个`allowance`功能，这在调用时会导致问题，然后再修改为不同的数量，如 OpenZeppelin 问题 \#438。在 ERC-721 中，没有允许，因为每个 NFT 都是唯一的，数量是无或一个。因此，我们获得了 ERC-20 原始设计的好处，而没有后来发现的问题。

规范中不包括创建 NFT（“铸造”）和销毁 NFT（“燃烧”）。您的合同可以通过其他方式实施这些。请参阅`event`文档了解您在创建或销毁 NFT 时的责任。

我们质疑`operator`参数在`onERC721Received`是否必要。在我们可以想象的所有情况下，如果运营商很重要，那么该运营商可以将代币转移给他们自己然后发送——那么他们就是`from`地址。这似乎是人为的，因为我们认为运营商是代币的临时所有者（向他们自己转移是多余的）。当运营商发送代币时，是运营商自行操作，而不是代表代币持有者行事的运营商。这就是为什么运营商和之前的代币所有者对代币接收者都很重要。


*考虑的替代方案：只允许两步 ERC-20 风格的事务，要求传递函数从不抛出，要求所有函数返回一个指示操作成功的布尔值。*

**ERC-165 接口**

我们选择了标准接口检测 (ERC-165) 来公开 ERC-721 智能合约支持的接口。

未来的 EIP 可能会为合约创建一个全球接口注册。我们坚决支持这样的EIP，它将使您的ERC-721实施来实现`ERC721Enumerable`，`ERC721Metadata`通过委托给一个独立的合同或其他接口。

**Gas 和　复杂性** (关于枚举扩展)

该规范考虑了管理少量和任意大量NFT 的实现。如果您的应用程序能够增长，那么请避免在您的代码中使用 for/while 循环（请参阅 CryptoKitties 赏金问题 \#4）。这些表明您的合约可能无法扩展，并且 gas 成本会随着时间的推移而无限制地上升。

我们已将合约 XXXXERC721 部署到测试网，该合约实例化并跟踪 340282366920938463463374607431768211456 种不同的行为（2^128）。这足以将每个 IPV6 地址分配给一个以太坊账户所有者，或者跟踪几微米大小、总计为地球一半大小的纳米机器人的所有权。您可以从区块链中查询它。并且每个函数都比查询 ENS 消耗更少的 gas 。

此图清楚地表明：ERC-721 标准量表。

*考虑的替代方案：如果需要 for 循环，则删除资产枚举函数，从枚举函数返回 Solidity 数组类型。*

**隐私**

在动机部分确定的钱包/经纪人/拍卖商非常需要确定所有者拥有哪些 NFT。

考虑不可枚举的 NFT 用例可能会很有趣，例如财产所有权的私有注册表或部分私有注册表。但是，无法获得隐私，因为攻击者可以简单地 (!) 调用`ownerOf`所有可能的`tokenId`。

**元数据选择** (元数据扩展)

我们在元数据扩展中有所需`name`和`symbol`功能。我们审查的每个 token EIP 和草案（ERC-20、ERC-223、ERC-677、ERC-777、ERC-827）都包含这些功能。

我们提醒实现作者，空字符串是一个有效的响应`name` 和　`symbol`，如果您不同意这一机制的使用。我们还提醒大家，任何智能合约都可以使用与您的合约相同的名称和符号。客户如何确定哪些 ERC-721 智能合约是众所周知的（规范的）超出了本标准的范围。

提供了一种机制来将 NFT 与 URI 相关联。我们预计许多实现将利用这一点为每个 NFT 提供元数据。图片大小推荐来自Instagram，他们可能对图片可用性了解很多。URI 可以是可变的（即它会不时更改）。我们考虑了代表房屋所有权的 NFT，在这种情况下，关于房屋的元数据（图像、居住者等）可以自然地改变。

元数据作为字符串值返回。目前这仅可用于从调用`web3`，不能从其他合约调用。这是可以接受的，因为我们还没有考虑过区块链应用程序会查询此类信息的用例。

*考虑的替代方案：将每个资产的所有元数据放在区块链上（太昂贵），使用 URL 模板查询元数据部分（URL 模板不适用于所有 URL 方案，尤其是 P2P URL），multiaddr 网络地址（不够成熟）*

**社区共识**

对原始 ERC-721 问题进行了大量讨论，此外，我们在 Gitter 上举行了第一次现场会议，该会议具有良好的代表性和广泛宣传（在 Reddit 上、Gitter #ERC 频道和原始 ERC-721 问题）。感谢参与者：

- [@ImAllInNow](https://github.com/imallinnow) Rob 来自 DEC Gaming / 2 月 7 日出席密歇根以太坊聚会
- [@Arachnid](https://github.com/arachnid) Nick Johnson
- [@jadhavajay](https://github.com/jadhavajay) Ajay Jadhav from AyanWorks
- [@superphly](https://github.com/superphly) Cody Marx Bailey - XRAM Capital / 1 月 20 日黑客马拉松分享 / 联合国未来金融黑客马拉松。
- [@fulldecent](https://github.com/fulldecent) William Entriken

第二次活动在 ETHDenver 2018 举行，讨论可区分的资产标准（注释待发布）。

我们在这个过程中非常包容，并邀请任何有问题或贡献的人参与我们的讨论。但是，编写此标准仅用于支持此处列出的已识别用例。

## 向后兼容

我们已经通过`balanceOf`，`totalSupply`，`name`并`symbol`从ERC-20规范的语义。一个实现还可能包括一个函数`decimals`，uint8(0)如果它的目标是在支持该标准的同时与 ERC-20 更兼容，则该函数会返回。然而，我们发现它要求所有 ERC-721 实现都支持该`decimals`功能。

截至 2018 年 2 月的 NFT 实施示例：

- CryptoKitties -- 与该标准的早期版本兼容。
- CryptoPunks -- 部分 ERC-20 兼容，但不容易推广，因为它直接在合约中包含拍卖功能，并使用明确将资产称为“朋克”的函数名称。
- Auctionhouse Asset Interface -- 作者需要Auctionhouse ÐApp（目前是冰盒的）的通用接口。他的“资产”合约非常简单，但缺少 ERC-20 兼容性、`approve()`功能和元数据。EIP-173 的讨论中引用了这项工作。

注意：“限量版，收藏代币”如 Curio Cards 和 Rare Pepe不是可区分的资产。它们实际上是单个可替代代币的集合，每个代币都由自己的智能合约跟踪，并具有自己的总供应量（可能`1`在极端情况下）。

该`onERC721Received`函数专门针对旧部署的合约工作，这些合约true在某些情况下可能会无意中返回 1 ( )，即使它们没有实现函数（请参阅 Solidity DelegateCallReturnValue 错误）。通过返回并检查魔法值，我们能够区分实际肯定响应与这些空值`true`。

## 测试案例

0xcert ERC-721 Token 包含使用 Truffle 编写的测试用例。

## 实现

0xcert ERC721 -- 参考实现

- MIT 许可，因此您可以在您的项目中自由使用它
- 包括测试用例
- 活跃的bug赏金，如果你发现错误，你会得到报酬

Su Squares -- 一个可以租用空间和放置图片的广告平台

- 完成 Su Squares Bug Bounty Program 以寻找与此标准或其实施有关的问题
- 实现完整的标准和所有可选接口

ERC721ExampleDeed -- 一个示例实现

- 使用 OpenZeppelin 项目格式实现

XXXXERC721, 作者 William Entriken——一个可扩展的示例实现

- 部署在拥有 10 亿资产的测试网上，并通过元数据扩展支持所有查找。这表明缩放不是问题。

## 参考

**标准**

1. [ERC-20](./eip-20.md) Token Standard.
1. [ERC-165](./eip-165.md) Standard Interface Detection.
1. [ERC-173](./eip-173.md) Owned Standard.
1. [ERC-223](https://github.com/ethereum/EIPs/issues/223) Token Standard.
1. [ERC-677](https://github.com/ethereum/EIPs/issues/677) `transferAndCall` Token Standard.
1. [ERC-827](https://github.com/ethereum/EIPs/issues/827) Token Standard.
1. Ethereum Name Service (ENS). https://ens.domains
1. Instagram -- What's the Image Resolution? https://help.instagram.com/1631821640426723
1. JSON Schema. https://json-schema.org/
1. Multiaddr. https://github.com/multiformats/multiaddr
1. RFC 2119 Key words for use in RFCs to Indicate Requirement Levels. https://www.ietf.org/rfc/rfc2119.txt

**问题**

1. The Original ERC-721 Issue. https://github.com/ethereum/eips/issues/721
1. Solidity Issue \#2330 -- 接口函数是外部的。 https://github.com/ethereum/solidity/issues/2330
1. Solidity Issue \#3412 -- 实现接口：允许更严格的可变性。 https://github.com/ethereum/solidity/issues/3412
1. Solidity Issue \#3419 -- 接口不能继承。 https://github.com/ethereum/solidity/issues/3419
1. Solidity Issue \#3494 -- 编译器错误地解释了`selector`函数的原因。 https://github.com/ethereum/solidity/issues/3494
1. Solidity Issue \#3544 -- 无法计算名为`transfer`。 https://github.com/ethereum/solidity/issues/3544
1. CryptoKitties Bounty Issue \#4 -- 列出用户拥有的所有 Kitties `O(n^2)`。 https://github.com/axiomzen/cryptokitties-bounty/issues/4
2. OpenZeppelin Issue \#438 -- `approve` 方法的实现违反了 ERC20 标准。 https://github.com/OpenZeppelin/zeppelin-solidity/issues/438
3. Solidity DelegateCallReturnValue Bug. https://solidity.readthedocs.io/en/develop/bugs.html#DelegateCallReturnValue

**讨论**

1. Reddit (第一次现场讨论的公告). https://www.reddit.com/r/ethereum/comments/7r2ena/friday_119_live_discussion_on_erc_nonfungible/
1. Gitter #EIPs (第一次现场讨论的公告). https://gitter.im/ethereum/EIPs?at=5a5f823fb48e8c3566f0a5e7
1. ERC-721 (第一次现场讨论的公告). https://github.com/ethereum/eips/issues/721#issuecomment-358369377
1. ETHDenver 2018. https://ethdenver.com

**NFT 实现和其他项目**

1. CryptoKitties. https://www.cryptokitties.co
1. 0xcert ERC-721 Token. https://github.com/0xcert/ethereum-erc721
1. Su Squares. https://tenthousandsu.com
1. Decentraland. https://decentraland.org
1. CryptoPunks. https://www.larvalabs.com/cryptopunks
1. DMarket. https://www.dmarket.io
1. Enjin Coin. https://enjincoin.io
1. Ubitquity. https://www.ubitquity.io
1. Propy. https://tokensale.propy.com
1. CryptoKitties Deployed Contract. https://etherscan.io/address/0x06012c8cf97bead5deae237070f9587f8e7a266d#code
1. Su Squares Bug Bounty Program. https://github.com/fulldecent/su-squares-bounty
1. XXXXERC721. https://github.com/fulldecent/erc721-example
1. ERC721ExampleDeed. https://github.com/nastassiasachs/ERC721ExampleDeed
1. Curio Cards. https://mycuriocards.com
1. Rare Pepe. https://rarepepewallet.com
1. Auctionhouse Asset Interface. https://github.com/dob/auctionhouse/blob/master/contracts/Asset.sol
1. OpenZeppelin SafeERC20.sol Implementation. https://github.com/OpenZeppelin/zeppelin-solidity/blob/master/contracts/token/ERC20/SafeERC20.sol

## 版本

通过[CC0](https://creativecommons.org/publicdomain/zero/1.0/)　放弃版权和相关权。