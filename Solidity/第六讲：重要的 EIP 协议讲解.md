# 一.内容提要

- [x] ERC20 
- [x] ERC721 与 ERC1155
- [x] EIP1559
- [x] EIP-1167 最小代理 clone （代理章节）
- [x] EIP-1967  信标代理 （代理章节）
- [x] EIP-2535  钻石代理 Diamond Standard （代理章节）
- [ ] EIP2930
- [ ] EIP3643
- [ ] EIP4337
- [ ] EIP4844
- [ ] EIP7702

# 二. ERC20 和 ERC721, ERC1155

## 1. 同质化代币

同质化代币是可以进行分割，例如 BTC，ETH 这样的 Token, 是可以切割，可以是 0.1，也可以 0.01，也可以是 1 个

## 2. 半同质化代币

半同质化化代币处于同质化代币非同质化代币之间，主要为了解决 NFT 的不可分割问题，提出可以交易包形式去售卖 NFT

- 一栋房子
  - 客厅
  - 厨房
  - 卧室
  - 这种交易包的形式交易
- 一栋房子
  - 总价值评估出来，发行可以交易的不同种 NFT，NFT 包就是这个房子

## 3. 非同质化代币

非同质化代币是独一无二，不可以分割的，例如，一张图片，一栋房子，一个碗，一旦分割，就是会损失原有的价值

## 4. ETH 的 EIP 和 ERC 协议

**ERC:** **Ethereum Request for Comments（以太坊征求意见提案）**的缩写，用以记录以太坊上应用级的各种开发标准和协议。ERC协议标准是影响以太坊发展的重要因素, 像`ERC20`, `ERC721`, 等, 都是对以太坊生态产生了很大影响。

**EIP: (Ethereum Improvement Proposals 以太坊升级提案)** 是以太坊开发者社区提出的改进建议, 是一系列以编号排定的文件, 类似互联网上IETF的RFC。EIP 可以是 Ethereum 生态中任意领域的改进, 比如新特性、ERC、协议改进、编程工具等等。

所以最终结论：`EIP`包含`ERC`。

### 4.1 什么是 ERC

ERC 是 Ethereum Request for Comments 的首字母缩写词。它就像技术文档，定义了适用于希望利用以太坊生态系统的一组开发人员和用户的方法、行为、创新和研究。 您可能想知道谁有权创建和管理 ERC。以太坊的智能合约程序员负责编写与 ERC 相关的文档，以描述每个基于以太坊的代币必须遵守的规则集。他们还经常审查这些文件并对其进行评论以进一步改进。 要轻松理解 ERC，请考虑一个工程任务组，它向开发人员传达技术说明和规则，如果每个人都想利用特定生态系统的好处，他们都需要遵守这些规则。

### 4.2 什么是 ERC 代币标准

ERC 代币标准解释了建立在以太坊区块链上的所有 ERC 代币的某些规则。以太坊社区对这套规则进行适当的审查，并根据不断变化的需求进行修改。此外，ERC 标准旨在允许 ERC 代币无缝交互。

ERC-20、ERC-721 和 ERC-1155 是三种流行的 ERC 代币标准或协议，它们在主要行业都有应用。以太坊社区完全认可这些代币标准，它们在具体特性和功能方面有所不同。

在了解令牌标准的确切含义或它的工作原理之前，我们应该首先了解以太坊智能合约标准的要领。以下语句对其进行了定义：

- 智能合约描述了智能合约程序员应该遵守的规则，以利用以太坊网络的潜在利益。
- 这些标准适用于支持智能合约和去中心化应用程序（dApps）开发的区块链。
- 智能合约标准包含代币标准、库主题和格式、名称注册表和相关详细信息。

ERC 代币标准只是智能合约标准的另一个名称。以太坊上的智能合约必须遵守标准或规则，才能实现代币创建、交易处理、支出等基本功能。通过引入改进的 ERC 标准，以太坊释放其生态系统的真正潜力，并赋能更具体的智能合同，有助于网络的发展。

## 5. ERC20

- ERC20 Ethereum 发行 token 一种标准协议，他里面定义 token 发行标准；

  - 标准的 ERC20 协议具备状态变量

  - ```Java
    mapping(address account => uint256) private _balances;
    
    mapping(address account => mapping(address spender => uint256)) private _allowances;
    
    //代币信息（可选）：名称(name())，代号(symbol())，小数位数(decimals())
    uint256 private _totalSupply;
    
    string private _name;
    string private _symbol;
    ```

  - 事件

  - ```
    /** 
    * @dev 释放条件：当 `value` 单位的货币从账户 (`from`) 转账到另一账户 (`to`)时. 
    */ 
    event Transfer(address indexed from, address indexed to, uint256 value); 
    /** 
    * @dev 释放条件：当 `value` 单位的货币从账户 (`owner`) 授权给另一账户 (`spender`)时. 
    */ 
    event Approval(address indexed owner, address indexed spender, uint256 value);
    ```

  - 具备的方法

  - ```SQL
    function totalSupply() external view returns (uint256); // 返回代币总供给
    function balanceOf(address account) external view returns (uint256); //返回账户`account`所持有的代币数
    
    //转账 `amount` 单位代币，从调用者账户到另一账户 `to`.如果成功，返回 `true`.释放 {Transfer} 事件.
    function transfer(address to, uint256 value) external returns (bool);
    
    //返回授权额度 :返回`owner`账户授权给`spender`账户的额度，默认为0.当{approve} 或 {transferFrom} 被调用时，`allowance`会改变
    function allowance(address owner, address spender) external view returns (uint256); 
    
    //授权: 调用者账户给`spender`账户授权 `amount`数量代币。如果成功，返回 `true`.释放 {Approval} 事件.
    function approve(address spender, uint256 value) external returns (bool); 
    
    //通过授权机制，从`from`账户向`to`账户转账`amount`数量代币。转账的部分会从调用者的`allowance`中扣除。如果成功，返回 `true`. 释放 
    // {Transfer} 事件.
    function transferFrom(address from, address to, uint256 value) external returns (bool); //授权转账
    ```

## 6. ERC721 和 ERC1155

Cryptokitties（广泛使用的不可替代代币）的创始人兼首席技术官 Dieter Shirley 最初提议开发一种新的代币类型来支持 NFT。该提案将于 2018 年晚些时候获得批准。它专门用于 NFT，这意味着按照 ERC-721 规则开发的代币可以代表以太坊区块链上任何数字资产的价值。

有了这个，我们得出一个结论：如果 ERC-20 对于发明新的加密货币至关重要，那么 ERC-721 对于代表某人对这些资产的所有权的数字资产来说是无价的。ERC-721 可以表示以下内容：

- 独一无二的数字艺术作品
- 推文和社交媒体帖子
- 游戏内收藏品
- 游戏角色
- 任何卡通人物和数百万其他 NFT ......

这种特殊类型的 Token 为使用 NFT 的企业带来了惊人的可能性。同样，ERC-721 也为他们带来了挑战，为了应对这些挑战，ERC-721 标准开始发挥作用。

请注意，每个 NFT 都有一个称为 tokenId 的 uint256 变量。因此，对于每个 EBR-721 合约，对合约地址 - uint256 tokenId 必须是唯一的。

此外，dApps 还应该有一个“转换器”来规范 NFT 的输入和输出过程。例如，转换器将 tokenId 视为输入并输出不可替代的令牌，例如僵尸、杀戮、游戏收藏品等的图像。

### 6.1 ERC-721 代币的特征 

https://github.com/AmazingAng/WTF-Solidity/tree/main/34_ERC721

- ERC-721 代币是不可替代代币 (NFT) 的标准。
- 这些代币不能交换任何同等价值的东西，因为它们是独一无二的。
- 每个 ERC-721 代表各自 NFT 的价值，可能不同。
- ERC-721 代币最受欢迎的应用领域是游戏中的 NFT。

### 6.2 标准的 ERC721 协议

- 状态变量

```Java
// Token名称
string public override name;
// Token代号
string public override symbol;
// tokenId 到 owner address 的持有人映射
mapping(uint => address) private _owners;
// address 到 持仓数量 的持仓量映射
mapping(address => uint) private _balances;
// tokenID 到 授权地址 的授权映射
mapping(uint => address) private _tokenApprovals;
// owner地址 到 operator地址 的批量授权映射
mapping(address => mapping(address => bool)) private _operatorApprovals;
```

- 标准的方法

```Java
// SPDX-License-Identifier: MIT
// by 0xAA
pragma solidity ^0.8.21;

import "./IERC165.sol";
import "./IERC721.sol";
import "./IERC721Receiver.sol";
import "./IERC721Metadata.sol";
import "./String.sol";

contract ERC721 is IERC721, IERC721Metadata{
    using Strings for uint256; // 使用Strings库，

    // Token名称
    string public override name;
    // Token代号
    string public override symbol;
    // tokenId 到 owner address 的持有人映射
    mapping(uint => address) private _owners;
    // address 到 持仓数量 的持仓量映射
    mapping(address => uint) private _balances;
    // tokenID 到 授权地址 的授权映射
    mapping(uint => address) private _tokenApprovals;
    // owner地址 到 operator地址 的批量授权映射
    mapping(address => mapping(address => bool)) private _operatorApprovals;

    // 错误 无效的接收者
    error ERC721InvalidReceiver(address receiver);

    /**
     * 构造函数，初始化`name` 和`symbol` .
     */
    constructor(string memory name_, string memory symbol_) {
        name = name_;
        symbol = symbol_;
    }

    // 实现IERC165接口supportsInterface
    function supportsInterface(bytes4 interfaceId)
        external
        pure
        override
        returns (bool)
    {
        return
            interfaceId == type(IERC721).interfaceId ||
            interfaceId == type(IERC165).interfaceId ||
            interfaceId == type(IERC721Metadata).interfaceId;
    }

    // 实现IERC721的balanceOf，利用_balances变量查询owner地址的balance。
    function balanceOf(address owner) external view override returns (uint) {
        require(owner != address(0), "owner = zero address");
        return _balances[owner];
    }

    // 实现IERC721的ownerOf，利用_owners变量查询tokenId的owner。
    function ownerOf(uint tokenId) public view override returns (address owner) {
        owner = _owners[tokenId];
        require(owner != address(0), "token doesn't exist");
    }

    // 实现IERC721的isApprovedForAll，利用_operatorApprovals变量查询owner地址是否将所持NFT批量授权给了operator地址。
    function isApprovedForAll(address owner, address operator)
        external
        view
        override
        returns (bool)
    {
        return _operatorApprovals[owner][operator];
    }

    // 实现IERC721的setApprovalForAll，将持有代币全部授权给operator地址。调用_setApprovalForAll函数。
    function setApprovalForAll(address operator, bool approved) external override {
        _operatorApprovals[msg.sender][operator] = approved;
        emit ApprovalForAll(msg.sender, operator, approved);
    }

    // 实现IERC721的getApproved，利用_tokenApprovals变量查询tokenId的授权地址。
    function getApproved(uint tokenId) external view override returns (address) {
        require(_owners[tokenId] != address(0), "token doesn't exist");
        return _tokenApprovals[tokenId];
    }
     
    // 授权函数。通过调整_tokenApprovals来，授权 to 地址操作 tokenId，同时释放Approval事件。
    function _approve(
        address owner,
        address to,
        uint tokenId
    ) private {
        _tokenApprovals[tokenId] = to;
        emit Approval(owner, to, tokenId);
    }

    // 实现IERC721的approve，将tokenId授权给 to 地址。条件：to不是owner，且msg.sender是owner或授权地址。调用_approve函数。
    function approve(address to, uint tokenId) external override {
        address owner = _owners[tokenId];
        require(
            msg.sender == owner || _operatorApprovals[owner][msg.sender],
            "not owner nor approved for all"
        );
        _approve(owner, to, tokenId);
    }

    // 查询 spender地址是否可以使用tokenId（需要是owner或被授权地址）
    function _isApprovedOrOwner(
        address owner,
        address spender,
        uint tokenId
    ) private view returns (bool) {
        return (spender == owner ||
            _tokenApprovals[tokenId] == spender ||
            _operatorApprovals[owner][spender]);
    }

    /*
     * 转账函数。通过调整_balances和_owner变量将 tokenId 从 from 转账给 to，同时释放Transfer事件。
     * 条件:
     * 1. tokenId 被 from 拥有
     * 2. to 不是0地址
     */
    function _transfer(
        address owner,
        address from,
        address to,
        uint tokenId
    ) private {
        require(from == owner, "not owner");
        require(to != address(0), "transfer to the zero address");

        _approve(owner, address(0), tokenId);

        _balances[from] -= 1;
        _balances[to] += 1;
        _owners[tokenId] = to;

        emit Transfer(from, to, tokenId);
    }
    
    // 实现IERC721的transferFrom，非安全转账，不建议使用。调用_transfer函数
    function transferFrom(
        address from,
        address to,
        uint tokenId
    ) external override {
        address owner = ownerOf(tokenId);
        require(
            _isApprovedOrOwner(owner, msg.sender, tokenId),
            "not owner nor approved"
        );
        _transfer(owner, from, to, tokenId);
    }

    /**
     * 安全转账，安全地将 tokenId 代币从 from 转移到 to，会检查合约接收者是否了解 ERC721 协议，以防止代币被永久锁定。调用了_transfer函数和_checkOnERC721Received函数。条件：
     * from 不能是0地址.
     * to 不能是0地址.
     * tokenId 代币必须存在，并且被 from拥有.
     * 如果 to 是智能合约, 他必须支持 IERC721Receiver-onERC721Received.
     */
    function _safeTransfer(
        address owner,
        address from,
        address to,
        uint tokenId,
        bytes memory _data
    ) private {
        _transfer(owner, from, to, tokenId);
        _checkOnERC721Received(from, to, tokenId, _data);
    }

    /**
     * 实现IERC721的safeTransferFrom，安全转账，调用了_safeTransfer函数。
     */
    function safeTransferFrom(
        address from,
        address to,
        uint tokenId,
        bytes memory _data
    ) public override {
        address owner = ownerOf(tokenId);
        require(
            _isApprovedOrOwner(owner, msg.sender, tokenId),
            "not owner nor approved"
        );
        _safeTransfer(owner, from, to, tokenId, _data);
    }

    // safeTransferFrom重载函数
    function safeTransferFrom(
        address from,
        address to,
        uint tokenId
    ) external override {
        safeTransferFrom(from, to, tokenId, "");
    }

    /** 
     * 铸造函数。通过调整_balances和_owners变量来铸造tokenId并转账给 to，同时释放Transfer事件。铸造函数。通过调整_balances和_owners变量来铸造tokenId并转账给 to，同时释放Transfer事件。
     * 这个mint函数所有人都能调用，实际使用需要开发人员重写，加上一些条件。
     * 条件:
     * 1. tokenId尚不存在。
     * 2. to不是0地址.
     */
    function _mint(address to, uint tokenId) internal virtual {
        require(to != address(0), "mint to zero address");
        require(_owners[tokenId] == address(0), "token already minted");

        _balances[to] += 1;
        _owners[tokenId] = to;

        emit Transfer(address(0), to, tokenId);
    }

    // 销毁函数，通过调整_balances和_owners变量来销毁tokenId，同时释放Transfer事件。条件：tokenId存在。
    function _burn(uint tokenId) internal virtual {
        address owner = ownerOf(tokenId);
        require(msg.sender == owner, "not owner of token");

        _approve(owner, address(0), tokenId);

        _balances[owner] -= 1;
        delete _owners[tokenId];

        emit Transfer(owner, address(0), tokenId);
    }

    // _checkOnERC721Received：函数，用于在 to 为合约的时候调用IERC721Receiver-onERC721Received, 以防 tokenId 被不小心转入黑洞。
    function _checkOnERC721Received(address from, address to, uint256 tokenId, bytes memory data) private {
        if (to.code.length > 0) {
            try IERC721Receiver(to).onERC721Received(msg.sender, from, tokenId, data) returns (bytes4 retval) {
                if (retval != IERC721Receiver.onERC721Received.selector) {
                    revert ERC721InvalidReceiver(to);
                }
            } catch (bytes memory reason) {
                if (reason.length == 0) {
                    revert ERC721InvalidReceiver(to);
                } else {
                    /// @solidity memory-safe-assembly
                    assembly {
                        revert(add(32, reason), mload(reason))
                    }
                }
            }
        }
    }

    /**
     * 实现IERC721Metadata的tokenURI函数，查询metadata。
     */
    function tokenURI(uint256 tokenId) public view virtual override returns (string memory) {
        require(_owners[tokenId] != address(0), "Token Not Exist");

        string memory baseURI = _baseURI();
        return bytes(baseURI).length > 0 ? string(abi.encodePacked(baseURI, tokenId.toString())) : "";
    }

    /**
     * 计算{tokenURI}的BaseURI，tokenURI就是把baseURI和tokenId拼接在一起，需要开发重写。
     * BAYC的baseURI为ipfs://QmeSjSinHpPnmXmspMjwiXyN6zS4E9zccariGR3jxcaWtq/ 
     */
    function _baseURI() internal view virtual returns (string memory) {
        return "";
    }
}
```

### 6.3 ERC1155

 (https://github.com/AmazingAng/WTF-Solidity/tree/main/40_ERC1155)

不论是`ERC20`还是`ERC721`标准，每个合约都对应一个独立的代币。假设我们要在以太坊上打造一个类似《魔兽世界》的大型游戏，这需要我们对每个装备都部署一个合约。上千种装备就要部署和管理上千个合约，这非常麻烦。因此，[以太坊EIP1155](https://eips.ethereum.org/EIPS/eip-1155)提出了一个多代币标准`ERC1155`，允许一个合约包含多个同质化和非同质化代币。`ERC1155`在GameFi应用最多，Decentraland、Sandbox等知名链游都使用它。

简单来说，`ERC1155`与之前介绍的非同质化代币标准类似：在`ERC721`中，每个代币都有一个`tokenId`作为唯一标识，每个`tokenId`只对应一个代币；而在`ERC1155`中，每一种代币都有一个`id`作为唯一标识，每个`id`对应一种代币。这样，代币种类就可以非同质的在同一个合约里管理了，并且每种代币都有一个网址`uri`来存储它的元数据，类似`ERC721`的`tokenURI`。

```
/** 
* @dev ERC1155的可选接口，加入了uri()函数查询元数据 
*/ 
interface IERC1155MetadataURI is IERC1155 {    
    /**     
    * @dev 返回第`id`种类代币的URI     
    */    
    function uri(uint256 id) external view returns (string memory); 
}
```

那么怎么区分`ERC1155`中的某类代币是同质化还是非同质化代币呢？其实很简单：如果某个`id`对应的代币总量为`1`，那么它就是非同质化代币，类似`ERC721`；如果某个`id`对应的代币总量大于`1`，那么他就是同质化代币，因为这些代币都分享同一个`id`，类似`ERC20`。

#### 6.3.1 ERC-1155 代币的特征

- ERC-1155 是一个智能合约接口，代表可替代、半可替代和不可替代的代币。
- ERC-1155 可以执行 ERC-20 和 ERC-720 的功能，甚至可以同时执行两者。
- 每个代币可以根据代币的性质代表不同的价值；可替代的、半可替代的或不可替代的。
- ERC-1155 适用于创建 NFT、可兑换购物券、ICO 等。

#### 6.3.2 标准协议实现 ERC1155主合约

- ERC1155主合约包含4个状态变量：

```Java
name：代币名称
symbol：代币代号
_balances：代币持仓映射，记录代币种类id下某地址account的持仓量balances。
_operatorApprovals：批量授权映射，记录持有地址给另一个地址的授权情况。
```

- ERC1155主合约包含16个函数：

```C++
构造函数：初始化状态变量name和symbol。
supportsInterface()：实现ERC165标准，声明它支持的接口，供其他合约检查。
balanceOf()：实现IERC1155的balanceOf()，查询持仓量。与ERC721标准不同，这里需要输入查询的持仓地址account以及币种id。
balanceOfBatch()：实现IERC1155的balanceOfBatch()，批量查询持仓量。
setApprovalForAll()：实现IERC1155的setApprovalForAll()，批量授权，释放ApprovalForAll事件。
isApprovedForAll()：实现IERC1155的isApprovedForAll()，查询批量授权信息。
safeTransferFrom()：实现IERC1155的safeTransferFrom()，单币种安全转账，释放TransferSingle事件。与ERC721不同，这里不仅需要填发出方from，接收方to，代币种类id，还需要填转账数额amount。
safeBatchTransferFrom()：实现IERC1155的safeBatchTransferFrom()，多币种安全转账，释放TransferBatch事件。
_mint()：单币种铸造函数。
_mintBatch()：多币种铸造函数。
_burn()：单币种销毁函数。
_burnBatch()：多币种销毁函数。
_doSafeTransferAcceptanceCheck：单币种转账的安全检查，被safeTransferFrom()调用，确保接收方为合约的情况下，实现了onERC1155Received()函数。
_doSafeBatchTransferAcceptanceCheck：多币种转账的安全检查，，被safeBatchTransferFrom调用，确保接收方为合约的情况下，实现了onERC1155BatchReceived()函数。
uri()：返回ERC1155的第id种代币存储元数据的网址，类似ERC721的tokenURI。
baseURI()：返回baseURI，uri就是把baseURI和id拼接在一起，需要开发重写。
```

# 四.EIP1559 和 EIP4844

## 1. 主流的几种交易类型

### 1.1 Legacy 交易

在以太坊上，Legacy 交易是指在EIP-1559之前的**传统交易类型**。这些交易类型依赖于**第一价格拍卖机制**来确定交易费用。以下是对Legacy交易类型的详细介绍：

#### 1.1.1 Legacy交易的结构

Legacy交易的结构包括以下字段：

- **nonce**：发送方账户的交易序号
- **gasPrice**：每单位 gas 的价格
- **gasLimit**：交易消耗的最大 gas 量
- **to**：接收方地址
- **value**：转移的以太币数量
- **data**：交易的附加数据（可选）
- **v, r, s**：交易的签名

#### 1.1.2 交易字段详细说明

- **nonce**：每个账户有一个递增的nonce值。每次发送交易，nonce值都会增加，以确保交易的唯一性和顺序性。例如，账户发送的第一笔交易nonce为0，第二笔为1，依此类推。
- **gasPrice**：这是用户愿意为每个gas单位支付的费用。矿工根据gasPrice的高低优先打包交易。gasPrice的单位为gwei（1 ETH = 10^9 gwei）。高gasPrice通常会导致交易更快被确认。
- **gasLimit**：这是用户愿意为交易支付的最大gas量。它限制了交易执行过程中可以消耗的gas总量。设置gasLimit过低可能导致交易失败（但仍需支付已消耗的gas），而设置过高则可能浪费资金。
- **to**：接收方的以太坊地址。对于普通交易，这是一个常规的地址。对于合约创建交易，此字段为空，表示将部署一个新合约。
- **value**：表示转移的以太币数量，单位为wei（1 ETH = 10^18 wei）。用户可以在交易中指定转移的金额。
- **data**：这是一个可选字段，包含交易的附加数据。对于调用合约函数的交易，data字段包含合约调用的详细信息。对于合约创建交易，data字段包含合约的字节码。
- **v, r, s**：这些是签名字段，用于验证交易的发送者。通过这些字段，可以确认交易确实是由拥有相应私钥的账户发送的。

#### 1.1.2 Legacy交易的执行流程

- **交易创建**：用户创建交易并指定所有必要的字段，包括nonce、gasPrice、gasLimit、to、value和data。
- **交易签名**：用户使用其私钥对交易进行签名，生成v、r、s字段。
- **交易广播**：签名后的交易被广播到以太坊网络。
- **交易验证**：矿工节点接收交易并进行验证，包括检查nonce、签名和账户余额等。
- **交易打包**：矿工将交易打包到区块中，优先打包gasPrice较高的交易。
- **交易执行**：区块被挖出后，交易在以太坊虚拟机（EVM）中执行，根据交易类型（转账或合约调用）改变账户状态。
- **交易确认**：交易被多个区块确认后，交易被视为最终确定。

#### 1.1.4 Legacy交易的优势与劣势

- 优势
  - **简单易理解**：交易费用机制简单明了，用户通过指定gasPrice来确定交易费用。
  - **灵活性**：用户可以根据需要灵活调整gasPrice以提高交易确认速度。
- 劣势
  - **费用波动大**：在网络拥堵时，交易费用可能会迅速飙升，导致用户难以预测和控制费用。
  - **用户体验差**：用户需要手动调整gasPrice以确保交易被及时确认，增加了使用的复杂性。
  - **矿工激励不稳定**：矿工可能会优先选择高gasPrice的交易，导致低费用交易长时间等待确认。

### 1.2 EIP1559 交易

EIP-1559 是以太坊改进提案中的一个重要提案，它对以太坊的交易费用机制进行了重大改进，旨在**解决交易费用的不确定性和网络拥堵**问题。以下是对 EIP-1559 交易的详细介绍：

#### 1.2.1 EIP-1559 的背景

在 EIP-1559 之前，以太坊的交易费用是通过“第一价格拍卖”机制确定的，即用户在交易中指定愿意支付的 gas 价格，矿工优先打包出价最高的交易。这种机制导致以下问题：

- **费用波动**：当网络拥堵时，交易费用可能会迅速飙升。
- **用户体验差**：用户很难确定合适的 gas 价格，经常要么支付过高的费用，要么交易迟迟未被确认。

#### 1.2.2 EIP-1559 的改进

EIP-1559 引入了一种新的费用结构，包括基本费用（base fee）、小费（maxPriorityFeePerGas）和 gas 限额（gas limit），以改善上述问题。

**基本费用**：基本费用是根据网络需求动态调整的最低费用。每个区块都会设定一个基本费用，这个费用会根据网络的拥堵情况自动调整：

- **机制**：基本费用在区块容量达到目标值时增加，在区块容量低于目标值时减少。
- **目的**：使交易费用更加可预测，减少用户手动调整 gas 价格的需求。

**小费：**用户可以选择支付额外的小费（maxPriorityFeePerGas）给矿工，以激励矿工优先处理他们的交易：

- **功能**：提高交易被打包的优先级。
- **灵活性**：用户可以根据需要设置小费的高低。

**最大费用：**用户在交易中指定愿意支付的最大费用（maxFeePerGas），包括基本费用和小费：

- **保障**：确保用户不会支付超过预期的费用。
- **计算**：实际支付的费用为基本费用加上小费，剩余部分退还给用户。

#### 1.2.3 EIP-1559 交易的结构

- **nonce**：发送方账户的交易序号。
- **maxFeePerGas**：**总费用上限**。你愿意为每个单位的 Gas 支付的**绝对最大值**（以 wei/gas 为单位）。它必须大于或等于 `(Base Fee + maxPriorityFeePerGas)`。
- **maxPriorityFeePerGas**：用户愿意支付的每单位 gas 的小费, 这部分费用直接付给矿工/验证者。
- **gasLimit**：交易允许消耗的最大 gas 量。
- **to**：接收方地址。
- **value**：交易中发送的以太币数量。
- **data**：交易附加数据，通常为合约调用数据。
- **v, r, s**：交易的签名。

#### 1.2.4 EIP-1559 交易的执行流程

- **交易创建**：用户创建一个包含 maxFeePerGas 和 maxPriorityFeePerGas 的交易。
- **交易签名**：用户使用其私钥对交易进行签名，生成v、r、s字段。
- **交易广播**：签名后的交易被广播到以太坊网络。
- **交易验证**：矿工节点接收交易并进行验证，包括检查nonce、签名和账户余额等。
- **基本费用确定**：矿工根据网络状态确定当前区块的基本费用。
- **费用计算**：矿工从用户提供的 maxFeePerGas 中扣除基本费用，剩余部分作为小费。
- **交易打包**：矿工优先打包小费较高的交易，以最大化收益。
- **交易确认**：交易被打包到区块中，用户支付的总费用为基本费用加小费。

#### 1.2.5 EIP-1559 的优势

- **费用预测性**：基本费用的动态调整使得交易费用更加可预测，改善用户体验。
- **网络稳定性**：通过自动调整基本费用，EIP-1559 有助于缓解网络拥堵。
- **矿工激励**：小费机制仍然为矿工提供了足够的激励，确保网络安全。

#### 1.2.6 实际应用中的变化

自 EIP-1559 实施以来，以太坊的交易费用结构发生了显著变化：

- **用户体验改善**：用户不再需要频繁调整 gas 价格，交易费用更加稳定。
- **费用燃烧机制**：基本费用部分被销毁（burned），减少了以太币的供应，可能对 ETH 的长期价值产生积极影响。

### 1.3 Legacy 交易与 EIP-1559 交易的对比

EIP-1559交易引入了基本费用（base fee）和小费（priority fee）机制，改善了Legacy交易中的一些问题：

- **费用预测性**：EIP-1559的基本费用根据网络需求动态调整，使交易费用更加可预测。
- **用户体验**：用户不再需要频繁调整gasPrice，简化了交易的创建和提交过程。
- **费用销毁机制**：EIP-1559中的基本费用部分被销毁，减少了以太币的供应，可能对ETH的长期价值产生积极影响。

总结来说，Legacy交易是以太坊早期采用的交易机制，尽管简单直接，但在网络拥堵和用户体验方面存在明显不足。EIP-1559交易通过引入新的费用机制，显著改善了这些问题，提高了网络的整体性能和用户体验。

| 特性          | Legacy 交易                    | EIP-1559 交易（Type 2）                |
| :------------ | :----------------------------- | :------------------------------------- |
| **类型字段**  | 无                             | `0x02`                                 |
| **费用模型**  | 单一 `gasPrice`                | `Base Fee` + `maxPriorityFeePerGas`    |
| **Gas Price** | 由用户设定，是全部给矿工的费用 | 分为基础费（被销毁）和优先费（给矿工） |
| **费用预测**  | 波动大，依赖第一价格拍卖       | 更稳定，基础费由协议设定               |

### 1.4 EIP-4844 和 Blob 交易

在以太坊扩展方案中，Rollups 提供了一种有效的方法来提高网络的吞吐量和降低交易费用。Rollups 主要有两种类型：Optimistic Rollups 和 ZK-Rollups。在这些扩展方案中，blob 交易是一种新兴的概念，特别是与以太坊改进提案EIP-4844 相关。EIP-4844，也被称为“Proto-Danksharding”，引入了一种新的交易类型，称为 blob-carrying transactions（携带数据块的交易），用于更高效地存储和传输大量数据 blobs。

#### 1.4.1 Blob 交易的特点

- **高效的数据传输**：Blob 交易允许 Rollups 提交大块数据（blobs）到以太坊主链。这些数据包含了 Rollup 内部的所有交易详细信息。
- **独立于主链状态**：Blob 数据不会直接影响以太坊主链的状态，这意味着它们不会增加主链的存储需求。这些数据主要用于验证和重建 Rollup 的状态。
- **费用优化**：由于 blob 数据不直接影响主链状态，处理这些数据的费用通常较低，从而降低了整体交易成本。

#### 1.4.2 Blob 交易类别

##### 1.4.2.1 数据可用性交易 (Data Availability Transactions)

这些交易主要用于确保 Rollups 提交的 blob 数据在以太坊网络中是可用和可验证的。数据可用性是 Rollups 安全性的重要组成部分，因为链下计算需要基于链上的数据进行验证。

- **提交和验证**：Blob 数据提交到以太坊主链，并通过专门的验证机制确保数据的完整性和可用性。
- **数据存储**：Blob 数据不会永久存储在以太坊主链上，而是通过优化的存储方案（如短期存储或外部存储）来管理。

##### 1.4.2.2 状态更新交易 (State Update Transactions)

这种交易类型用于提交 Rollup 状态的更新信息。虽然具体的交易细节在 blob 中，但状态更新交易会包含指向这些 blob 的引用，以确保主链上的状态可以被正确更新和验证。

- **状态根更新**：每个状态更新交易会包含一个新的状态根，这个状态根是通过处理 blob 数据得到的。
- **验证机制**：状态根的更新和 blob 数据的处理需要通过加密验证机制来确保其准确性和安全性。

#### 1.4.3 Blob 交易的结构

Blob 交易的结构与传统以太坊交易有一定的区别，它引入了一些新的字段来处理和管理 blob 数据。以下是 blob 交易的一些关键字段：

- **nonce**：发送方账户的交易序号。
- **gasPrice**：每单位 gas 的价格。
- **gasLimit**：交易消耗的最大 gas 量。
- **to**：接收方地址，通常为空，因为 blob 交易主要用于数据提交。
- **value**：转移的以太币数量，通常为 0。
- **blobData**：包含实际的 blob 数据。
- **blobVersion**：blob 数据的版本号，用于处理不同版本的 blob 交易格式。
- **blobRoot**：blob 数据的 Merkle 根，用于数据验证。
- **v, r, s**：交易的签名。

#### 1.4.4 Blob 交易的执行流程

- **交易创建**：Rollup 协调器创建一个包含大量交易数据的 blob，并将其打包成 blob 交易。
- **提交到主链**：将 blob 交易提交到以太坊主链，包含 blob 数据和相关的验证信息。
- **验证和存储**：以太坊网络验证 blob 数据的完整性和可用性，并将其存储在优化的存储方案中。
- **状态更新**：Rollup 通过处理 blob 数据更新其状态，并将新的状态根提交到以太坊主链进行记录。