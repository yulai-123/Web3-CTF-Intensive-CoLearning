---
timezone: Asia/Shanghai
---


# Ha2ryzhang

1. 自我介绍

    Hi,I'm ha2ryzhang.
    正在学习合约安全相关的内容.

2. 你认为你会完成本次残酷学习吗？

    尽力而为之.

## Notes

<!-- Content_START -->

### 2024.08.29

接着之前一直做了一半的 `Ethernaut` 继续学习
#### A-Ethernaut-NaughtCoin
[link](https://ethernaut.openzeppelin.com/level/0x80934BE6B8B872B364b470Ca30EaAd8AEAC4f63F)

##### 这题问题主要是 ERC20转账方式
[WTF Academy](https://www.wtf.academy/docs/solidity-103/ERC20/)
1. `transfer`
2. `transferFrom`
他这里只对 `transfer`进行了重写 所以 `transferFrom` 是可以绕过`lockTokens`的检查的.
按顺序做的,这题还是相对简单点.

### 2024.08.30

#### A-Ethernaut-Preservation
看到题目是需要获取合约的owner权限,但是又没看到set owner的方法.
看到了`delegatecall`,想到了`slot`相关的,很久之前看过了,有点忘记了,先查一下.

[delegatecall](https://www.wtf.academy/docs/solidity-102/Delegatecall/) 是需要顺序和数据类型保持一致的.
##### 分析

[资料](https://ctf-wiki.org/blockchain/ethereum/storage/)

`address`：地址类型存储一个 20 字节的值（以太坊地址的大小）。 地址类型也有成员变量，并作为所有合约的基础。
```
-----------------------------------------------------
| unused (12)   |   timeZone1Library (20)           | <- slot 0
-----------------------------------------------------
| unused (12)   |   timeZone2Library (20)           | <- slot 1
-----------------------------------------------------
| unused (12)   |             owner (20)            | <- slot 2
-----------------------------------------------------
|                storedTime (32)                    | <- slot 3
-----------------------------------------------------
| unused (28)   |        setTimeSignature (4)       | <- slot 4
-----------------------------------------------------

```
上面是`Preservation.sol`合约中变量的`Storage Slot`


问题就在于`LibraryContract`合约中slot位置对不上

```
-----------------------------------------------------
|                storedTime (32)                    | <- slot 0
-----------------------------------------------------

```
这么一对比可以很直观看到.理论上无论调用`setFirstTime`或者`setSecondTime`都会是修改
变量`timeZone1Library`的值.那么只需要一个攻击合约对应上`owner`所在的slot就能修改owner了

##### 攻击

```solidity
contract Attack{
    address public tmp1;
    address public tmp2;
    address public owner;
    function setTime(uint256 _time) public {
        //类型转换 time 为 address
        owner = address(uint160(_time));
    }
}
```
合约中owner对上即可,测试通过!
还有一个类型转换的点:
1. uint256 到 uint160:
address 类型在 Solidity 中是一个 20 字节（160 位）的值，而 uint256 是一个 32 字节（256 位）的值。
因此，需要先将 uint256 转换为 uint160，因为 uint160 也正好是 20 字节，可以容纳 address 类型的数据。
2. uint160 到 address:
将 uint160 转换为 address 类型非常简单，只需要将其强制转换即可。

### 2024.08.31

#### A-Ethernaut-Recovery

看了眼题目,一开始没明白什么意思,为什么会找不到地址.etherscan里是可以看到地址的,哈哈.
直接调用`destroy()`就可以拿回资金了.

去看了一眼Factory合约,答案直接都看到了,知识点是地址生成的规则,去年看过,这么久忘了,正好复习复习.

##### 合约创建方式

1. create

create的用法很简单，就是new一个合约，并传入新合约构造函数所需的参数：
```solidity

Contract x = new Contract{value: _value}(params)

```
**CREATE如何计算地址**

`新地址 = hash(创建者地址, nonce)`

创建者地址不会变，但nonce可能会随时间而改变，因此用CREATE创建的合约地址不好预测。

2. create2

[详解](https://www.wtf.academy/docs/solidity-102/Create2/)
和create差不多,多了一个`salt`参数

**CREATE2如何计算地址**

`新地址 = hash("0xFF",创建者地址, salt, initcode)`

CREATE2 确保，如果创建者使用 CREATE2 和提供的 salt 部署给定的合约initcode，它将存储在 新地址 中。

-------------------

回到题目 Recovery 使用的是create来创建的Token合约,所以模拟对应参数就行

`address token=address(
            uint160(uint256(keccak256(abi.encodePacked(uint8(0xd6), uint8(0x94), levelAddress, uint8(0x01)))))
        );`

这里levelAddresss是已知的,`0x01`可以理解为nonce,如果再调用一次`generateToken`那么就应该是`0x02`.


### 2024.09.01

#### A-Ethernaut-MagicNumber

这题第一次看的时候很懵,考察的点是`opcode`和`assembly`的使用.


[evm.code](https://www.evm.codes/?fork=shanghai)
[WTF 教程](https://www.wtf.academy/docs/evm-opcodes-101/)

知识点很多,目前还没看完,过几天补一补.

### 2024.09.02

#### A-Ethernaut-AlienCodex

这一题也是`storage slot` 相关的.
因为低版本的solidity是没有溢出检查的
```solidity
pragma solidity ^0.4.0;

contract C {
    uint256 a;      // 0
    uint[] b;       // 1
    uint256 c;      // 2
}

```

```
-----------------------------------------------------
|                      a (32)                       | <- slot 0
-----------------------------------------------------
|                    b.length (32)                  | <- slot 1
-----------------------------------------------------
|                      c (32)                       | <- slot 2
-----------------------------------------------------
|                        ...                        |   ......
-----------------------------------------------------
|                      b[0] (32)                    | <- slot `keccak256(1)`
-----------------------------------------------------
|                      b[1] (32)                    | <- slot `keccak256(1) + 1`
-----------------------------------------------------
|                        ...                        |   ......
-----------------------------------------------------

```
就是需要codex[] 越界访问 owner
`(2**256 - 1) + 1 = 0`


### 2024.09.03

#### A-Ethernaut-Denial
这题直接死循环消耗完gas即可

#### A-Ethernaut-Shop

攻击合约定义一个price方法 通过`isSold`判断价格,返回不同的价格

#### A-Ethernaut-Dex

这题是第一个碰到的defi相关的题目.题目需要把dex的余额变为0.

问题出现在`getSwapPrice`方法,这个算法有问题,如果多swap几次,余额只会慢慢变多.

#### A-Ethernaut-DexTwo

这一题对比上一题缺少了`require((from == token1 && to == token2) || (from == token2 && to == token1), "Invalid tokens");`
可以利用其他代币来换出token.



### 2024.09.04

#### A-Ethernaut-PuzzleWallet

今天这道题卡了几个小时,最开始发现proxy的`storage slot`对不上,就开始研究.
实际上`delegatecall`proxy合约的`proposeNewAdmin`其实就是对应 Wallet合约的`owner`(同slot 0),
获取了`owner`权限,就可以添加whiteliste,就可以调用wallet合约的方法,观察 如果想改变proxy合约的owner 就得改变wallet合约的`maxBalance`(同样对应slot 1),可是maxBalance如果想要修改的话,又必须得保证`address(wallet).balance == 0`,所以得想办法让合约的余额等于0.

分析了调用`execute`方法,发现只能发送自己的余额,而wallet的初始余额是`factory`合约的,
所以有陷入了死胡同,`multicall`是唯一可以利用的地方,但是这个方法又有一个很巧妙设计的地方,为了`mag.value`不被重复利用,做了`selector`的判断,换个角度calldata的第二个交易如果还是`multicall`就能完美绕过.

就可以存双份钱,绕过`require(balances[msg.sender] >= value, "Insufficient balance");`,
然后就可以修改`maxBalance`为`player`地址,就结束.


### 2024.09.05

#### A-Ethernaut-Motorbike

是个uups的合约,实际上逻辑合约是没有init的,因为proxy合约`delegatecall`了`initialize`.
验证发现逻辑合约的`upgrader`和`horsePower`确实都是0,没有初始化的.
既然没有初始化,意味着可以调用`initialize`让自己变成`upgrader`,然后升级合约.

这题是要让合约`selfdestruct`,写个攻击合约,带个`selfdestruct`方法,然后调用`upgradeToAndCall`
即可. 

貌似 foundry test中 `selfdestruct` 无效,所以 poc 中后面手动处理了

### 2024.09.06

#### A-Ethernaut-DoubleEntryPoint

这题分析业务看了有点久,不太明白想要干啥.

`CryptoVault`中的`underlying`是不应该被transfer的,
`require(token != underlying, "Can't transfer underlying token");`做了判断,可是`sweptTokensRecipient`的`token`是`LegacyToken`(LGT), `LegacyToken`的transfer又是代理的`DoubleEntryPoint`(DET)

```solidity
function transfer(address to, uint256 value) public override returns (bool) {
    if (address(delegate) == address(0)) {
        return super.transfer(to, value);
    } else {
        return delegate.delegateTransfer(to, value, msg.sender);
    }
}
```
`delegate`调用的其实就是DET,
这里的`delegateTransfer`的`origSender`是`CryptoVault`,

所以 `CryptoVault`中的`DET`最后会被转移为0


`DET`中有个 `modifier fortaNotify()`

```solidity

modifier fortaNotify() {
    address detectionBot = address(forta.usersDetectionBots(player));

    // Cache old number of bot alerts
    uint256 previousValue = forta.botRaisedAlerts(detectionBot);

    // Notify Forta
    forta.notify(player, msg.data);

    // Continue execution
    _;

    // Check if alarms have been raised
    if (forta.botRaisedAlerts(detectionBot) > previousValue) revert("Alert has been triggered, reverting");
}

```

`Foeta`允许用户设置机器人来提醒交易,需要自己定义个合约,如果交易有问题调用`raiseAlert`这个modifier
就会`revert`.
最后实现Bot的`handleTransaction`方法判断`originSender == cryptoVault`来触发`raiseAlert`.


### 2024.09.07

#### A-Ethernaut-GoodSamaritan

这题`requestDonation`里catch到错误了,会判断是否是`NotEnoughBalance()`,是的话就转走剩余的coin,所以实现`notify`来判断`amount==10`(转走剩余的coin肯定不等于10),然后revert同样的错误就行

### 2024.09.08

#### A-Ethernaut-GatekeeperThree

1. `gateOne()`:attack合约调用`construct0r()`即可,这里方法写错了,并不是构造器.
2. `gateTwo()`:调用`getAllowance(uint256 _password)`来校验trick的密码,因为`password`是`private`的,可以通过`storage slot`获取,poc里是带创建关卡一起的,所以直接用了`block.timestamp`
3. `gateThree()`:给合约转账大于0.001e,并且attack合约在接受e的时候`revert`就好 

### 2024.09.09

#### A-Ethernaut-Switch

这题需要调用`turnSwitchOn()`让`switchOn`=`true`,因为`turnSwitchOn()`有`onlyThis`修饰,所以只能通过`flipSwitch(bytes memory _data)`来调用.

`modifier onlyOff()`中判断了selector
```solidity
// we use a complex data type to put in memory
bytes32[1] memory selector;
// check that the calldata at position 68 (location of _data)
assembly {
    calldatacopy(selector, 68, 4) // grab function selector from calldata
}
```
这里68是`calldata`中对应调用函数的`selector`的offset,然后4字节大小.
modifier中判断了`selector[0] == offSelector`要求必须是调用的`turnSwitchOff()`,这就陷入了僵局.

看一下正常调用
```solidity

switch.flipSwitch(abi.encodeWithSelector(Switch.turnSwitchOff.selector));

//calldata

// 0x 
// 30c13ade flipSwitch函数签名
// 0000000000000000000000000000000000000000000000000000000000000020 _data offset 0x20=32
// 0000000000000000000000000000000000000000000000000000000000000004 _data length
// 20606e1500000000000000000000000000000000000000000000000000000000 turnSwitchOff 

```
所以`calldatacopy(selector, 68, 4)`中68对应的`turnSwitchOff`通常情况下没问题,但是可以手动修改offset来绕过检查.
这里具体的calldata的细节见[文档](https://docs.soliditylang.org/en/latest/abi-spec.html),
动态类型的 calldata 编码，前 32 字节用于存储偏移量（offset）, 接下来的 32 字节用于存储长度（length），然后是用于存储值的区域.

```solidity
// 0x
//4  bytes: 30c13ade 
//32 bytes: 0000000000000000000000000000000000000000000000000000000000000060 offset 0x60=96
//32 bytes: 0000000000000000000000000000000000000000000000000000000000000004 length
//32 bytes: 20606e1500000000000000000000000000000000000000000000000000000000 turnSwitchOff

// 0x60: 
//32 bytes: 0000000000000000000000000000000000000000000000000000000000000004 length
//32 bytes: 76227e1200000000000000000000000000000000000000000000000000000000 turnSwitchOn

```
在原有基础上增加 `turnSwitchOn`的calldata 修改offset来正确调用`turnSwitchOn`同时也满足`modifier`的检查.


#### A-Ethernaut-HigherOrder

这题用的`solc`还是`0.6.x`的,abicoder还没有类型检查.`0.8.0`以上不会有这个问题(默认`abicoder`V2).

```solidity
abi.encodeWithSignature("registerTreasury(uint8)", 256);
```

#### A-Ethernaut-Stake

正常调用满足要求就行

`0xdd62ed3e`对应 erc20的`allowance(address owner, address spender)`

要过`bytesToUint(allowance) >= amount`的检查,先授权一遍就行.题目没有校验`call`的返回值,所以没有weth也是可以增加`totalStaked`的.

需要另一个账户stakeETH一次来满足`totalStaked must be greater than the Stake contract's ETH balance.`

到这里 `ethernaut`完结撒花,学到了很多.



### 2024.09.11

#### A-DamnVaulnerableDefi-UnStoppable

还是选择了A系列的DamnVaulnerableDefi,想做一些defi相关.

大概了解了什么是[ERC-4626](https://ethereum.org/zh/developers/docs/standards/tokens/erc-4626/)

卡住了(搞了好久依赖,和之前的ethernaut的冲突)


### 2024.09.12

#### A-DamnVaulnerableDefi-UnStoppable
题目要求闪电贷失败.
```solidity
if (convertToShares(totalSupply) != balanceBefore) revert InvalidBalance();
```
把player的token转给vault就好

### 2024.09.13

#### A-DamnVaulnerableDefi-NaiveReceiver

又卡住了... 

### 2024.09.14

#### A-DamnVaulnerableDefi-NaiveReceiver

pool合约中的msgSender

```solidity

function _msgSender() internal view override returns (address) {
    if (msg.sender == trustedForwarder && msg.data.length >= 20) {
        return address(bytes20(msg.data[msg.data.length - 20:]));
    } else {
        return super._msgSender();
    }
}

```
如果来源是 `trustedForwarder`合约的话并且data.length>=20,返回的地址是获取`msg.data`最后20个字节来作为地址.

而这个msgSender又是`withdraw`方法需要的

这里需要去了解EIP712,看看具体怎么个调用法

这道题,参考的sun哥的.实在是卡在这了,今天累了,明天再整理下,不会组织语言了.


<!-- Content_END -->
