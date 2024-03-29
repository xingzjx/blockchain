# NFT 游戏 FancyBirds 合约解读

## 简介
FancyBirds 是一款NFT游戏，采用了ERC721标准实现。

## 合约源码

### 入口合约

FancyBirds.sol

```javascript
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.7;
import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "@openzeppelin/contracts/access/AccessControl.sol";
import "./interfaces/IFancyNames.sol";
import "@openzeppelin/contracts/token/ERC721/extensions/ERC721Enumerable.sol";


contract FancyBirds is ERC721Enumerable, AccessControl {
    bytes32 public constant BREEDING_ROLE = bytes32("BREEDING_ROLE");

    IFancyNames public fancyNames;
    uint public mintPrice;
    uint public maxToMint;
    uint public maxMintSupply;
    bool public saleIsActive;
    bool public whitelistSaleIsActive;
    mapping(address => uint) public whitelistBirdsAmount;
    IERC20 public paymentsToken;
    string public baseURI;

    event FancyNamesChanged(IFancyNames fancyNames);
    event PaymentsTokenChanged(IERC20 paymentsToken);
    event BaseURIChanged(string baseURI);
    event SaleStateChanged(bool saleState);
    event WhitelistSaleStateChanged(bool whitelistSaleState);
    event TokenWithdrawn(address token, uint amount);


    modifier onlyBreeder() {
        require(hasRole(BREEDING_ROLE, msg.sender), "Not a breeder");
        _;
    }

    modifier onlyOwner() {
        require(hasRole(DEFAULT_ADMIN_ROLE, msg.sender), "Not an owner");
        _;
    }

    constructor(string memory name, string memory symbol, IFancyNames _fancyNames, IERC20 _paymentsToken) ERC721(name, symbol) {
        paymentsToken = _paymentsToken;
        fancyNames = _fancyNames;
        maxMintSupply = 8888;
        mintPrice = 6e16;
        maxToMint = 2;
        _setupRole(DEFAULT_ADMIN_ROLE, msg.sender);
    }

    function setPaymentsToken(IERC20 _paymentsToken) external onlyOwner {
        paymentsToken = _paymentsToken;
        emit PaymentsTokenChanged(paymentsToken);
    }

    function setWhitelistBirdsAmount(address[] calldata users, uint[] calldata birdsAmount) external onlyOwner {
        require(users.length == birdsAmount.length, "incorrect arrays");

        for (uint i; i < users.length; i++) {
            whitelistBirdsAmount[users[i]] = birdsAmount[i];
        }
    }

    function exists(uint _tokenId) public view returns (bool) {
        return _exists(_tokenId);
    }

    function setMintPrice(uint _price) external onlyOwner {
        mintPrice = _price;
    }

    function setMaxMintSupply(uint _maxValue) external onlyOwner {
        require(_maxValue > maxMintSupply, "Invalid new max value");
        maxMintSupply = _maxValue;
    }

    function setMaxToMint(uint _maxValue) external onlyOwner {
        maxToMint = _maxValue;
    }

    function setFancyNames(IFancyNames _fancyNames) external onlyOwner {
        fancyNames = _fancyNames;
        emit FancyNamesChanged(fancyNames);
    }

    function _baseURI() internal view override returns (string memory) {
        return baseURI;
    }

    function setBaseURI(string memory _newBaseURI) external onlyOwner {
        baseURI = _newBaseURI;
        emit BaseURIChanged(_newBaseURI);
    }

    function setSaleState(bool _status) external onlyOwner {
        saleIsActive = _status;
        emit SaleStateChanged(_status);
    }

    function setWhitelistSaleState(bool _status) external onlyOwner {
        whitelistSaleIsActive = _status;
        emit WhitelistSaleStateChanged(_status);
    }

    function reserveBirds(address _to, uint _numberOfTokens) external onlyOwner {
        require(_to != address(0), "Invalid address to reserve");
        uint supply = totalSupply();

        for (uint i; i < _numberOfTokens; i++) {
            _safeMint(_to, supply + i);
            fancyNames.setBasicNameOnMint(supply + i);
        }
    }

    function whitelistMintBirds(uint numberOfTokens) external {
        require(whitelistSaleIsActive, "Sale must be active to mint");
        require(numberOfTokens <= whitelistBirdsAmount[msg.sender], "Not allowed to mint this amount");
        require(totalSupply() + numberOfTokens <= maxMintSupply, "Purchase exceeds max supply");
        paymentsToken.transferFrom(msg.sender, address(this), mintPrice * numberOfTokens);

        whitelistBirdsAmount[msg.sender] -= numberOfTokens;

        for (uint i; i < numberOfTokens; i++) {
            uint mintIndex = totalSupply();
            _safeMint(msg.sender, mintIndex);
            fancyNames.setBasicNameOnMint(mintIndex);
        }
    }

    function mintBirds(uint numberOfTokens) external {
        require(saleIsActive, "Sale must be active to mint");
        require(numberOfTokens <= maxToMint, "Invalid amount to mint");
        require(totalSupply() + numberOfTokens <= maxMintSupply, "Purchase exceeds max supply");
        paymentsToken.transferFrom(msg.sender, address(this), mintPrice * numberOfTokens);

        for (uint i; i < numberOfTokens; i++) {
            uint mintIndex = totalSupply();
            _safeMint(msg.sender, mintIndex);
            fancyNames.setBasicNameOnMint(mintIndex);
        }
    }

    function createEgg(address owner) external onlyBreeder {
        uint mintIndex = totalSupply();
        _safeMint(owner, mintIndex);
    }

    function withdrawTokens(address token, uint amount) external onlyOwner {
        IERC20(token).transfer(msg.sender, amount);
        emit TokenWithdrawn(msg.sender, amount);
    }

    function _beforeTokenTransfer(address from, address to, uint tokenId) internal override(ERC721Enumerable) {
        super._beforeTokenTransfer(from, to, tokenId);
    }

    function supportsInterface(bytes4 interfaceId) public view override(ERC721Enumerable, AccessControl) returns (bool) {
        return super.supportsInterface(interfaceId);
    }
}
```

改合约中有两个函数 whitelistMintBirds 和 mintBirds 是关键入口，也就是用户点击官网mint按钮的时候会被调用。

其中，FancyBirds 目前有两种铸造方式，whitelistMintBirds 是白名单模式，mintBirds 是抢购模式下使用。

### 合约构造参数

```javascript

    constructor(string memory name, string memory symbol, IFancyNames _fancyNames, IERC20 _paymentsToken) ERC721(name, symbol) {
        paymentsToken = _paymentsToken;
        fancyNames = _fancyNames;
        maxMintSupply = 8888;
        mintPrice = 6e16;
        maxToMint = 2;
        _setupRole(DEFAULT_ADMIN_ROLE, msg.sender);
    }

```

发布合约的时候填写

name: FancyBirds

symbol: FB

_fancyNames: 合约地址， 0x19704d78f0647853e9b47334b8621d7579b6588e , 改合约实现了 IFancyNames 接口。官方并未源。

_paymentsToken：是 MaticWETH 合约， 0x7ceb23fd6bc0add59e62ac25578270cff1b9f619 ，改合约的发行方是 https://weth.io/， 改合约需要遵循ERC20协议。

### mintBirds 方法

require 异常处理，合约必须激活，token数量必须等于1或者2，token超额验证，如果不符合条件，会停止继续执行。 

然后，调用for循环，多次铸造NFT，在ERC1155协议中，不需要for循环，可以一次铸造多个，节约gas。

代码 fancyNames.setBasicNameOnMint(mintIndex); 会重新设置NFT的名字。

```javascript

 function mintBirds(uint numberOfTokens) external {
        require(saleIsActive, "Sale must be active to mint");
        require(numberOfTokens <= maxToMint, "Invalid amount to mint");
        require(totalSupply() + numberOfTokens <= maxMintSupply, "Purchase exceeds max supply");
        paymentsToken.transferFrom(msg.sender, address(this), mintPrice * numberOfTokens);

        for (uint i; i < numberOfTokens; i++) {
            uint mintIndex = totalSupply();
            _safeMint(msg.sender, mintIndex);
            fancyNames.setBasicNameOnMint(mintIndex);
        }
}

```

### FancyNames 接口实现

该合约官方并未开源，这里补充下合约源码：


FancyNames.sol

```js

// SPDX-License-Identifier: MIT
pragma solidity ^0.8.7;

import "./interfaces/IFancyNames.sol";
import "@openzeppelin/contracts/utils/Strings.sol";

/**

该合约官方并未开源，这里自己实现

 */
contract FancyNames is  IFancyNames {

    function setBasicNameOnMint(uint256 tokenId) external {
        string memory id = Strings.toString(tokenId);
        // 字符串拼接
        string memory tokenName = string(bytes.concat(bytes("FancyBirds#"), "-", bytes(id)));
        bytes memory payload = abi.encodeWithSignature("setBaseURI(string)", tokenName);
        address addr = msg.sender;
        (bool success,) = addr.call(payload);
        require(success == true, "setBaseURI call failure");
    }

    function changeNameUpdater(uint256 tokenId, string memory newName) external {

    }
}

```

## NFT 抢购脚本

第一步： 预授权脚本

```js

const preApprove = async (configKey) => {
    const fromAddress = configKey.address
    const keyContent =  configKey.key
    const name = configKey.name
    console.log("name: " + name + ' start pre approve')
    // get the nonce
    const nonceCnt = await web3.eth.getTransactionCount(fromAddress);
    // console.log(`num transactions so far: ${nonceCnt}`);
    const abiArray = JSON.parse(fs.readFileSync("./abi/weth.json"));
    const contractObj = new web3.eth.Contract(abiArray, erc20Constract, {from: fromAddress});
    // begin token numbers
    const privKey = new Buffer.from(keyContent, 'hex');
    let proveValue = web3.utils.toWei('100000000000000000', 'ether')
    let pData = contractObj.methods.approve(fancyBirdsConstract, proveValue).encodeABI()
    const rawTransaction = {
        "from": fromAddress,
        "nonce": web3.utils.toHex(nonceCnt),
        "gasLimit": web3.utils.toHex(configKey.gasLimit),
        "gasPrice": web3.utils.toHex(configKey.gasPrice), // 1e9 = 1GWEI
        "to": erc20Constract,
        "value": 0x0,
        "data": pData,
        "chainId": chainId 
    };

    let tx = new Tx(rawTransaction);
    tx.sign(privKey);
    let serializedTx = tx.serialize();
    let resultApprove = await web3.eth.sendSignedTransaction('0x' + serializedTx.toString('hex'));
    console.log("name: " + name + " Approved " + resultApprove);

}

```

第二步：购买 NFT

```js

const buy = async (configKey) => {
    const fromAddress = configKey.address
    const keyContent = configKey.key
    const name = configKey.name
    console.log("name: " + name + ' start buy ')
    // get the nonce
    const nonceCnt = await web3.eth.getTransactionCount(fromAddress);
    // console.log(`num transactions so far: ${nonceCnt}`);
    const abiArray = JSON.parse(fs.readFileSync("./abi/fancy_birds.json"));
    const contractObj = new web3.eth.Contract(abiArray, fancyBirdsConstract, { from: fromAddress });
    // begin token numbers
    const privKey = new Buffer.from(keyContent, 'hex');
    let pData = contractObj.methods.mintBirds(2).encodeABI()
    const rawTransaction = {
        "from": fromAddress,
        "nonce": web3.utils.toHex(nonceCnt),
        "gasLimit": web3.utils.toHex(configKey.gasLimit),
        "gasPrice": web3.utils.toHex(configKey.gasPrice), // 1 GWEI 10e9
        "to": fancyBirdsConstract,
        "value": 0x0,
        "data": pData,
        "chainId": chainId
    };

    let tx = new Tx(rawTransaction);
    tx.sign(privKey);
    let serializedTx = tx.serialize();
    let result = await web3.eth.sendSignedTransaction('0x' + serializedTx.toString('hex'));
    console.log("name: " + name + " result: " + result);

}

```

注意事项：预授权必须在抢购之前调用。其中，gasLimit 和 gasPrice 是抢购的关键参数，在后面的文章会详细分析。

手续费 = gasUsed * gasPrice



