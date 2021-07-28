# 构建、部署和销售你自己的动态NFT

nft是区块链空间之外无法找到的工具，它允许广泛的应用程序和可能性。对于那些希望构建收藏品、独立代币、门票、游戏应用程序或ERC721代币标准允许的任何东西的人来说，构建动态和随机的非功能性测试是一个很好的开始。但我们现在能用它做什么呢?展示你新创造的随机或动态角色不是很棒吗?

我们认为也是。在本教程中，我们将带领您完成将自己的动态或随机非功能性测试部署到OpenSea市场的所有步骤。这是最后的一个例子。


## 快速回顾NFT

ERC721(也称为NFTs)定义了一个框架，用于制作唯一且彼此不同的令牌(因此出现了术语不可替代)，而流行的ERC20标准定义了“可替代”的令牌，这意味着所有令牌都是可互换的，并保证具有相同的价值。我们将更深入地探讨如何构建这些平台，以及社区如何跨平台呈现它们。你也可以在OpenSea非语言测试圣经中读到更多。

如果您还没有阅读上一篇关于在非功能性测试中获取随机数的文章，请务必回头看看!开发人员选项卡充满了各种智能合约和区块链工程教学的教程、指南和操作指南。

## 什么是元数据

在上一篇博客中，我们学习了如何构建随机nft。现在，我们将使用ERC721标准的另一个重要部分:元数据，将它推进到下一个层次。

所有nft都有所谓的元数据。你可以在ERC/EIP 721的原始提案中读到这一点。基本上，社区发现的是，在以太坊上存储图像是非常费力和昂贵的。如果你想存储一张8 * 8的图片，存储这么多数据是相当便宜的，但如果你想要一张分辨率不错的图片，你就需要花更多的钱。

数据存储的成本是(大约)640k gas 每Kb数据。如果当前的 gas 价格大约是50 Gwei或0.000000050 ETH，而1 ETH目前等于600美元，那么你将花费20美元。

再加20美元。这并没有真正让非功能性测试的创造者感到兴奋。

我们知道，以太坊2.0将解决许多这些扩展性问题(也祝贺以太坊2.0的成功发布)，但目前，社区需要一个标准来帮助解决这个问题。元数据就是这个帮助。

元数据为链外存储的tokenId提供描述性信息。这些是简单的api，链外ui调用它们来收集关于令牌的所有信息。每个tokenId都有一个特定的tokenURI来定义这个API调用，它返回一个JSON对象，看起来像这样:

```json

{
    "name": "You NFT token name",
    "description": "Something Cool here",
    "image": "https://ipfs.io/ipfs/QmTgqnhFBMkfT9s8PHKcdXBn1f5bG3Q5hmBaR4U6hoTvb1?filename=Chainlink_Elf.png",
    "attributes": [. . .]
}

```

您会注意到元数据有四个不同的键。

- 名称，它定义了tokenid人类可读的名称

- 描述，提供有关令牌的一些背景信息

- image是图像的另一个URI

- 属性，允许您显示您的令牌的统计信息

重要的是，如果您的NFT与其他NFT交互，请确保tokenURI上的属性与您的NFT智能合约的属性相匹配，否则当战斗或交互没有达到预期效果时，您可能会感到困惑!

一旦我们将token id分配给他们的token uri，非功能性市场将能够显示您的token，允许您展示您的创造力。你可以在Rinkeby测试网的OpenSea市场上看到我们使用更新的龙与地下城随机nft。有许多这样的市场，如Mintable, Rarible和OpenSea。

## 链上和链下的元数据

您总是可以在链上存储所有元数据(事实上，这是令牌交互的唯一方式)，但许多非功能性市场还不知道如何读取链上元数据。所以目前，使用链下元数据来可视化你的令牌，同时拥有所有链上元数据是理想的，这样你的令牌就可以相互交互。

名称、描述和属性很容易在链上存储，但图像是困难的部分。另外，我们在哪里存储这个用于tokenURI的API ?很多人选择运行服务器来承载信息，这很好，但这是可视化令牌的集中地。如果我们能将图像存储在链上，这样它们就不会崩溃或被黑，那就更好了。您将注意到，在上面的示例中，它们的图像使用指向IPFS的URL，这是一种流行的存储图像的方法。

IPFS是星际文件系统(InterPlanetary File System)的缩写，是一种点对点的超媒体协议，旨在使网络更快、更安全、更开放。它允许任何人上传文件，文件被哈希，如果它改变了，它的哈希也会改变。这对于存储映像非常理想，因为这意味着每次映像更新时，链上散列/tokenURI也必须更改，这意味着我们可以有元数据历史记录。将映像添加到IPFS中也非常容易，而且不需要运行服务器!

现在我们知道要做什么了，让我们构建和部署吧!一旦你部署了你的非功能性令牌和市场，令牌看起来就像这样:

给你的额外惊喜

现在，对于所有已经读过这个博客的人，我们有一个额外的惊喜给你。我们将在一秒钟内了解如何部署这些NFT，但在此之前，我们在这个游戏中创建的这四个代币将在我们的第一个非NFT寻宝游戏中进行争夺!

## 怎样部署动态的NFT市场

再一次，我们将使用更新版本的**龙与地下城**仓库，它在自述中也有说明。

以下是我们将要做的:

- 使用Chainlink VRF创建一个可验证的随机D&D角色

- 使用IPFS添加一个标记uri

- 将你的随机NFTs添加到OpenSea市场

请记住，您可以更改repo，使其适用于动态nft。您可以很容易地将VRF换成Chainlink Price Feeds或Chainlink API。

这个 repo 目前只与 Rinkeby 工作，所以请务必跳转到 Rinkeby !我们将从头开始，所以如果你没有读上一篇文章，也不用担心。

你的钱包里需要有Rinkeby Testnet ETH和Rinkeby Testnet LINK才能继续。

### 定义环境变量

您需要一个助记词和一个RINKEBY_RPC_URL环境变量。你的助记词是你钱包的种子短语。你可以在Infura等节点提供商服务中找到RINKEBY_RPC_URL

然后，要么在bash_profile文件中设置它们，要么像这样将它们导出到终端:

```bash

export MNEMONIC='cat dog frog....'

export RINKEBY_RPC_URL='www.infura.io/asdfadsfafdadf'

```

补充: 这两个全局变量也可以在源码里面的 .env　文件定义

### 克隆仓库和集成

```bash

git clone https://github.com/PatrickAlphaC/dungeons-and-dragons-nft

cd dungeons-and-dragons-nft

git checkout opensea-update

npm install

truffle migrate --reset --network rinkeby

```

### 创建角色

在工程目录下执行命令：

```bash

truffle exec scripts/fund-contract.js --network rinkeby

truffle exec scripts/generate-character.js --network rinkeby

truffle exec scripts/get-character.js --network rinkeby


```

这将会随机创建一个角色

根据部署的情况，您可以通过更改get-character.js中的dnd.getCharacterOverView(1)命令来选择哪个字符，将0替换为您喜欢的字符的任何tokenId。

这将给你们非功能性测试的概述。你会看到BN，因为调用返回的是大数，你可以将它们转换为int，看看它们是什么。或者你可以更进一步……

## Etherscan 查看

您可以免费获得一个Etherscan API密钥，并与链上NFTs进行交互。然后将ETHERSCAN_API_KEY设置为环境变量。

```bash

npm install truffle-plugin-verify

truffle run verify DungeonsAndDragonsCharacter --network rinkeby --license MIT

```

这将验证并发布你的合同，你可以去Etherscan提供的阅读合同部分。

或者，您也可以使用oneclickdapp，只添加合同地址和ABI。您可以在build/contract文件夹中找到ABI。只要记住不是整个文件是ABI，只是说ABI的部分。

## 部署到OpenSea

一旦我们创建了nft，我们需要给它们一个tokenURI。tokenuri是向世界展示NFTs数据的标准。这使得存储像图像这样的东西变得更容易，因为我们不需要浪费在链上添加它们的时间。

TokenURI表示一个URL或其他唯一标识符，它是一个带有一些参数的.json文件。

```json

{

    "name": "Name for it ",

    "description": "Anything you want",

    "image": "https://ipfs.io/ipfs/HASH_HERE?file.png",

    "attributes": [...]

}

```

## 下载 IPFS 和 IPFS Companion

现在，我们将在IPFS中存储这些图像和元数据。你需要:

- ipfs : https://ipfs.io/

- ipfs Companion: https://chrome.google.com/webstore/detail/ipfs-companion/nibjojkomfdiaoajekhjakgkdhaomnch?hl=en

- Pinata　：https://pinata.cloud/pinataupload

IPFS可以让我们在像Brave或Chrome这样的浏览器中查看IPFS数据。而且Pinata允许我们在节点宕机时保持IPFS文件处于up状态(暂时不要担心这个问题)。如果您在浏览器中单击此链接:https://ipfs.io/ipfs/QmTgqnhFBMkfT9s8PHKcdXBn1f5bG3Q5hmBaR4U6hoTvb1?filename=Chainlink_Elf.png，就会知道 IPFS Companion 正在工作

### 添加图片到 IPFS

IPFS节点启动后，就可以开始向它添加文件了。我们首先要上传非功能性测试的图像。转到IPFS安装的“文件”部分。

《龙与地下城》的角色长什么样?将它添加到IPFS节点，然后“固定”它。现在，你可以随意固定一个空白的图像，或者其他一些愚蠢的东西。


### 添加元数据到 IPFS

然后，您需要将元数据JSON对象添加到IPFS。您需要从已部署的令牌中获取您的名称和属性。我们在create-metadata.js脚本中为您做了一些工作。

```bash

truffle exec scripts/create-metadata.js --network rinkeby 

```

执行命令后，元数据会在当前工程目录的　metadata 文件夹中生成。它现在只需要图像URL !元数据是我们使用Chainlink VRF创建的随机数和统计数据。现在，我们获得所创建的固定图像的CID，并将其添加到元数据JSON文件中，然后将该文件添加到IPFS中，并将其固定在那里!

### Pinata

如果我们的IPFS节点宕机，或者我们关闭了计算机，我们将无法提取元数据，因此我们需要一种方法来固定它们，并让其他节点托管数据。这就是皮纳塔的由来。别担心，这是免费的!这将有助于在IPFS节点关闭时保持数据正常运行。我们复制我们的图像和JSON元数据文件的CID，并将其添加到我们的Pinata帐户。注册。需要几秒钟。

然后运行：

```bash

truffle exec scripts/set-token-uri.js --network rinkeby

```

现在，我们可以获得 NFT 测试的地址，然后前往OpenSea测试网市场，看看我们是否做对了。如果操作正确，它看起来就像这样。我们得先和OpenSea做个账。

这里是添加你的测试网非测试合同的链接，可以在opensea上查看。然后，你就可以开始推销你的 NFT 了!

## 接下来

我们应该都准备好了!我们在这里涵盖了很多信息，所以如果你有任何问题，请联系我们的Discord。智能合约和Chainlink工程师的社区是巨大的，许多聪明的人聚集在一起，将NFTs和智能合约推到聚光灯下，所以Discord也是一个认识其他人的好地方。你也会想要检查Chainlink建设者计划，在那里你可以赢得一些很酷的奖品与Chainlink建筑!

和往常一样，一定要访问开发人员文档，你也可以订阅Chainlink Newsletter，以便与Chainlink堆栈中的所有内容保持同步。

如果你在这里学到了一些新东西，想要展示你已经创建的内容，或者为一些演示回购开发了前端，确保你在Twitter、Discord或Reddit上分享它，并给你的回购贴上#chainlink标签。


源码：

https://github.com/PatrickAlphaC/dungeons-and-dragons-nft

原文：

[Build, Deploy, and Sell Your Own Dynamic NFT](https://blog.chain.link/build-deploy-and-sell-your-own-dynamic-nft/)

參考：

[Chainlink VRF 可验证随机函数详解](https://learnblockchain.cn/article/1041)

[chainlink VRF 官方介绍](https://docs.chain.link/docs/chainlink-vrf/)