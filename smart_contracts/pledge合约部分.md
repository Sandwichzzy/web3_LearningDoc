# Pledge Solidity - 质押借贷协议

## 📋 项目概述

Pledge Solidity 是一个基于以太坊的智能合约质押借贷协议，采用多签名治理机制，支持多种代币的质押借贷业务。系统通过智能合约实现去中心化的资金匹配、利息计算和风险管理。

## 🏗️ 系统架构

### 核心合约组件

```
PledgePool (主合约)
├── MultiSignature (多签名治理)
├── BscPledgeOracle (价格预言机)
├── DebtToken (债务代币)
├── AddressPrivileges (权限管理)
└── SafeTransfer (安全转账库)
```

### 合约继承关系

- PledgePool: 继承 `ReentrancyGuard` 和 `multiSignatureClient`
- BscPledgeOracle: 继承 `MultiSignatureClient`
- DebtToken: 继承 `ERC20` 和 `AddressPrivileges`
- AddressPrivileges: 继承 `multiSignatureClient`

## 🔐 多签名治理系统

### MultiSignature 合约

- 功能: 实现多签名治理机制，要求多个签名者同意才能执行重要操作

- 数据结构：

  ```
    uint256 private constant defaultIndex=0; *// 默认申请索引*
    using whiteListAddress for address[];  *// 使用白名单地址库*
    address[] public signatureOwners; *// 签名者地址数组*
    uint256 public threshold; *// 签名阈值（需要多少个签名才能通过）*
  
    *struct* signatureInfo{
  	address applicant; *// 申请人地址*
  	address[] signatures; *// 签名者列表*
    }
    *// 消息哈希 => 签名信息数组的映射*
    mapping(bytes32=>signatureInfo[]) signatureMap;
  ```

- 特性:
  
  - 白名单地址管理：*提供地址数组的增删查功能，用于管理签名者白名单*
  
  - 可配置签名阈值
  
  - 支持申请创建、签名、撤销
  
  - 自动验证签名数量
  
    ```
    /
     @dev 获取有效的签名索引
     @param msghash 消息哈希
     @param lastIndex 上次检查的索引
     @return uint256 返回达到阈值的申请索引+1，如果没有则返回0
     @notice 这是多签名验证的核心函数，由客户端合约调用
    /
       function getValidSignature(bytes32 msghash,uint256 lastIndex) external view returns (uint256){
            signatureInfo[] storage signInfo=signatureMap[msghash];
            // 从lastIndex开始检查每个申请
            for (uint256 i=lastIndex;i<signInfo.length;i++){
            // 如果签名数量达到阈值，返回索引+1
            if(signInfo[i].signatures.length>=threshold){
            	return i+1;
        	}
    	}
    		return 0;// 没有达到阈值的申请
       }
    ```
  
    

### MultiSignatureClient 合约

- 功能: 为其他合约提供多签名验证功能的基础合约

- 使用: 任何需要多签名保护的合约都应该继承此合约 *并使用validCall修饰符*

- 检查多签名验证：

  ```
  /
      @dev 检查多签名验证
      @notice 核心验证逻辑：
      1. 生成消息哈希（调用者地址 + 当前合约地址）
      2. 向多签名合约查询该哈希是否有足够的签名
      3. 如果没有足够签名，交易将回滚
     */
     function checkMultiSignature() internal view {
     	   uint256 value;
         // 获取调用的以太币值（当前未使用，为未来扩展预留）
         assembly {
         value:=callvalue()
         }
         // 生成唯一的消息哈希：调用者地址 + 目标合约地址
         // 这确保了每个(调用者, 目标合约)组合都有唯一的哈希
         bytes32 msghash = keccak256(abi.encodePacked(msg.sender, address(this)));
         // 获取多签名合约地址
         address multiSign=getMultiSignatureAddress();
  
          // 查询多签名合约，检查是否有足够的签名
          // getValidSignature的实现逻辑（在multiSignature.sol中）：
          // 1. 遍历该msgHash对应的所有申请
          // 2. 检查每个申请的签名数量是否 >= threshold
          // 3. 如果找到达到阈值的申请，返回其索引+1（确保非零）
          // 4. 如果没有找到，返回0
          uint256 newIndex=IMultiSignature(multiSign).getValidSignature(msghash,defaultIndex);
          require(newIndex>defaultIndex,"multiSignatureClient : This tx is not aprroved");
      }
  ```

  

## 💰 质押借贷池系统

### PledgePool 合约

- 核心功能: 管理多个借贷池，处理用户存款、借款、结算等操作

- 查询函数

  | 函数                     | 返回类型   | 查询内容   | 使用场景       | 权限要求 |
  | ------------------------ | ---------- | ---------- | -------------- | -------- |
  | `poolLength`             | uint256    | 池数量     | 获取池总数     | 公开     |
  | `getPoolState`           | uint256    | 池状态     | 查询池当前状态 | 公开     |
  | `checkoutSettle`         | bool       | 是否可结算 | 检查结算条件   | 公开     |
  | `checkoutFinish`         | bool       | 是否可完成 | 检查完成条件   | 公开     |
  | `checkoutLiquidate`      | bool       | 是否可清算 | 检查清算条件   | 公开     |
  | `getUnderlyingPriceView` | uint256[2] | 代币价格   | 获取预言机价格 | 公开     |

- 管理员操作函数

  | 函数                   | 权限要求              | 操作内容     | 影响范围   | 使用场景         | 返回值 |
  | ---------------------- | --------------------- | ------------ | ---------- | ---------------- | ------ |
  | `createPool`           | validCall(多签名验证) | 创建新池     | 新增借贷池 | 部署新的借贷产品 | 无     |
  | `settle`               | validCall             | 结算池       | 池状态变更 | 完成资金匹配     | 无     |
  | `finish`               | validCall             | 完成池       | 池状态变更 | 计算利息并完成   | 无     |
  | `liquidate`            | validCall             | 清算池       | 池状态变更 | 处理风险池       | 无     |
  | `setFee`               | validCall             | 设置费用     | 全局费用   | 调整协议费用     | 无     |
  | `setSwapRouterAddress` | validCall             | 设置路由器   | 交换功能   | 更换 DEX 路由器  | 无     |
  | `setFeeAddress`        | validCall             | 设置费用地址 | 费用接收   | 更换费用接收地址 | 无     |
  | `setMinAmount`         | validCall             | 设置最小金额 | 存款限制   | 调整参与门槛     | 无     |
  | `setPause`             | validCall             | 暂停/恢复    | 全局状态   | 紧急情况控制     | 无     |

- 出借人(lender)操作函数:

  | 函数                      | 状态要求                     | 时间要求 | 操作类型 | 代币处理              | 使用场景         | 返回值 |
  | ------------------------- | ---------------------------- | -------- | -------- | --------------------- | ---------------- | ------ |
  | `depositLend`             | MATCH                        | 结算前   | 存入资金 | 转入池中              | 提供借贷资金     | 无     |
  | `refundLend`              | EXECUTION/FINISH/LIQUIDATION | 结算后   | 退还超额 | 转出超额部分          | 退还未匹配资金   | 无     |
  | `claimLend`               | EXECUTION/FINISH/LIQUIDATION | 结算后   | 领取凭证 | 铸造 SP 代币          | 获得资金凭证     | 无     |
  | `withdrawLend`            | FINISH/LIQUIDATION           | 到期后   | 提取本息 | 销毁 SP 代币+转出资金 | 取回本金和利息   | 无     |
  | `emergencyLendWithdrawal` | UNDONE                       | 无限制   | 紧急退出 | 转出全部存款          | 异常情况安全退出 | 无     |

- 借款人(Borrow)操作函数:
  
  | 函数                        | 状态要求                     | 时间要求 | 操作类型   | 代币处理                | 使用场景         | 返回值 |
  | --------------------------- | ---------------------------- | -------- | ---------- | ----------------------- | ---------------- | ------ |
  | `depositBorrow`             | MATCH                        | 结算前   | 质押抵押品 | 转入抵押品              | 提供借款担保     | 无     |
  | `refundBorrow`              | EXECUTION/FINISH/LIQUIDATION | 结算后   | 退还超额   | 转出超额抵押品          | 退还超额质押     | 无     |
  | `claimBorrow`               | EXECUTION/FINISH/LIQUIDATION | 结算后   | 领取资金   | 铸造 JP 代币+转出借款   | 获得借款资金     | 无     |
  | `withdrawBorrow`            | FINISH/LIQUIDATION           | 到期后   | 赎回抵押品 | 销毁 JP 代币+转出抵押品 | 取回质押的抵押品 | 无     |
  | `emergencyBorrowWithdrawal` | UNDONE                       | 无限制   | 紧急退出   | 转出全部抵押品          | 异常情况安全退出 | 无     |
  
- 池状态管理:
  
  | 阶段        | 状态     | 可执行操作                                         | 用户行为         | 状态说明                                 |
  | ----------- | -------- | -------------------------------------------------- | ---------------- | ---------------------------------------- |
  | MATCH       | 匹配阶段 | depositLend, depositBorrow                         | 存款和质押       | 用户可以向池中存入资金或质押抵押品       |
  | EXECUTION   | 执行阶段 | claimLend, claimBorrow, refundLend, refundBorrow   | 领取凭证和资金   | 资金匹配完成，用户可以领取凭证或申请退款 |
  | FINISH      | 完成阶段 | withdrawLend, withdrawBorrow                       | 提取本息和抵押品 | 借贷周期结束，用户可以取回资金和抵押品   |
  | LIQUIDATION | 清算阶段 | withdrawLend, withdrawBorrow                       | 清算后提取       | 风险触发清算，用户按清算价格提取         |
  | UNDONE      | 异常阶段 | emergencyLendWithdrawal, emergencyBorrowWithdrawal | 紧急退出         | 池无法正常运作，用户紧急退出             |

### 用户操作流程

#### 出借人流程

```
存款 → 等待结算 → 领取凭证 → 等待到期 → 提取本息
```

#### 借款人流程

```
质押抵押品 → 等待结算 → 领取资金 → 等待到期 → 赎回抵押品
```

### 主要函数

| 用户类型 | 函数             | 功能描述         |
| -------- | ---------------- | ---------------- |
| 出借人   | `depositLend`    | 存入借贷资金     |
| 出借人   | `claimLend`      | 领取 SP 代币凭证 |
| 出借人   | `withdrawLend`   | 提取本息         |
| 借款人   | `depositBorrow`  | 质押抵押品       |
| 借款人   | `claimBorrow`    | 领取借款资金     |
| 借款人   | `withdrawBorrow` | 赎回抵押品       |

### 重要管理员操作函数

1. 触发结算池

```
function settle(uint256 _pid) public validCall{
	PoolBaseInfo storage pool = poolBaseInfos[_pid];
    PoolDataInfo storage data= poolDataInfos[_pid];
 	require(checkoutSettle(_pid),"settle: time is less than settle time");
	require(pool.state==PoolState.*MATCH*,"settle: pool state must be MATCH");

	if(pool.lendSupply>*0* && pool.borrowSupply>*0*){
​      *//获取资产对价格*
​      uint256[*2*] memory prices=getUnderlyingPriceView(_pid);
​      *//计算质押保证金总价值 =价格比率（抵押品价格/出借代币价格）* 抵押品数量
​      uint256 valueRatio=safeDiv(safeMul(prices[1],calDecimals),prices[0]);
​      uint256 totalValue=safeDiv(safeMul(pool.borrowSupply,valueRatio),calDecimals);

​      //计算实际价值 = 总价值 ÷抵押率
​      // totalValue = 50,000 USDC*
​      // 抵押率 = 150%（1.5倍）*
​      // actualValue = 50,000 × 1e8 ÷ 150,000,000 = 33,333.33 USDC*
​      uint256 actualValue=safeDiv(safeMul(totalValue,baseDecimal),pool.martgageRate);

​      if(pool.lendSupply>actualValue){
​        //借出资金总量大于实际可借价值
​        data.settleAmountLend=actualValue;
​        data.settleAmountBorrow=pool.borrowSupply;
​      }else{
​         //借出资金总量小于实际可借价值
​        data.settleAmountLend=pool.lendSupply;
​        *//结算时的实际借款金额 
		 //实际借款金额计算=(借出资金 × 抵押率) ÷ 价格比率
​        uint256 priceRatio = safeDiv(safeMul(prices[*1*], baseDecimal), prices[*0*]);
​        data.settleAmountBorrow = safeDiv(safeMul(pool.lendSupply, pool.martgageRate), priceRatio);
​      }

​      *// 更新池子状态为执行*
​      pool.state=PoolState.*EXECUTION*;
​       *// 触发事件*
​      emit StateChange(_pid,uint256(PoolState.*MATCH*), uint256(PoolState.*EXECUTION*));

​    } else {
​      *// 极端情况，借款或借出任一为0*
​      pool.state=PoolState.*UNDONE*;

​      data.settleAmountLend=pool.lendSupply;

​      data.settleAmountBorrow=pool.borrowSupply;
​      *// 触发事件*
​      emit StateChange(_pid,uint256(PoolState.*MATCH*), uint256(PoolState.*UNDONE*));
​    }
  }
```

2. 执行finish池

   ```
   */*
      * *@dev* *完成一个借贷池的操作，包括计算利息、执行交换操作、赎回费用和更新池子状态等步骤。*
      * *@param* *_pid* *是池子的索引*
      /*
     function finish(uint256 _pid) public validCall{
   ​    *// 获取基础池子信息和数据信息*
   ​    PoolBaseInfo storage pool = poolBaseInfos[_pid];
   ​    PoolDataInfo storage data = poolDataInfos[_pid];
   ​    require(checkoutFinish(_pid),"finish: less than end time");
   ​    require(pool.state==PoolState.*EXECUTION*,"finish: pool state must be execution");
   
   ​    (address token0,address token1)=(pool.borrowToken,pool.lendToken);
   ​    *// timeRatio = (结束时间 - 结算时间) × 1e8 ÷ 365天
   ​    uint256 timeRatio = safeDiv(safeMul(safeSub(pool.endTime, pool.settleTime), baseDecimal), baseYear);
   ​    *// 计算利息(1e18) = 基础利息（结算贷款金额(1e18)× 利率(1e8) ）× 时间比率(1e8)*
   ​    uint256 interest = safeDiv(safeMul(timeRatio, safeMul(pool.interestRate, data.settleAmountLend)), 1e16);
   ​    uint256 lendAmount = safeAdd(data.settleAmountLend, interest); *// 计算贷款金额 = 结算贷款金额 + 利息*
   
   ​    *// 计算需要变现的抵押品价值 = 贷款金额 * (1 + lendFee费用)
   ​    uint256 sellAmount = safeDiv(safeMul(lendAmount, safeAdd(lendFee, baseDecimal)), baseDecimal);
   ​     *// 执行代币交换操作 amountSell：实际卖出的抵押品数量 amountIn：实际获得的出借代币数量*
   ​    (uint256 amountSell,uint256 amountIn) = _sellExactAmount(swapRouter,token0,token1,sellAmount);
   
   ​    require(amountIn >= lendAmount,"finish: Slippage is too high");
   
   ​    if(amountIn>lendAmount){
   ​      uint256 feeAmount = safeSub(amountIn, lendAmount);
   ​      *//如果变现收益超过还款需求：超额部分作为协议费用*
   ​      _redeem(payable(feeAddress),pool.lendToken, feeAmount);
   ​      data.finishAmountLend = safeSub(amountIn, feeAmount); *//更新完成时的出借金额*
   ​    }else{
   ​       data.finishAmountLend = amountIn;
   ​    }
   
   ​     *// 计算剩余的抵押品数量*
   
   ​     uint256 remainNowAmount = safeSub(data.settleAmountBorrow, amountSell);
   ​     uint256 remainBorrowAmount=redeemFees(borrowFee,pool.borrowToken,remainNowAmount);*//返回扣除费用后的剩余金额 
          //剩余金额 = 原金额 - (原金额 × borrowFee费率)
   ​     data.finishAmountBorrow=remainBorrowAmount;
   ​     pool.state=PoolState.*FINISH*;
   
   ​     emit StateChange(_pid,uint256(PoolState.*EXECUTION*), uint256(PoolState.*FINISH*));
   
     }
   ```

   

3. 触发清算

   ```
     *function* liquidate(uint256 _pid) public validCall{
   ​    PoolDataInfo storage data = poolDataInfos[_pid]; 
   ​    PoolBaseInfo storage pool = poolBaseInfos[_pid]; 
   ​    require(block.timestamp > pool.settleTime, "liquidate: time is less than settle time"); *// 需要当前时间大于结算时间
   ​    require(pool.state == PoolState.*EXECUTION*,"liquidate: pool state must be execution"); *// 需要池子的状态是执行状态*
   
   ​    (address token0,address token1)=(pool.borrowToken,pool.lendToken);
   ​     *// 时间比率(1e8) = ((结束时间 - 结算时间) \* 基础小数)/365天*
   ​    uint256 timeRatio = safeDiv(safeMul(safeSub(pool.endTime, pool.settleTime), baseDecimal), baseYear);
   ​    *// 计算利息(1e18) = 基础利息（结算贷款金额(1e18)× 利率(1e8) ）× 时间比率(1e8)*
   ​    uint256 interest = safeDiv(safeMul(timeRatio, safeMul(pool.interestRate, data.settleAmountLend)), 1e16);
   ​    *// 计算贷款金额 = 结算贷款金额 + 利息*
   
   ​    uint256 lendAmount = safeAdd(data.settleAmountLend, interest);
   ​    *// 添加贷款费用*
   ​    uint256 sellAmount = safeDiv(safeMul(lendAmount, safeAdd(lendFee, baseDecimal)), baseDecimal);
   ​    (uint256 amountSell,uint256 amountIn) = _sellExactAmount(swapRouter,token0,token1,sellAmount); *// 卖出准确的金额*
   
   ​    *// 可能会有滑点，amountIn - lendAmount < 0;*
   ​    if (amountIn > lendAmount) {
   ​      uint256 feeAmount = safeSub(amountIn, lendAmount); *// 费用金额*
   ​      *// 贷款费用*
   ​      _redeem(payable(feeAddress),pool.lendToken, feeAmount);
   ​      data.liquidationAmountLend = safeSub(amountIn, feeAmount);
   ​    }else {
   ​      data.liquidationAmountLend = amountIn;
   ​    }
   
   ​    *// liquidationAmountBorrow  借款费用*
   ​    uint256 remainNowAmount = safeSub(data.settleAmountBorrow, amountSell); *// 剩余的现在的金额*
   ​    uint256 remainBorrowAmount = redeemFees(borrowFee,pool.borrowToken,remainNowAmount); *// 剩余的借款金额*
   ​    data.liquidationAmountBorrow = remainBorrowAmount;
   
   ​    *// 更新池子状态*
   ​    pool.state = PoolState.*LIQUIDATION*;
   ​     *// 事件*
   ​    emit StateChange(_pid,uint256(PoolState.*EXECUTION*), uint256(PoolState.*LIQUIDATION*));
   
     }
   ```

   

## 🔮 价格预言机系统

### BscPledgeOracle 合约

- 功能:1. *混合价格预言机系统，支持Chainlink聚合器和手动价格设置*  2.*为Pledge系统提供可靠的价格数据，支持多种资产的价格查询*
- 特性:
  - 支持 Chainlink 聚合器价格
  - 支持手动设置价格（备用）
  - 统一 18 位小数精度
  - 多签名控制价格设置

### 设置价格

手动设置：

```
 /
  @notice 批量设置资产价格
  @dev 手动设置多个资产的价格（备用价格源）
  @param assets 资产ID数组
  @param prices 对应的价格数组（18位小数精度）
     */
    function setPrices(uint256[] memory assets, uint256[] memory prices) external validCall{
		require(assets.length==prices.length,"input arrays length are not equal");
        uint256 len=assets.length;
        for (uint256 i=0;i<len;i++){
            pricesMap[assets[i]]=prices[i];
         }
   	}
```

使用 Chainlink 聚合器价格获取价格

```
 /
 @notice 获取单个资产价格（核心函数）
 @dev 实现双重价格源的价格获取逻辑
 @param underlying 资产标识符（地址转uint256或自定义ID）
 @return uint256 资产价格（18位小数精度）
     */
    function getUnderlyingPrice(uint256 underlying) public view returns (uint256){
		//获取配置的chainlink聚合器
		AggregatorV3Interface assetsPrice=assetsMap[underlying];
		//优先使用chainlink聚合器价格
		if (address(assetsPrice)!=address(0)){
    	// 调用Chainlink聚合器获取最新价格数据
    	(,int256 price,,,) = assetsPrice.latestRoundData();
    	// 根据资产精度进行转换
    	uint256 tokenDecimals=decimalsMap[underlying];
    	if (tokenDecimals<18){
		// 例如：USDC(6位) → 18位
        // price: $1.000000 (8位小数) → 需要补足到18位
        return uint256(price)/decimals*(10(18-tokenDecimals));
            }else if (tokenDecimals>18){
        // 理论情况：如果代币精度超过18位 → 需要降低精度
        return uint256(price)/decimals/(10(tokenDecimals-18));
            }else{
        // 如果精度正好是18位 → 直接除以精度
        return uint256(price)/decimals;
            }
        }else{
            // 如果没有聚合器 → 返回手动设置的价格
            return pricesMap[underlying];
        }
    }
```

### 询优先级及精度

1. Chainlink 聚合器价格（优先）

2. 手动设置价格（备用）

3. 无价格时返回 0

4. *Chainlink价格通常是8位小数（如BTC/USD $50000.12345678）*

   本系统统一使用18位小数（以太坊标准）自动进行精度转换以确保计算准确性

## 代币系统

### DebtToken 合约

- 功能: 代表用户权益的 ERC20 代币
- 类型:
  - SP 代币: 出借人凭证，代表存款权益
  - JP 代币: 借款人凭证，代表抵押品权益

### AddressPrivileges 合约

- 功能: 管理铸币者权限：*添加铸币者地址*，*删除铸币者地址*，*检查地址是否为铸币者* ，*根据索引获取铸币者地址*
- 特性: 基于 OpenZeppelin 的 EnumerableSet 实现高效权限管理

## 🛡️ 安全特性

### 重入攻击防护

- 使用 `ReentrancyGuard` 修饰符
- 所有涉及资金转移的函数都有重入保护

### 多签名验证

- 重要操作需要多签名验证
- 可配置签名阈值
- 防止单点故障

### 状态检查

- 严格的状态转换控制
- 时间限制验证
- 金额限制检查



## 📊 合约状态管理

### 池生命周期

```
创建池 → MATCH → 用户操作 → 结算 → EXECUTION → 完成/清算 → FINISH/LIQUIDATION
```

### 状态转换条件

- MATCH → EXECUTION: 达到结算时间，资金匹配完成
- EXECUTION → FINISH: 借贷期限到期，计算利息
- EXECUTION → LIQUIDATION: 触发风险阈值，启动清算
- MATCH → UNDONE: 存款或借款为 0，池无法正常运作

### 状态管理流程图

![poolStateflow](imgs/poolStateflow.png)
