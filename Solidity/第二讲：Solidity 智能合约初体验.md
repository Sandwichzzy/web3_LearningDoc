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

    event MintCapNumeratorChanged(address indexed from, uint256 previousMintCapNumerator, uint256 newMintCapNumerator);

    error TheWeb3Token_ImproperlyInitialized();

    error TheWeb3Token_MintAmountTooLarge(uint256 amount, uint256 maximumAmount);

    error TheWeb3Token_NextMintTimestampNotElapsed(uint256 currentTimestamp, uint256 nextMintTimestamp);

    error TheWeb3Token_MintCapNumeratorTooLarge(uint256 numerator, uint256 maximumNumerator);

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

    function setMintCapNumerator(uint256 _mintCapNumerator) public onlyOwner {
        if (_mintCapNumerator > MINT_CAP_MAX_NUMERATOR) {
            revert TheWeb3Token_MintCapNumeratorTooLarge(_mintCapNumerator, MINT_CAP_MAX_NUMERATOR);
        }
        emit MintCapNumeratorChanged(msg.sender, mintCapNumerator, _mintCapNumerator);
        mintCapNumerator = _mintCapNumerator;
    }
}
```