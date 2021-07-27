# multisender 批量转账合约分析

## 准备工作

项目代码：https://gitee.com/xingzjx/batch-sender

按照　**Contract Deployment** 部分，使用 remix　或者　tu 部署合约后生成　flats/UpgradebleStormSender_flat.sol　文件。

整个合约代码：https://gitee.com/xingzjx/batch-sender/blob/master/contracts/flats/UpgradebleStormSender_flat.sol

## 批量转账方法

### 前端调用

前端代码会调用　multisendToken　方法实现批量转账

```solidity

　　 /**
　　　multisendEther：以太坊多转一
　　　address　: 合约地址
     _contributors : 转账的接收账号地址
     _balances : 转账金额
     **/
    function multisendToken(address token, address[] _contributors, uint256[] _balances) public hasFee payable {
        if (token == 0x000000000000000000000000000000000000bEEF){
            multisendEther(_contributors, _balances);
        } else { // 批量转 ERC20
            uint256 total = 0;
            require(_contributors.length <= arrayLimit());
            ERC20 erc20token = ERC20(token);
            uint8 i = 0;
            for (i; i < _contributors.length; i++) {
                erc20token.transferFrom(msg.sender, _contributors[i], _balances[i]);
                total += _balances[i];
            }
            setTxCount(msg.sender, txCount(msg.sender).add(1));
            Multisended(total, token);　
        }
    }

```

### 批量转以太坊

```solidity

    function multisendEther(address[] _contributors, uint256[] _balances) public payable {
        uint256 total = msg.value;
        uint256 fee = currentFee(msg.sender);
        require(total >= fee);
        require(_contributors.length <= arrayLimit());
        total = total.sub(fee);
        uint256 i = 0;
        for (i; i < _contributors.length; i++) {
            require(total >= _balances[i]);
            total = total.sub(_balances[i]);
            _contributors[i].transfer(_balances[i]);
        }
        setTxCount(msg.sender, txCount(msg.sender).add(1));
        Multisended(msg.value, 0x000000000000000000000000000000000000bEEF);
    }

```

### 事件通知

```solidity

event Multisended(uint256 total, address tokenAddress);

```

## 合约授权

在 erc20　协议已经定义，两个重要的方法：

allowance：允许转账的金额大小, 其中，owner是合约拥有者，spender是申请转账的地址。

```solidiy

function allowance(address owner, address spender) public view returns (uint256);

```

approve: 授权方法
 
```solidity

function approve(address spender, uint256 value) public returns (bool);

```

## arrayLimit

每次转账的地址数量限制，默认设置为2000