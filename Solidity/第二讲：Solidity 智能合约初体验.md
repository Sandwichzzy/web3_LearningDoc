# 一.内容提要

- 合约简介
- 合约文件结构
- 合约的定义
- Remix 使用初体验
- HardHat 使用介绍
- Foundry 使用介绍
- 合约，合约 ABI 和 CallData 的关系

# 二.Solidity 智能合约的简介

Solidity 是一种面向智能合约的高级编程语言，是在 EVM 链上开发，部署和运行的智能合约代码，由 Ethereum 团队开发，创建和管理，主要是为简化智能合约开发的创建和管理，语言特点如下：

- **面向合约编程：**Solidity 专为编写智能合约设计，这些合约运行在以太坊虚拟机（EVM）上，执行和管理区块链上的交易和协议。
- **静态类型：**Solidity 是一种静态类型语言，变量类型在编译时确定，提供了更严格的类型检查和更高的代码安全性。
- **合约继承：**Solidity 支持合约继承，允许开发者创建可重用的合约组件，从而简化代码结构和提高代码的可维护性。
- **库和接口：**Solidity 提供了库和接口功能，帮助开发者模块化代码，提高代码的重用性和可维护性。
- **事件和日志：**Solidity 支持事件和日志功能，可以在合约执行过程中生成日志记录，方便在区块链上进行事件追踪和调试。
- **访问控制和权限管理：**Solidity 提供了灵活的访问控制机制，可以定义不同的访问权限和角色，确保合约的安全性。
- **ABI 和 CallData:** 根据合约的 ABI 构建 calldata 去调用合约

# 三.合约的文件结构

```Java
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;


import "@openzeppelin/contracts-upgradeable/token/ERC20/ERC20Upgradeable.sol";
import "@openzeppelin/contracts-upgradeable/token/ERC20/extensions/ERC20BurnableUpgradeable.sol";
import "@openzeppelin/contracts-upgradeable/access/OwnableUpgradeable.sol";
import "@openzeppelin/contracts-upgradeable/proxy/utils/Initializable.sol";

contract TheWebThree is Initializable, ERC20Upgradeable, ERC20BurnableUpgradeable, OwnableUpgradeable {
    string private constant NAME = "Holesky USDT";
    string private constant SYMBOL = "HUSDT";

    uint256 public constant MIN_MINT_INTERVAL = 365 days;

    uint256 public constant MINT_CAP_DENOMINATOR = 10_000;

    uint256 public constant MINT_CAP_MAX_NUMERATOR = 200;

    uint256 public mintCapNumerator;

    uint256 public nextMint;
    
    address public managerAddress;

    event MintCapNumeratorChanged(address indexed from, uint256 previousMintCapNumerator, uint256 newMintCapNumerator);

    error TheWeb3Token_ImproperlyInitialized();

    error TheWeb3Token_MintAmountTooLarge(uint256 amount, uint256 maximumAmount);

    error TheWeb3Token_NextMintTimestampNotElapsed(uint256 currentTimestamp, uint256 nextMintTimestamp);

    error TheWeb3Token_MintCapNumeratorTooLarge(uint256 numerator, uint256 maximumNumerator);

    modifier OnlyManager(){
        require(msg.send == managerAddress, "only manager can call this function");
        _;
    }
    
    constructor() {
        _disableInitializers();
    }

    function initialize(uint256 _initialSupply, address _owner) public initializer {
        if (_initialSupply == 0 || _owner == address(0)) revert TheWeb3Token_ImproperlyInitialized();

        __ERC20_init(NAME, SYMBOL);
        __ERC20Burnable_init();
        __Ownable_init(_owner);

        _mint(_owner, _initialSupply);

        nextMint = block.timestamp + MIN_MINT_INTERVAL;

        _transferOwnership(_owner);
    }

    function mint(address _recipient, uint256 _amount) public onlyOwner {
        uint256 maximumMintAmount = (totalSupply() * mintCapNumerator) / MINT_CAP_DENOMINATOR;
        if (_amount > maximumMintAmount) {
            revert TheWeb3Token_MintAmountTooLarge(_amount, maximumMintAmount);
        }
        if (block.timestamp < nextMint) revert TheWeb3Token_NextMintTimestampNotElapsed(block.timestamp, nextMint);

        nextMint = block.timestamp + MIN_MINT_INTERVAL;
        super._mint(_recipient, _amount);
    }

    function setMintCapNumerator(uint256 _mintCapNumerator) public OnlyManager {
        if (_mintCapNumerator > MINT_CAP_MAX_NUMERATOR) {
            revert TheWeb3Token_MintCapNumeratorTooLarge(_mintCapNumerator, MINT_CAP_MAX_NUMERATOR);
        }
        emit MintCapNumeratorChanged(msg.sender, mintCapNumerator, _mintCapNumerator);
        mintCapNumerator = _mintCapNumerator;
    }
}
```

- 版本声明： 指定合约编译器版本

```Plain
pragma solidity ^0.8.13;
```

- 导入声明：如果合约依赖其他代码库，可以使用 `import` 关键字导入

```Python
import "@openzeppelin/contracts-upgradeable/token/ERC20/ERC20Upgradeable.sol";
import "@openzeppelin/contracts-upgradeable/token/ERC20/extensions/ERC20BurnableUpgradeable.sol";
import "@openzeppelin/contracts-upgradeable/access/OwnableUpgradeable.sol";
import "@openzeppelin/contracts-upgradeable/proxy/utils/Initializable.sol";
```

- 合约声明：使用 `contract` 关键字进行合约声明，合约的主体代码在大括号内，如果合约有继承的话，使用 `is` 关键字来处理, 下面的 TheWebThree 合约继承了 *`Initializable, ERC20Upgradeable, ERC20BurnableUpgradeable, OwnableUpgradeable`*

```Plain
contract TheWebThree is Initializable, ERC20Upgradeable, ERC20BurnableUpgradeable, OwnableUpgradeable {
   合约的代码逻辑
}
```

- 常量

```Java
string private constant NAME = "Holesky USDT";
string private constant SYMBOL = "HUSDT";

uint256 public constant MIN_MINT_INTERVAL = 365 days;

uint256 public constant MINT_CAP_DENOMINATOR = 10_000;

uint256 public constant MINT_CAP_MAX_NUMERATOR = 200;
```

- 状态变量：状态变量是存在在区块链的上，合约中可以访问修饰这些变量，如果是 `public` 修饰的变量从外部可以读取

```Java
uint256 public mintCapNumerator;

uint256 public nextMint;
```

- 合约事件定义，合约执行状态发生改变，一般都会抛出合约事件

```Plain
event MintCapNumeratorChanged(address indexed from, uint256 previousMintCapNumerator, uint256 newMintCapNumerator);
```

- 合约自定错误

```Plain
error TheWeb3Token_ImproperlyInitialized();

error TheWeb3Token_MintAmountTooLarge(uint256 amount, uint256 maximumAmount);

error TheWeb3Token_NextMintTimestampNotElapsed(uint256 currentTimestamp, uint256 nextMintTimestamp);

error TheWeb3Token_MintCapNumeratorTooLarge(uint256 numerator, uint256 maximumNumerator);
```

- 修饰符：限制函数调用权限， 代码 *`setMintCapNumerator`* *被修饰，只能这个* *`managerAddress`* *调用*

```Plain
 modifier OnlyManager(){
    require(msg.send == managerAddress, "only manager can call this function");
    _;
}
```

**构造函数：**构造函数在合约部署时执行，用于初始化状态变量。

```Solidity
constructor() {
    _disableInitializers();
}
```

**初始化函数:** 参数初始化，函数职能执行一次

```Bash
 function initialize(uint256 _initialSupply, address _owner) public initializer {
    if (_initialSupply == 0 || _owner == address(0)) revert TheWeb3Token_ImproperlyInitialized();

    __ERC20_init(NAME, SYMBOL);
    __ERC20Burnable_init();
    __Ownable_init(_owner);

    _mint(_owner, _initialSupply);

    nextMint = block.timestamp + MIN_MINT_INTERVAL;

    _transferOwnership(_owner);
}
```

**普通函数**： 满足一定条件就可以 *`mint`* *token* 

```Plain
 function mint(address _recipient, uint256 _amount) public onlyOwner {
    uint256 maximumMintAmount = (totalSupply() * mintCapNumerator) / MINT_CAP_DENOMINATOR;
    if (_amount > maximumMintAmount) {
        revert TheWeb3Token_MintAmountTooLarge(_amount, maximumMintAmount);
    }
    if (block.timestamp < nextMint) revert TheWeb3Token_NextMintTimestampNotElapsed(block.timestamp, nextMint);

    nextMint = block.timestamp + MIN_MINT_INTERVAL;
    super._mint(_recipient, _amount);
}
```

- 其他的结构， `interface` `abstract` 等等后面会讲到

# 四. 合约的定义

合约的定义使用的是 *`contract`* *定义格式如下*

```Plain
contract xxx {
    代码块
}
```

- xxx 代表的是合约的名字
- 代码块可以包含上面合约文件结构里面的任意结构

# 五.合约的运行之---Remix 使用初体验

真正的一个项目一般都是不会使用 Remix, 学习时候使用的小工具，提供丰富界面操作，但是功能并没有 foundry 和 hardhat 这样工具强大；实际中 hardhat 和 foundy 更使用大项目项目的管理。

使用 Remix 部署合约到 RootHash Chain, 下面是 RootHashChain 的网络信息

- 测试网 RPC 与浏览器
  - https://rpc-testnet.roothashpay.com
  - https://wss-testnet.roothashpay.com
  - https://explorer-testnet.roothashpay.com

- 主网 RPC 与浏览器
  - https://rpc.roothashpay.com
  - https://wss.roothashpay.com
  - https://explorer.roothashpay.com

## 1.Remix 本地网络部署

- 操作流程请看视频

## 2.将 Remix 自带的合约部署到 RootHash Chain

将 RootHash Chain 的 RPC 链接添加到 MetaMask, 链接 MetaMask 部署合约即可,  操作流程请回看视频

## 3.用 Remix 写一个简单合约进行部署操作

- 代码如下

```Java
// SPDX-License-Identifier: GPL-3.0

pragma solidity >=0.8.2 <0.9.0;

contract TheWeb3First {

    uint256 public number;

    constructor(uint256 _number) {
        number = _number;
    }

    function add(uint256 a, uint256 b) public pure returns (uint256){
        return a + b;
    }

  
    function mul(uint256 a, uint256 b) public pure returns (uint256){
        return a * b;
    }

    function sub(uint256 a, uint256 b) public pure returns (uint256) {
        return  a - b;
    }

     function div(uint256 a, uint256 b) public pure returns (uint256) {
        return  a / b;
    }

     function returnNumber() public view returns (uint256){
        return number;
    }
}
```

# 六. EVM 链的 RPC 节点

## 1. 开放 RPC 节点的寻找办法

开放节点能满足日常的开发需求，但是当数据请求量过大或者生产环境都不会使用开发的 RPC 

### 1.1 项目方的官方文档

- Cpchain: https://cpchain.gitbook.io/cpchaingitbook/user-guides/connecting-wallet-to-cp-chain
- Mantle: https://docs.mantle.xyz/network/for-developers/resources-and-tooling/node-endpoints-and-providers
- Manta: https://docs.manta.network/docs/manta-pacific/Tools/Node%20Providers

### 1.2 Chainlist 

- https://chainlist.org/
- 搜索 manta, 在红线可以找到

![chainlist_1](imgs/chainlist_1.png)

- 如果需要测试网，include 测试网就行

![chainlist_2](imgs/chainlist_2.png)

## 2. 付费 RPC 

如果需要大批量数据请求或者生产环境使用，需要自建 RPC 或者使用付费 RPC 

- 使用付费 RPC 可以选择的厂商
  - https://www.alchemy.com/ 稳定性比较高的厂商
  - https://www.ankr.com/ 稳定性比较高的厂商
  - https://getblock.io/
- 自建 RPC 节点
  - 按照官方文档的指导搭建，但是需要做负载均衡

![自建RPC负载均衡](imgs/自建RPC负载均衡.png)

# 七.合约，合约 ABI 和 CallData 的关系

在 Remix 里面我们提到了几个东西，一个是合约，一个 abi,  还有 calldata, 那么合约，abi 和 calldata 之间的关系是什么呢

![合约 ABI 和 CallData](imgs/合约 ABI 和 CallData.png)

- Solidity 智能合约部署到链上是形成字节码
- Solidity 智能合约编译之后得到是 abi 文件，abi 里面也含有合约字节码
- 调用者使用 ABI 生成调用 calldata 放到交易里面发送到链上，链上回去解析 calldata 执行对应合约逻辑

