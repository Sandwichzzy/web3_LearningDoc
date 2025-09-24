# MetaStake 合约深度解析

## 1. 项目概述

MetaStake  是一个基于以太坊的质押挖矿智能合约系统，支持 UUPS 可升级代理模式。用户可以通过质押代币获得 MetaNode 奖励，系统采用权重分配机制确保公平的奖励分配。

- 支持 **多池质押**（ETH + ERC20 代币）
- 采用 **可升级合约架构**（UUPS + OpenZeppelin）
- 动态调整 **区块奖励权重**
- 提现 **延迟解锁机制**（防挤兑攻击）

------



## **2. 合约架构**

### **2.1 技术栈**

- **Solidity 0.8.20**（安全数学运算）
- **OpenZeppelin 库**：
  - `UUPSUpgradeable`（可升级代理模式）
  - `AccessControl`（权限管理）
  - `SafeERC20`（安全转账）
  - `Pausable`（紧急暂停功能）

### **2.2 关键角色**

| 角色                 | 权限                         |
| :------------------- | :--------------------------- |
| `DEFAULT_ADMIN_ROLE` | 超级管理员（默认合约部署者） |
| `UPGRADE_ROLE`       | 合约升级权限                 |
| `ADMIN_ROLE`         | 日常管理（如修改参数）       |

------

### **2.3 核心合约**

\- ***\*MetaNodeStake.sol\****: 主要的质押合约，实现质押、解质押、领取奖励等核心功能

\- ***\*StakeTokenERC20.sol\****: 质押代币合约，用于测试和演示

### **2.4 技术特性**

- **UUPS 代理模式**: 支持合约升级，无需重新部署代理

- **权重分配系统**: 基于池权重的奖励分配机制

- **解质押锁定机制**: 防止短期投机行为

- **多池支持**: 支持多个质押池，每个池可以有不同的权重和参数

- **权限管理**: 基于角色的访问控制

  

## **3. 核心机制解析**

### **3.1 数据结构**

- 质押池：

```
 *struct* Pool {
	  address stTokenAddress;   *// Address of staking token*
	  uint256 poolWeight;  		*// Weight of pool 质押池的权重，影响奖励分配。*
	  uint256 lastRewardBlock;	*// Last block number that MetaNodes distribution occurs for pool*
      uint256 accMetaNodePerST; *// Accumulated MetaNodes per staking token of pool 每个质押代币累积的 MetaNode 数量*
	  uint256 stTokenAmount;    *// Staking token amount 池中的总质押代币量*
      uint256 minDepositAmount; *// Min staking amount 最小质押量*
	  uint256 unstakeLockedBlocks; *// Withdraw locked blocks : 解除质押的锁定区块数*
  } 
```

- 用户：

```
  *struct* User {

​    uint256 stAmount;  *// Staking token amount that user provided 用户质押的代币数量。*

​    uint256 finishedMetaNode;  *// Finished distributed MetaNodes to user 已分配的 MetaNode 数量*

​    uint256 pendingMetaNode;   *// Pending to claim MetaNodes 待领取的MetaNode奖励*

​    UnstakeRequest[] requests; *// Withdraw request list 解质押请求列表，每个请求包含解质押数量和解锁区块。*

  }
```

- 解质押请求：

```
  *struct* UnstakeRequest {
​    uint256 amount;     *// Request withdraw amount*
​    uint256 unlockBlocks;     *// The blocks when the request withdraw amount can be released*
  }
```

其他状态变量：

```
  uint256 public startBlock;   *// First block that MetaNodeStake will start from*
  uint256 public endBlock;   *// First block that MetaNodeStake will end from*
  uint256 public MetaNodePerBlock;   *// MetaNode token reward per block*
  bool public withdrawPaused;   *// Pause the withdraw function*
  bool public claimPaused;    *// Pause the claim function*
  *IERC20* public MetaNode;   *// MetaNode token*
  uint256 public totalPoolWeight;   *// Total pool weight / Sum of all pool weights*
  
  Pool[] public pool;
  mapping (uint256 => mapping (address => User)) public user;   *// pool id => user address => user info*
```



### **3.2 质押挖矿逻辑**

- **多池权重分配**：

  ```
  MetaNodeForPool = (blocksPassed × MetaNodePerBlock) × (poolWeight / totalPoolWeight)
  ```

- **每单位代币累积奖励更新**：

  累积奖励机制的核心原理：accMetaNodePerST 是一个累积值，它记录的是从质押池创建开始到当前时刻，每单位质押代币累积获得的总奖励。

  **为什么要累积？**

  历史奖励保留：

  - accMetaNodePerST 保存了从池子创建到上次更新时的所有历史奖励

  - 新的奖励需要叠加到历史奖励上，而不是替换

  ```
  accMetaNodePerST += (MetaNodeForPool * 1e18) / pool.stTokenAmount //更新每单位stToken的MetaNode奖励
  ```

  **奖励计算**（基于权重的奖励分配机制）：

  ```
  pendingMetaNode = (user.stAmount × pool.accMetaNodePerST) / 1e18 - user.finishedMetaNode
  ```

  - 用户的待领取奖励 = 用户质押量 × 每单位累积奖励 - 已分配奖励 

  - 如果 accMetaNodePerST 不累积，用户就会丢失之前的奖励

  - **完整的奖励计算**：

    ```
    pendingMetaNode = (user_.stAmount ×  accMetaNodePerST)/1e18 - user_.finishedMetaNode + user_.pendingMetaNode;
    ```

    ##### 核心1：finishedMetaNode 的作用是记录用户已经"计算过"的奖励基准：

    ##### 更新公式：**finishedMetaNode =user_.stAmount * pool_.accMetaNodePerST / (1 ether)**

    1. 在 claim 函数中有条件更新（**领取奖励时**）

       更新时机：只有当用户的所有待领取奖励都被完全分发时

       更新逻辑：

       - 如果 pendingMetaNode == 0（所有奖励都已领取完毕）

       - 才重新设置 finishedMetaNode 为当前的累积奖励基准

    2. 在 unstake 函数中更新（**解质押时**）：

       更新时机：用户解质押代币后

    ​       更新逻辑：

    - 先计算并累积当前奖励到 pendingMetaNode

    - 减少用户的质押量 user_.stAmount

    - 根据用户的剩余质押量重新计算 finishedMetaNode

    3. 在 _deposit 函数中更新（**质押时**）：

       更新时机：用户质押代币后

    ​       更新逻辑：

    - 先计算并累积用户的待领取奖励到 pendingMetaNode

    - 然后根据用户的新质押量重新计算 finishedMetaNode

    - 这样确保了奖励计算的基准点被正确设置

    

    ##### **核心2：pendingMetaNode** 是一个缓存字段，用来暂存用户已计算但未领取的奖励,为了处理部分领取和合约余额不足的情况:

    1. **操作时奖励累积（unstake/deposit时）:**

       场景：用户在 unstake 或 deposit 时，会先计算当前的奖励，但不立即发放，而是累积到 pendingMetaNode 中。

       ```
       uint256 pendingMetaNode_ = user_.stAmount * pool_.accMetaNodePerST / 1e18 - user_.finishedMetaNode;
       
       if(pendingMetaNode_ > 0) {
       
         user_.pendingMetaNode = user_.pendingMetaNode + pendingMetaNode_; *//累积待领取奖励*
       
       }
       ```

       

    2. **部分领取claim场景（合约余额不足）：**

       ```
       uint256 MetaNodeBal = MetaNode.balanceOf(address(this));
       
       actualTransfer = pendingMetaNode_ > MetaNodeBal ? MetaNodeBal : pendingMetaNode_;
       
       *// 只更新实际转移的部分*
       
       user_.pendingMetaNode = pendingMetaNode_ - actualTransfer;
       ```

    - 举例：

      - 用户应得奖励：100 MetaNode

      - 合约余额：只有 60 MetaNode

      - 实际转给用户：60 MetaNode

      - 剩余未领取：40 MetaNode → 存储在 user_.pendingMetaNode

        



### **3.3 延迟提现设计**

- **防挤兑攻击**：
  - 用户发起 `unstake()` 后，需等待 `unstakeLockedBlocks` 才能 `withdraw()`
  - 请求存储在 `UnstakeRequest[]` 数组中，按区块高度分批解锁

### **3.4 ETH 与 ERC20 双模式**

- **ETH 池**（`pid=0`）：
  - `stTokenAddress = address(0)`
  - 通过 `depositETH()` + `msg.value` 存入
- **ERC20 池**：
  - 需先 `approve()` 授权合约转移代币

------



## **4. 安全策略**

### **4.1 防御措施**

| 风险     | 解决方案                                            |
| :------- | :-------------------------------------------------- |
| 重入攻击 | 使用 `SafeERC20` + 先更新状态再转账                 |
| 算术溢出 | Solidity 0.8 默认检查 + `Math` 库的 `tryMul/tryDiv` |
| 恶意升级 | `UUPSUpgradeable` 仅允许 `UPGRADE_ROLE` 操作        |
| 紧急情况 | `Pausable` 暂停关键功能（提现/领取奖励）            |

### **4.2 边界处理**

- **存款下限**：`amount ≥ minDepositAmount`

- **奖励分配**：`block.number ∈ [startBlock, endBlock]`

- **代币不足时**：`_safeMetaNodeTransfer()` 自动调整转账金额,  

  如果pendingMetaNode > 合约里的metaNode余额：那么 调整为转移合约剩余的metaNode, user_.pendingMetaNode只更新实际转移的部分

  ```
        actualTransfer = pendingMetaNode_ > MetaNodeBal ? MetaNodeBal : pendingMetaNode_;
  ​      if (actualTransfer > 0 ){
  ​        _safeMetaNodeTransfer(msg.sender, actualTransfer);
  ​      }
  		// 只更新实际转移的部分
          user_.pendingMetaNode = pendingMetaNode_ - actualTransfer;
         }
          // 只在完全分发时更新finishedMetaNode
          if(user_.pendingMetaNode == 0){
              user_.finishedMetaNode = user_.stAmount * pool_.accMetaNodePerST / (1 ether);
          }
  ```

  

------



## **5. 操作**

### **5.1 用户流程**

1. **存入**：

   ```
   // ETH 池 池ID：1
   contract.depositETH{value: 1 ether}();
   
   // ERC20 池
   token.approve(contract, _amount);
   contract.deposit(pid, _amount);
   ```

2. **领取奖励**：

   ```
   contract.claim(pid);
   ```

3. **提现**：

   ```
   contract.unstake(pid, 500);  // 发起请求
   contract.withdraw(pid);       // 实际提币（需等待解锁）
   ```

### **5.2 管理员操作**

- 添加池子：

  ```
  contract.addPool(_stTokenAddress , _poolWeight , _minDepositAmount，_unstakeLockedBlocks，_withUpdate) 
  //添加池子，_withUpdate为true是立刻更新所有池子的奖励
  ```

  调整参数：

- ```
  contract.setMetaNodePerBlock(100);  // 修改区块奖励，没经过一个区块奖励多少个metaNode
  contract.setPoolWeight(1, 200);     // 调整池权重
  contract.updatePool(_pid, _minDepositAmount, _unstakeLockedBlocks) //调节池子 最小质押量和解质押所需区块数
  ```

- 紧急暂停：

  ```
  contract.pauseWithdraw();  // 暂停提现
  contract.pauseClaim();     // 暂停领取奖励
  ```

------



## **6. 总结**

### **6.1 适用场景**

- 多代币质押挖矿平台
- 需要灵活升级的 DeFi 协议
- 流动性挖矿（dex协议sushiswap开启，为了吸引流动性提供商，增加池子深度）

### **6.2 改进建议**

- 添加 **时间锁**（TimelockController）管理关键参数变更
- 实现 **动态奖励衰减**（如每 10 万区块减半）
- 前端集成 **预估收益计算器**