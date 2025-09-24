### Uniswap 基本介绍

Uniswap 是以太坊上最大的去中心化交易所（DEX），我们在上一讲中提到了，Uniswap 这样的去中心化交易所采用的不是订单薄交易的方式，而是由 LP 提供流动性来交易，这个流动性池子中的代币如何定价则成为了去中心化交易所的关键。Uniswap 在其流动性池上构建了一种特定的自动做市商（AMM）机制。称为恒定乘积做市商（Constant Product Market Makers，CPMM）。顾名思义，其核心是一个非常简单的乘积公式：

x∗y=k*x*∗*y*=*k*

流动性池是一个持有两种不同 token 的合约， x 和 y 分别代表 token0 的数目和 token1 的数目， k 是它们的乘积，当 swap 发生时，token0 和 token1 的数量都会发生变化，但二者乘积保持不变，仍然为 k 。

另外，我们一般说的 token0 的价格是指在流动性池中相对于 token1 的价格，价格与数量互为倒数，因此公式为：

P=y/x*P*=*y*/*x*

就比如说我作为 LP 在池子中放了 1 个 ETH(token0) 和 3000 个 USDT(token1)，那么 k 就是 1*3000=3000，ETH 价格就是 3000/1 = 3000U。那你作为交易方就可以把大概 30 USDT 放进去，拿出来 0.01 个 ETH。然后池子里面就变成了 3030 个 USDT 和 0.99 个 ETH，价格变 3030/0.99≈3030U。ETH 涨价了，这样是不是就解决了定价的问题，有人要换 ETH，ETH 变得稀缺，所以涨价了，下次要换 ETH 就需要更多的 USDT，只要保证池子中的 ETH * USDT 等于一个常量，这样自然就会此消彼长，当 ETH 变少时，你要通过 USDT 换取 ETH 时候就需要消耗更多 USDT，反之亦然。

当然上面的例子没有考虑滑点、手续费、取整等细节，实际合约实现时也有很多细节需要考虑。这里只是为了让大家理解基础逻辑，具体的细节会在后面展开。

Uniswap 到目前已经迭代了好几个版本，下面是各个版本的发展历程：

2018 年 11 月 Uniswap V1 发布，创新性地采用了上述 CPMM，支持 ERC-20 和 ETH 的兑换，为后续版本的 Uniswap 奠定了基础，并成为其他 AMM 协议的启示；

2020 年 5 月 Uniswap V2 发布， 在 V1 的基础上引入了 ERC-20 之间的兑换，以及时间加权平均价格（TWAP）预言机，增加交易对的灵活性，巩固了 Uniswap 在 DEX 的领先地位；

2021 年 5 月 Uniswap V3 发布，引入了集中流动性（Concentrated Liquidity），允许 LP 在交易对中定义特定的价格范围，以实现更精确的价格控制，提升了 LP 的资产利用率；

2023 年 6 月 Uniswap V4 公开了白皮书的草稿版本，引入了 Hook、Singleton、Flash Accounting 和原生 ETH 等多个优化，其中 Hook 是最重要的创新机制，给开发者提供了高度自定义性。

2025年1月Uniswap V4 发布，V4 版本在算法上并没有改变，依然还是采用集中流动性，但通过 Hooks 实现了可定制的池，单例合约和闪电记账大幅度降低了 gas 成本，对原生 ETH 的支持也同样减少了 gas，还有对动态费用的支持、ERC1155 的支持等，都大大提高了 Uniswap 的灵活性、可扩展性。

### Uniswap V3 代码解析

如上所说，Uniswap 核心就是要基于 CPMM 来实现一个自动化做市商，除了用户调用的交易合约外，还需要有提供给 LP 管理流动性池子的合约，以及对流动性的管理。这些功能在不同的合约中实现，在 Uniswap 的架构中，Uniswap V3 的合约大概被分为两类，分别存储在不同的仓库中：

[UniswapV3白皮书](https://github.com/adshao/publications/blob/master/uniswap/dive-into-uniswap-v3-whitepaper/README_zh.md)

![uniswapV3](img/uniswapV3_stru)*uniswapV3*

- Uniswap v3-periphery

  面向用户的接口代码，如头寸管理、swap 路由等功能，Uniswap 的前端界面与 periphery 合约交互，主要包含三个合约：

  - NonfungiblePositionManager.sol：对应头寸管理功能，包含交易池（又称为流动性池或池子，后文统一用交易池表示）创建以及流动性的添加删除；
  - NonfungibleTokenPositionDescriptor.sol：对头寸的描述信息；
  - SwapRouter.sol：对应 swap 路由的功能，包含单交易池 swap 和多交易池 swap。

- Uniswap v3-core

  Uniswap v3 的核心代码，实现了协议定义的所有功能，外部合约可直接与 core 合约交互，主要包含三个合约；

  - UniswapV3Factory.sol：工厂合约，用来创建交易池，设置 Owner 和手续费等级；
  - UniswapV3PoolDeployer.sol：工厂合约的基类，封装了部署交易池合约的功能；
  - UniswapV3Pool.sol：交易池合约，持有实际的 Token，实现价格和流动性的管理，以及在当前交易池中 swap 的功能。

我们主要解析核心流程，包括以下：

1. 部署交易池；
2. 创建/添加/减少流动性；
3. swap。

其中 1 和 2 都是合约提供给 LP 操作的功能，通过部署交易池和管理流动性来提供和管理流动性。而 3 则是提供给普通用户使用 Uniswap 的核心功能（甚至可以说是唯一的功能）swap，也就是交易。接下来我们将依次讲解 Uniswap 中的相关代码。

#### **部署交易池**

在 Uniswap V3 中，通过合约 [UniswapV3Pool](https://github.com/Uniswap/v3-core/blob/main/contracts/UniswapV3Pool.sol#L30) 来定义一个交易池子，Uniswap 最核心的交易功能在最底层就是调用了该合约的 swap 方法。

而不同的交易对，以及不同的费率和价格区间（后面会具体讲到 tickSpacing）都会部署不同的 UniswapV3Pool 合约实例来负责交易。部署交易池则是针对某一对 token 以及指定费率的和价格区间来部署一个对应的交易池，当部署完成后再次出现同样条件下的交易池则不再需要重复部署了。

部署交易池调用的是 NonfungiblePositionManager 合约的 [createAndInitializePoolIfNecessary](https://github.com/Uniswap/v3-periphery/blob/main/contracts/base/PoolInitializer.sol#L13)，参数为：

- token0：token0 的地址，需要小于 token1 的地址且不为零地址；

- token1：token1 的地址；

- fee：以 1,000,000 为基底的手续费费率，Uniswap v3 前端界面支持四种手续费费率（0.01%，0.05%、0.30%、1.00%），对于一般的交易对推荐 0.30%，fee 取值即 3000；

- sqrtPriceX96：当前交易对价格的算术平方根左移 96 位的值，目的是为了方便合约中的计算。例如，如果 ETH 的价格是 2500 USDC，那么 sqrtPriceX96

  将是 sqrt(2500) << 96 的近似值。

代码为：

```
/// @inheritdoc IPoolInitializer
function createAndInitializePoolIfNecessary(
    address token0,
    address token1,
    uint24 fee,
    uint160 sqrtPriceX96
) external payable override returns (address pool) {
    require(token0 < token1);
    pool = IUniswapV3Factory(factory).getPool(token0, token1, fee);

    if (pool == address(0)) {
        pool = IUniswapV3Factory(factory).createPool(token0, token1, fee);
        IUniswapV3Pool(pool).initialize(sqrtPriceX96);
    } else {
        (uint160 sqrtPriceX96Existing, , , , , , ) = IUniswapV3Pool(pool).slot0();
        if (sqrtPriceX96Existing == 0) {
            IUniswapV3Pool(pool).initialize(sqrtPriceX96);
        }
    }
}
```

逻辑非常直观，首先将 token0，token1 和 fee 作为三元组取出交易池的地址 pool，如果取出的是零地址则创建交易池然后初始化，否则继续判断是否初始化过（当前价格），未初始化过则初始化。

我们分别看创建交易池的方法和初始化交易池的方法。

#### **创建交易池**

创建交易池调用的是 UniswapV3Factory 合约的 [createPool](https://github.com/Uniswap/v3-core/blob/main/contracts/UniswapV3Factory.sol#L35)，参数为：

- token0：token0 的地址
- token1 地址：token1 的地址；
- fee：手续费费率。

代码为：

```
/// @inheritdoc IUniswapV3Factory
function createPool(
    address token0,
    address token1,
    uint24 fee
) external override noDelegateCall returns (address pool) {
    require(token0 != token1);
    (address token0, address token1) = token0 < token1 ? (token0, token1) : (token1, token0);
    require(token0 != address(0));
    int24 tickSpacing = feeAmountTickSpacing[fee];
    require(tickSpacing != 0);
    require(getPool[token0][token1][fee] == address(0));
    pool = deploy(address(this), token0, token1, fee, tickSpacing);
    getPool[token0][token1][fee] = pool;
    // populate mapping in the reverse direction, deliberate choice to avoid the cost of comparing addresses
    getPool[token1][token0][fee] = pool;
    emit PoolCreated(token0, token1, fee, tickSpacing, pool);
}
```

通过 fee 获取对应的 tickSpacing，要解释 tickSpacing 必须先解释 tick。

```
int24 tickSpacing = feeAmountTickSpacing[fee];
```

tick 是 V3 中价格的表示，如下图所示：

图片加载失败: img/tick.webp*tick*

在 V3，整个价格区间由离散的、均匀分布的 ticks 进行标定。因为在 Uniswap V3 中 LP 添加流动性时都会提供一个价格的范围（为了 LP 可以更好的管理头寸），要让不同价格范围的流动性可以更好的管理和利用，需要 ticks 来将价格划分为一个一个的区间，每个 tick 有一个 index 和对应的价格：

P(i)=1.0001i*P*(*i*)=1.0001*i*

P(i) 即为 tick 在 i 位置的价格. 后一个价格点的价格是前一个价格点价格基础上浮动万分之一。我们可以得到关于 i 的公式：

i=log1.0001⁡(P(i))*i*=*l**o**g*1.0001⁡(*P*(*i*))

V3 规定只有被 tickSpacing 整除的 tick 才允许被初始化，tickSpacing 越大，每个 tick 流动性越多，tick 之间滑点越大，但会节省跨 tick 操作的 gas。

随后确认对应的交易池合约尚未被创建，调用 [deploy](https://github.com/Uniswap/v3-core/blob/main/contracts/UniswapV3PoolDeployer.sol#L27)，参数为工厂合约地址，token0地址，token1地址，fee，以及上面提到的tickSpacing。

```
pool = deploy(address(this), token0, token1, fee, tickSpacing);
```

#### 初始化交易池

初始化交易池调用的是 UniswapV3Factory 合约的 [initialize](https://github.com/Uniswap/v3-core/blob/main/contracts/UniswapV3Pool.sol#L271)，参数为当前价格 sqrtPriceX96，含义上面已经介绍过了。



代码如下：

```
/// @inheritdoc IUniswapV3PoolActions
/// @dev not locked because it initializes unlocked
function initialize(uint160 sqrtPriceX96) external override {
    require(slot0.sqrtPriceX96 == 0, 'AI');

    int24 tick = TickMath.getTickAtSqrtRatio(sqrtPriceX96);

    (uint16 cardinality, uint16 cardinalityNext) = observations.initialize(_blockTimestamp());

    slot0 = Slot0({
        sqrtPriceX96: sqrtPriceX96,
        tick: tick,
        observationIndex: 0,
        observationCardinality: cardinality,
        observationCardinalityNext: cardinalityNext,
        feeProtocol: 0,
        unlocked: true
    });

    emit Initialize(sqrtPriceX96, tick);
}
```



slot0 是池子合约中一个非常重要的存储槽（Struct），它包含了最重要的状态变量（价格、Tick等）。在首次初始化之前，slot0.sqrtPriceX96 的值肯定是 0。如果调用者尝试再次初始化，这个值已经被设置为一个非零的价格，因此require 检查会失败，并回滚交易，抛出错误代码 'AI'（猜测是 ‘Already Initialized’ 的缩写）。



- 计算初始Tick：

  - 作用：根据给定的sqrtPriceX96 计算出对应的 tick指数。

  - 深入：Uniswap V3 使用离散的「Tick」来管理价格。每个 Tick 对应一个特定的价格。

    TickMath.getTickAtSqrtRatio 是一个库函数，它实现了复杂的数学计算（包括二分查找和迭代），来找到给定 sqrtPriceX96

    所属的 Tick 索引。这个 tick 值将被存储在 slot0 中，代表资产的当前价格位置。

```
int24 tick = TickMath.getTickAtSqrtRatio(sqrtPriceX96);
```

- 初始化Oracle模块：
  - 作用：调用 observations 库的 initialize方法来设置预言机观察数组。
  - _blockTimestamp() ：获取当前区块的时间戳。
  - observations.initialize(...)：该方法会初始化一个环形缓冲区（ring buffer）来存储价格观察数据。它返回：
    - cardinality：当前观察数组的初始大小（通常是 1）。
    - cardinalityNext：观察数组下一次扩容后的目标大小（通常也是 1，但在需要时会增长）。
  - Oracle 功能是 Uniswap V3 的核心特性之一，它依靠这个历史价格数据缓冲区来提供可靠的时间加权平均价格（TWAP）。

```
(uint16 cardinality, uint16 cardinalityNext) = observations.initialize(_blockTimestamp());
```

最后初始化 slot0 变量，用于记录交易池的全局状态，这里主要就是记录价格和预言机的状态。

```
slot0 = Slot0({
    sqrtPriceX96: sqrtPriceX96,
    tick: tick,
    observationIndex: 0,
    observationCardinality: cardinality,
    observationCardinalityNext: cardinalityNext,
    feeProtocol: 0, 
    unlocked: true
});
```

[Slot0](https://github.com/Uniswap/v3-core/blob/main/contracts/UniswapV3Pool.sol#L56)结构如下，源码中已经有了详细的注释。

```
struct Slot0 {
    //设置池子的初始价格
    uint160 sqrtPriceX96;
    // the current tick
    int24 tick;
    // 初始化为 0，指向观察数组的第一个位置
    uint16 observationIndex;
    // the current maximum number of observations that are being stored
    uint16 observationCardinality;
    // the next maximum number of observations to store, triggered in observations.write
    uint16 observationCardinalityNext;
    // the current protocol fee as a percentage of the swap fee taken on withdrawal
    // represented as an integer denominator (1/x)%
    //协议手续费开关。0 表示默认情况下不收取协议手续费，所有手续费都归流动性提供者。
    uint8 feeProtocol;
    //初始化重入锁为解锁状态，允许后续的正常交易函数执行。
    bool unlocked;
}
```

至此完成了交易池合约的初始化。

#### 创建流动性

创建流动性调用的是 NonfungiblePositionManager 合约的 [mint](https://github.com/Uniswap/v3-periphery/blob/main/contracts/NonfungiblePositionManager.sol#L128)。这个函数是用户为池子**添加流动性并铸造代表该头寸的 NFT** 的主要入口。

参数如下：

```
struct MintParams {
    address token0; // token0 地址
    address token1; // token1 地址
    uint24 fee; // 费率 token0, token1, fee: 唯一标识一个 V3 Pool。
    int24 tickLower; // 流动性区间下界
    int24 tickUpper; // 流动性区间上界
    uint256 amount0Desired; // 添加流动性中 token0 数量 : 用户愿意提供的最大代币数量。合约会按最优比例计算实际需要存入的数量。
    uint256 amount1Desired; // 添加流动性中 token1 数量
    uint256 amount0Min; // 最小添加 token0 数量: 用户能接受的最少代币存入数量。这是防止前端计算出错或高滑点交易的保护措施。
    uint256 amount1Min; // 最小添加 token1 数量
    address recipient; // 头寸接受者的地址、接收新铸造的 NFT 的地址。
    uint256 deadline; // 交易有效的最后区块时间戳，防止过时交易被意外执行。
}
```

代码如下：

```
/// @inheritdoc INonfungiblePositionManager
function mint(MintParams calldata params)
    external
    payable
    override
    checkDeadline(params.deadline)
    returns (
        uint256 tokenId,
        uint128 liquidity,
        uint256 amount0,
        uint256 amount1
    )
{
    IUniswapV3Pool pool;
    (liquidity, amount0, amount1, pool) = addLiquidity(
        AddLiquidityParams({
            token0: params.token0,
            token1: params.token1,
            fee: params.fee,
            recipient: address(this),
            tickLower: params.tickLower,
            tickUpper: params.tickUpper,
            amount0Desired: params.amount0Desired,
            amount1Desired: params.amount1Desired,
            amount0Min: params.amount0Min,
            amount1Min: params.amount1Min
        })
    );

    _mint(params.recipient, (tokenId = _nextId++)); //: 继承自 ERC721 的标准函数，将 Token ID 为 _nextId 的 NFT 铸造给 		                                                           //params.recipient 地址

    //计算一个在池子合约中唯一标识此特定头寸的键（Key）。这个键由所有者地址、Tick下界和Tick上界共同生成。
    //这里的所有者地址是 address(this)，即 NonfungiblePositionManager 合约本身。这是因为所有流动性在池子看来都是由这个管理器合约拥有的。真正的     //所有者信息存储在管理器的 _positions 映射中。
    bytes32 positionKey = PositionKey.compute(address(this), params.tickLower, params.tickUpper);
    //调用池子合约的 positions 映射，获取该头寸的最新信息。这里最关键的是获取当前的 feeGrowthInside0LastX128 和 feeGrowthInside1LastX128。：这两个值记录了截至此刻，在该头寸价格区间内累计的每单位流动性应得的费用总数。
    //记录下这个初始值 (feeGrowthInsideLastX128) 至关重要，用于在未来计算该头寸自创建以来累计应得但尚未领取的费用。
    (, uint256 feeGrowthInside0LastX128, uint256 feeGrowthInside1LastX128, , ) = pool.positions(positionKey);

    // 缓存池子信息
    //cachePoolKey: 一个内部函数，它将池子地址和其关键参数（token0, token1, fee）存储在一个映射中，并返回一个唯一的 poolId（一个 uint80 的 ID）。节省存储成本。在头寸信息中存储一个 uint80 poolId 比存储整个 pool 地址或 PoolKey 结构体要便宜得多。当需要池子信息时，可以通过这个 poolId 反向查找。
    uint80 poolId =
        cachePoolKey(
            address(pool),
            PoolAddress.PoolKey({token0: params.token0, token1: params.token1, fee: params.fee})
        );
	//存储头寸信息 将新头寸的所有元数据存入管理器合约的 _positions 映射中，以 tokenId 为键。(重要记录了所有计算收益和管理头寸所需的数据)
    _positions[tokenId] = Position({
        nonce: 0,
        operator: address(0),
        poolId: poolId,
        tickLower: params.tickLower,
        tickUpper: params.tickUpper,
        liquidity: liquidity,
        feeGrowthInside0LastX128: feeGrowthInside0LastX128,
        feeGrowthInside1LastX128: feeGrowthInside1LastX128,
        tokensOwed0: 0,
        tokensOwed1: 0
    });

    emit IncreaseLiquidity(tokenId, liquidity, amount0, amount1);
}
```

梳理下整体逻辑，首先是 addLiquidity 添加流动性，然后调用 _mint 发送凭证（NFT）给头寸接受者，接着计算一个自增的 poolId，跟交易池地址互相索引，最后将所有信息记录到头寸的结构体中。

addLiquidity 方法定义在[这里](https://github.com/Uniswap/v3-periphery/blob/main/contracts/base/LiquidityManagement.sol#L51)，核心是计算出 liquidity 然后调用交易池合约 mint方法。

- 调用内部的addLiquidity 函数。这个函数会完成一系列繁重的工作：

  1. 获取或创建池子：根据 (token0, token1, fee)参数，通过UniswapV3Factory找到或创建对应的 Pool 合约。

  2. 计算流动性数量：根据当前池子的价格、用户指定的区间和期望的存款数量，计算出最优的liquidity值以及实际需要存入的amount0

     和amount1。

  3. 转移代币：将计算好的amount0和amount1从用户地址转移到本合约(address(this))。

  4. 调用池子的mint函数：最终调用 IUniswapV3Pool(pool).mint，将代币真正存入池中，并在指定的 Tick 区间内铸造流动性。池子的mint函数会返回实际的

     amount0 和 amount1（可能与计算的有细微差别 due to slippage）以及流动性数量liquidity。

```
 function addLiquidity(AddLiquidityParams memory params)
        internal
        returns (
            uint128 liquidity,
            uint256 amount0,
            uint256 amount1,
            IUniswapV3Pool pool
        )
    {
        PoolAddress.PoolKey memory poolKey =
            PoolAddress.PoolKey({token0: params.token0, token1: params.token1, fee: params.fee});

        pool = IUniswapV3Pool(PoolAddress.computeAddress(factory, poolKey));

        // compute the liquidity amount
        {
            (uint160 sqrtPriceX96, , , , , , ) = pool.slot0();
            uint160 sqrtRatioAX96 = TickMath.getSqrtRatioAtTick(params.tickLower);
            uint160 sqrtRatioBX96 = TickMath.getSqrtRatioAtTick(params.tickUpper);

            liquidity = LiquidityAmounts.getLiquidityForAmounts(
                sqrtPriceX96,
                sqrtRatioAX96,
                sqrtRatioBX96,
                params.amount0Desired,
                params.amount1Desired
            );
        }

        (amount0, amount1) = pool.mint(
            params.recipient,
            params.tickLower,
            params.tickUpper,
            liquidity,
            abi.encode(MintCallbackData({poolKey: poolKey, payer: msg.sender}))
        );

        require(amount0 >= params.amount0Min && amount1 >= params.amount1Min, 'Price slippage check');
    }
```

**liquidity ，即流动性，跟 tick 一样，也是 V3 中的重要概念。**

在 V2 中，如果我们设定乘积 k=L2， L 就是我们常说的流动性，得出如下公式：

L=x∗y*L*=*x*∗*y*

V2 流动性池的流动性是分布在 0 到正无穷，如下图所示：

[图片加载失败: img/liquidity.webp*liquidity*

在 v3 中，每个头寸提供了一个价格区间，假设 token0 的价格在价格上界 a 和价格下界 b 之间波动，为了实现集中流动性，那么曲线必须在 x/y 轴进行平移，使得 a/b 点和 x/y 轴重合，如下图：

图片加载失败: img/liquidityv3.webp*liquidityv3*

我们忽略推导过程，直接给出数学公式：

- *L*: 流动性
- *Pb*: 价格下界（lower tick）
- *Pa*: 价格上界（upper tick）
- *Pc*: 当前价格
- *x* 和 *y* 分别是 token0 和 token1 的数量，

(x+L/Pb)∗(y+L∗Pa)=L2(*x*+*L*/*P**b*)∗(*y*+*L*∗*P**a*)=*L*2

我们将图中的曲线分为两部分：起始点左边和起始点右边。在swap过程中，当前价格会朝着某个方向移动：升高或降低。对于价格的移动，仅有一种 token 会起作用：当前价格升高时，swap 仅需要 token0；当前价格降低时，swap 仅需要 token1。

当添加流动性时，用户提供代币数量 Δ*x*（token0）和/或 Δ*y*（token1）。系统根据当前价格 *Pc* 相对于区间 [[Pa,Pb][*P**a*,*P**b*] ]的位置计算流动性 L。以下是三种情况:

1. 当前价格在区间内（Pb≤Pc≤Pa)

当流动性提供者提供了 Δx 个 token0 时，意味着向起始点左边添加了如下流动性：

L=ΔxPb∗Pc/(Pb−Pc)*L*=Δ*x**P**b*∗*P**c*/(*P**b*−*P**c*)

当流动性提供者提供了 Δy 个 token1 时，意味着向起始点右边添加了如下流动性：

L=Δy/(Pc−Pa)*L*=Δ*y*/(*P**c*−*P**a*)

如果当前价格超过价格区间属于只能添加单边流动性的情况。

2. 当前价格低于下界（Pc<Pb）

当前价格小于下界 b 时，只有 Δy 个 token1 起作用，意味着向 b 点右边添加了如下流动性：

L=Δy/(Pb−Pa)*L*=Δ*y*/(*P**b*−*P**a*)

3. 当前价格高于上界（Pc>Pa）

当前价格大于上界 a 时，只有 Δx 个 token0 起作用，意味着向 a 点左边添加了如下流动性：

L=ΔxPb∗Pa/(Pb−Pa)*L*=Δ*x**P**b*∗*P**a*/(*P**b*−*P**a*)

回到代码，计算流动性方式：

1. 如果价格在价格区间内，分别计算出两边流动性然后取最小值；
2. 如果当前价格超过价格区间则是计算出单边流动性。

**交易池合约的 [mint](https://github.com/Uniswap/v3-core/blob/main/contracts/UniswapV3Pool.sol#L457)方法。**

参数为：

- recipient：头寸接收者地址
- tickLower：流动性区间下界
- tickUpper：流动性区间上界
- amount：流动性数量
- data：回调参数

代码为：

```
/// @inheritdoc IUniswapV3PoolActions
/// @dev noDelegateCall is applied indirectly via _modifyPosition
function mint(
    address recipient,
    int24 tickLower,
    int24 tickUpper,
    uint128 amount,
    bytes calldata data
) external override lock returns (uint256 amount0, uint256 amount1) {
    require(amount > 0);
    (, int256 amount0Int, int256 amount1Int) =
        _modifyPosition(
            ModifyPositionParams({
                owner: recipient,
                tickLower: tickLower,
                tickUpper: tickUpper,
                liquidityDelta: int256(amount).toInt128()
            })
        );

    amount0 = uint256(amount0Int);
    amount1 = uint256(amount1Int);

    uint256 balance0Before;
    uint256 balance1Before;
    if (amount0 > 0) balance0Before = balance0();
    if (amount1 > 0) balance1Before = balance1();
    IUniswapV3MintCallback(msg.sender).uniswapV3MintCallback(amount0, amount1, data);
    if (amount0 > 0) require(balance0Before.add(amount0) <= balance0(), 'M0');
    if (amount1 > 0) require(balance1Before.add(amount1) <= balance1(), 'M1');

    emit Mint(msg.sender, recipient, tickLower, tickUpper, amount, amount0, amount1);
}
```

首先调用 _modifyPosition 方法计算添加流动性所需的实际代币数量（ amount0 和 amount1），并更新池子的状态（如 tick 信息和流动性映射），这个方法相对复杂，放到后面专门讲。其返回的 amount0Int 和 amount1Int 表示 amount 流动性对应的 token0 和 token1 的代币数量。

调用 mint 方法的合约需要实现 IUniswapV3MintCallback 接口完成代币的转入操作：

```
IUniswapV3MintCallback(msg.sender).uniswapV3MintCallback(amount0, amount1, data);
```

IUniswapV3MintCallback 的实现在 periphery 仓库的 LiquidityManagement.sol 中。目的是通知调用方向交易池合约转入 amount0 个 token0 和 amount1 个 token2。

```
/// @inheritdoc IUniswapV3MintCallback
    function uniswapV3MintCallback(
        uint256 amount0Owed,
        uint256 amount1Owed,
        bytes calldata data
    ) external override {
        MintCallbackData memory decoded = abi.decode(data, (MintCallbackData));
        CallbackValidation.verifyCallback(factory, decoded.poolKey);

        if (amount0Owed > 0) pay(decoded.poolKey.token0, decoded.payer, msg.sender, amount0Owed);
        if (amount1Owed > 0) pay(decoded.poolKey.token1, decoded.payer, msg.sender, amount1Owed);
    }
```

回调完成后会检查交易池合约的对应余额是否发生变化，并且增量应该大于 amount0 和 amount1：这意味着调用方确实转入了所需的资产。

```
if (amount0 > 0) require(balance0Before.add(amount0) <= balance0(), 'M0');
if (amount1 > 0) require(balance1Before.add(amount1) <= balance1(), 'M1');
```

至此完成了流动性的创建。

#### 增加流动性

添加流动性调用的是 NonfungiblePositionManager 合约的 [increaseLiquidity](https://github.com/Uniswap/v3-periphery/blob/main/contracts/NonfungiblePositionManager.sol#L198)。 这个函数是用户为其**已有的**流动性头寸（NFT）**增加流动性**的主要方法,区别上面的mint

参数如下：

```
struct IncreaseLiquidityParams {
    uint256 tokenId; // 头寸 id
    uint256 amount0Desired; // 添加流动性中 token0 数量
    uint256 amount1Desired; // 添加流动性中 token1 数量
    uint256 amount0Min; // 最小添加 token0 数量
    uint256 amount1Min; // 最小添加 token1 数量
    uint256 deadline; // 过期的区块号
}
```

代码如下：

```
/// @inheritdoc INonfungiblePositionManager
function increaseLiquidity(IncreaseLiquidityParams calldata params)
    external
    payable
    override
    checkDeadline(params.deadline)
    returns (
        uint128 liquidity,
        uint256 amount0,
        uint256 amount1
    )
{
    Position storage position = _positions[params.tokenId]; //通过 params.tokenId 从存储映射 _positions 中获取对应的 头寸信息
    //根据头寸中存储的 poolId（一个压缩的标识符），从另一个映射 _poolIdToPoolKey 中查询出完整的池子密钥信息 poolKey（包含 token0, token1, fee）。这是在 mint 时缓存的，现在用于重新定位到正确的池子合约。
    PoolAddress.PoolKey memory poolKey = _poolIdToPoolKey[position.poolId];

    IUniswapV3Pool pool;
    (liquidity, amount0, amount1, pool) = addLiquidity(
        AddLiquidityParams({
            token0: poolKey.token0,
            token1: poolKey.token1,
            fee: poolKey.fee, 
            tickLower: position.tickLower, // 使用已有头寸的区间
            tickUpper: position.tickUpper, // 使用已有头寸的区间
            amount0Desired: params.amount0Desired,
            amount1Desired: params.amount1Desired,
            amount0Min: params.amount0Min,
            amount1Min: params.amount1Min,
            recipient: address(this)
        })
    );
     //计算该头寸在池子合约中的键 positionKey。
    bytes32 positionKey = PositionKey.compute(address(this), position.tickLower, position.tickUpper);

    //查询池子合约，获取该头寸当前的 feeGrowthInside0LastX128 和 feeGrowthInside1LastX128。这个值反映了到此刻为止，该头寸区间内累计的每单位流动性费用总和。
    (, uint256 feeGrowthInside0LastX128, uint256 feeGrowthInside1LastX128, , ) = pool.positions(positionKey);

    //关键步骤：结算未领取的费用
    position.tokensOwed0 += uint128(
    	//将费用增长因子差值乘以头寸原有的流动性数量，再除以 Q128（一个固定点数精度常量），得到应累加的费用代币数量。
        FullMath.mulDiv(
            feeGrowthInside0LastX128 - position.feeGrowthInside0LastX128, //计算出自上次记录以来，费用增长因子的差值（ΔfeeGrowth）。
            position.liquidity,
            FixedPoint128.Q128
        )
    );
    position.tokensOwed1 += uint128(
        FullMath.mulDiv(
            feeGrowthInside1LastX128 - position.feeGrowthInside1LastX128,//计算出自上次记录以来，费用增长因子的差值（ΔfeeGrowth）。
            position.liquidity,
            FixedPoint128.Q128
        )
    );

    position.feeGrowthInside0LastX128 = feeGrowthInside0LastX128;
    position.feeGrowthInside1LastX128 = feeGrowthInside1LastX128;
    position.liquidity += liquidity;

    emit IncreaseLiquidity(params.tokenId, liquidity, amount0, amount1);
}
```

整体逻辑跟 mint 类似，先从 tokeinId 拿到头寸，然后 addLiquidity 添加流动性，返回添加成功的流动性 liquidity，所消耗的 amount0 和 amount1，以及交易池合约 pool。根据 pool 对象里的最新头寸信息，更新头寸状态。

- 为什么必须现在结算？

  因为接下来就要增加头寸的流动性 

  ```
  position.liquidity += liquidity。
  ```

​	如果不先结算旧流动性产生的费用，那么新添加的流动性也会参与到未来的费用计算中。如果现在不结算，那么旧流动性应得的费用就会错误地与新流动性共	享。这一步确保了在改变流动性数量之前，先将此前累计的收益准确无误地记录到tokensOwed中。

#### 减少流动性

减少流动性调用的是 NonfungiblePositionManager 合约的 [decreaseLiquidity](https://github.com/Uniswap/v3-periphery/blob/main/contracts/NonfungiblePositionManager.sol#L257)。

参数如下：

```
struct DecreaseLiquidityParams {
    uint256 tokenId; // 头寸 id
    uint128 liquidity; // 减少流动性数量
    uint256 amount0Min; // 最小减少 token0 数量 用户能接受的最少返还的 token0 和 token1 数量（包含本金+费用）
    uint256 amount1Min; // 最小减少 token1 数量
    uint256 deadline; // 过期的区块号
}
```

代码如下：

```
/// @inheritdoc INonfungiblePositionManager
function decreaseLiquidity(DecreaseLiquidityParams calldata params)
    external
    payable
    override
    isAuthorizedForToken(params.tokenId)
    checkDeadline(params.deadline)
    returns (uint256 amount0, uint256 amount1)
{
    require(params.liquidity > 0); // 确保要移除的流动性大于0
    Position storage position = _positions[params.tokenId];  // 获取存储中的头寸信息

    uint128 positionLiquidity = position.liquidity;
    require(positionLiquidity >= params.liquidity);  // 确保头寸的流动性足够被移除

    PoolAddress.PoolKey memory poolKey = _poolIdToPoolKey[position.poolId]; //根据头寸中存储的 poolId 找到对应的 poolKey
    使用工厂合约地址和 poolKey 计算出目标池子的合约地址，并创建接口实例 pool。
    IUniswapV3Pool pool = IUniswapV3Pool(PoolAddress.computeAddress(factory, poolKey));
    (amount0, amount1) = pool.burn(position.tickLower, position.tickUpper, params.liquidity);
    // 滑点检查：确保实际返还的本金数量大于用户设定的最小值。如果因为价格变动导致返还的代币过少，交易将回滚并抛出 'Price slippage check' 错误。
    require(amount0 >= params.amount0Min && amount1 >= params.amount1Min, 'Price slippage check');
    //获取最新的费用增长因子：
    bytes32 positionKey = PositionKey.compute(address(this), position.tickLower, position.tickUpper);
    // this is now updated to the current transaction
    (, uint256 feeGrowthInside0LastX128, uint256 feeGrowthInside1LastX128, , ) = pool.positions(positionKey);

    //关键步骤：结算所有应得资产
    position.tokensOwed0 +=
        uint128(amount0) +
        uint128(
            FullMath.mulDiv(
                feeGrowthInside0LastX128 - position.feeGrowthInside0LastX128,
                positionLiquidity,
                FixedPoint128.Q128
            )
        );
    position.tokensOwed1 +=
        uint128(amount1) +
        uint128(
            FullMath.mulDiv(
                feeGrowthInside1LastX128 - position.feeGrowthInside1LastX128,
                positionLiquidity,
                FixedPoint128.Q128
            )
        );
    //更新费用快照：将头寸的费用增长因子记录更新为最新的查询值
    position.feeGrowthInside0LastX128 = feeGrowthInside0LastX128;
    position.feeGrowthInside1LastX128 = feeGrowthInside1LastX128;
    // 更新头寸流动性：从总流动性中减去移除的部分。
    position.liquidity = positionLiquidity - params.liquidity;

    emit DecreaseLiquidity(params.tokenId, params.liquidity, amount0, amount1);
}
```

跟 increaseLiquidity 是反向操作，核心逻辑是调用交易池合约的 burn方法。

```
(amount0, amount1) = pool.burn(position.tickLower, position.tickUpper, params.liquidity);
```

[burn](https://github.com/Uniswap/v3-core/blob/main/contracts/UniswapV3Pool.sol#L517) 的参数为流动性区间下界 tickLower，流动性区间上界 tickUpper 和流动性数量 amount，

- 调用池子合约的burn方法。这个方法会：
  1. 根据当前价格和提供的流动性数量，计算应返还给流动性提供者的代币数量 (amount0,amount1)。
  2. 从对应 Tick 区间的流动性中减去params.liquidity。
  3. 将这些代币的本金部分记录在池子合约中，等待被所有者（即NonfungiblePositionManager合约）取走。

代码如下：

```
/// @inheritdoc IUniswapV3PoolActions
/// @dev noDelegateCall is applied indirectly via _modifyPosition
function burn(
    int24 tickLower,
    int24 tickUpper,
    uint128 amount
) external override lock returns (uint256 amount0, uint256 amount1) {
    (Position.Info storage position, int256 amount0Int, int256 amount1Int) =
        _modifyPosition(
            ModifyPositionParams({
                owner: msg.sender,
                tickLower: tickLower,
                tickUpper: tickUpper,
                liquidityDelta: -int256(amount).toInt128()
            })
        );

    amount0 = uint256(-amount0Int);
    amount1 = uint256(-amount1Int);

    if (amount0 > 0 || amount1 > 0) {
        (position.tokensOwed0, position.tokensOwed1) = (
            position.tokensOwed0 + uint128(amount0),
            position.tokensOwed1 + uint128(amount1)
        );
    }

    emit Burn(msg.sender, tickLower, tickUpper, amount, amount0, amount1);
}
```

也是调用 _modifyPosition 方法修改当前价格区间的流动性，返回的 amount0Int 和 amount1Int 表示 amount 流动性对应的 token0 和 token1 的代币数量，position 表示用户的头寸信息，在这里主要作用是用来记录待取回代币数量。



```
if (amount0 > 0 || amount1 > 0) {
    (position.tokensOwed0, position.tokensOwed1) = (
        position.tokensOwed0 + uint128(amount0),
        position.tokensOwed1 + uint128(amount1)
    );
}
```

用户可以通过主动调用 collect 方法取出自己头寸信息记录的 tokensOwed0 数量的 token0 和 tokensOwed1 数量对应的 token1。collect方法在下一节展开。



#### `collect`

取出待领取代币调用的是 NonfungiblePositionManager 合约的 [collect](https://github.com/Uniswap/v3-periphery/blob/main/contracts/NonfungiblePositionManager.sol#L309)。



##### 函数概述

- 功能：从一个由 tokenId标识的头寸中提取最多指定数量的待领取代币（token0 和 token1）。
- 调用者：NFT 的所有者或被授权的操作员（通过 isAuthorizedForToken 修饰符检查）。
- **核心作用**：首先更新头寸的待领取代币数量（如果有流动性，则先结算自上次操作以来的费用），然后从池子中提取指定数量的代币，并更新头寸的待领取代币余额。

参数如下：

```
struct CollectParams {
    uint256 tokenId; // 头寸 id
    address recipient; // 接收者地址
    uint128 amount0Max; // 最大 token0 数量
    uint128 amount1Max; // 最大 token1 数量
}
```

代码如下：

```
/// @inheritdoc INonfungiblePositionManager
function collect(CollectParams calldata params)
    external
    payable
    override
    isAuthorizedForToken(params.tokenId)
    returns (uint256 amount0, uint256 amount1)
{
    require(params.amount0Max > 0 || params.amount1Max > 0);
    // allow collecting to the nft position manager address with address 0
    address recipient = params.recipient == address(0) ? address(this) : params.recipient;

    Position storage position = _positions[params.tokenId];//从存储中获取头寸信息

    //通过 poolId 获取池子的密钥，然后计算出池子的合约地址。
    PoolAddress.PoolKey memory poolKey = _poolIdToPoolKey[position.poolId];

    IUniswapV3Pool pool = IUniswapV3Pool(PoolAddress.computeAddress(factory, poolKey));
    //将头寸中当前记录的待领取代币数量缓存到局部变量 tokensOwed0 和 tokensOwed1。
    (uint128 tokensOwed0, uint128 tokensOwed1) = (position.tokensOwed0, position.tokensOwed1);

    //如果头寸还有流动性，则更新费用信息：
    if (position.liquidity > 0) {
    	//传入流动性为0。这是一个技巧，目的是触发池子更新该头寸的费用增长因子（因为 burn 函数会更新头寸的 tokensOwed 和费用增长因子）。
        pool.burn(position.tickLower, position.tickUpper, 0);
        //查询池子获取最新的 feeGrowthInside0LastX128 和 feeGrowthInside1LastX128。
        (, uint256 feeGrowthInside0LastX128, uint256 feeGrowthInside1LastX128, , ) =
            pool.positions(PositionKey.compute(address(this), position.tickLower, position.tickUpper));
		//计算自上次更新以来，头寸的流动性新产生的费用，并将其加到缓存的 tokensOwed0 和 tokensOwed1 上。
        tokensOwed0 += uint128(
            FullMath.mulDiv(
                feeGrowthInside0LastX128 - position.feeGrowthInside0LastX128,
                position.liquidity,
                FixedPoint128.Q128
            )
        );
        tokensOwed1 += uint128(
            FullMath.mulDiv(
                feeGrowthInside1LastX128 - position.feeGrowthInside1LastX128,
                position.liquidity,
                FixedPoint128.Q128
            )
        );
		//更新头寸存储中的费用增长因子为最新值
        position.feeGrowthInside0LastX128 = feeGrowthInside0LastX128;
        position.feeGrowthInside1LastX128 = feeGrowthInside1LastX128;
    }

    // 计算实际要提取的数量：如果用户指定的最大提取量（params.amount0Max 或 params.amount1Max）大于待领取的代币数量，则提取全部待领取的代币；否则提取指定的最大数量。
    (uint128 amount0Collect, uint128 amount1Collect) =
        (
            params.amount0Max > tokensOwed0 ? tokensOwed0 : params.amount0Max,
            params.amount1Max > tokensOwed1 ? tokensOwed1 : params.amount1Max
        );

    // 从池子中提取代币：池子的 `collect` 方法会返回实际提取的数量。
    (amount0, amount1) = pool.collect(
        recipient,
        position.tickLower,
        position.tickUpper,
        amount0Collect,
        amount1Collect
    );
	//更新头寸的待领取代币余额：
    //注意：这里减去的是 amount0Collect 和 amount1Collect（即请求提取的数量），而不是实际提取的数量 amount0 和 amount1。
    //注释解释：由于池子核心合约中的向下取整，实际提取的数量可能会比请求的少几个 wei，但这里仍然按请求的数量来减少待领取余额。这样做的目的是为了确保待领取余额能够被完全清零（否则可能会留下一点点余额无法提取）。
    //实际上，因为我们在计算可提取数量时已经 capped by tokensOwed，所以 amount0Collect 和 amount1Collect 不会超过 tokensOwed，因此减法不会下溢。
    (position.tokensOwed0, position.tokensOwed1) = (tokensOwed0 - amount0Collect, tokensOwed1 - amount1Collect);

    emit Collect(params.tokenId, recipient, amount0Collect, amount1Collect);
}
```

首先获取待取回代币数量，如果该头寸含有流动性，则触发一次头寸状态的更新，这里调用了交易池合约的burn方法，但是传入的流动性参数为 0。这是因为 V3 只在mint和burn时才更新头寸状态，而collect方法可能在swap之后被调用，可能会导致头寸状态不是最新的。

最后调用了交易池合约的collect方法取回代币。

- 调用池子的collect方法，将计算出的amount0Collect和amount1Collect作为参数，请求池子将代币发送给recipient。
- 池子的collect方法会返回实际提取的数量。

```
// the actual amounts collected are returned
(amount0, amount1) = pool.collect(
    recipient,
    position.tickLower,
    position.tickUpper,
    amount0Collect,
    amount1Collect
);
```

交易池合约的 [collect](https://github.com/Uniswap/v3-core/blob/main/contracts/UniswapV3Pool.sol#L490) 的逻辑比较简单，这里就不展开了

#### `_modifyPosition`

[_modifyPosition](https://github.com/Uniswap/v3-core/blob/main/contracts/UniswapV3Pool.sol#L306) 方法是mint和burn的核心方法。用于修改一个流动性头寸, 根据当前价格位置和用户指定的价格区间，计算需要提供或返还的代币数量。



参数如下：

```
struct ModifyPositionParams {
    // the address that owns the position
    address owner;
    // the lower and upper tick of the position
    int24 tickLower;
    int24 tickUpper;
    // 流动性变化量（正数为增加，负数为减少）
    int128 liquidityDelta;
}
```

代码如下：

```
/// @dev Effect some changes to a position
/// @param params the position details and the change to the position's liquidity to effect
/// @return position a storage pointer referencing the position with the given owner and tick range
/// @return amount0 the amount of token0 owed to the pool, negative if the pool should pay the recipient
/// @return amount1 the amount of token1 owed to the pool, negative if the pool should pay the recipient
function _modifyPosition(ModifyPositionParams memory params)
    private
    noDelegateCall
    returns (
        Position.Info storage position,
        int256 amount0,
        int256 amount1
    )
{
    checkTicks(params.tickLower, params.tickUpper);

    Slot0 memory _slot0 = slot0; // SLOAD for gas optimization

    position = _updatePosition(
        params.owner,
        params.tickLower,
        params.tickUpper,
        params.liquidityDelta,
        _slot0.tick
    );

    if (params.liquidityDelta != 0) {
    	//根据当前价格位置计算代币数量
        if (_slot0.tick < params.tickLower) {
            // 当前价格在区间下方; 流动性只能通过价格从右到左穿越区间时激活
            // 只需要提供token0（因为token0相对更有价值）
            amount0 = SqrtPriceMath.getAmount0Delta(
                TickMath.getSqrtRatioAtTick(params.tickLower),
                TickMath.getSqrtRatioAtTick(params.tickUpper),
                params.liquidityDelta
            );
        } else if (_slot0.tick < params.tickUpper) {
            // 当前价格在区间内
            uint128 liquidityBefore = liquidity; // SLOAD for gas optimization

            // 写入预言机数据
            (slot0.observationIndex, slot0.observationCardinality) = observations.write(
                _slot0.observationIndex,
                _blockTimestamp(),
                _slot0.tick,
                liquidityBefore,
                _slot0.observationCardinality,
                _slot0.observationCardinalityNext
            );
			// 计算两种代币的数量
            amount0 = SqrtPriceMath.getAmount0Delta(
                _slot0.sqrtPriceX96,
                TickMath.getSqrtRatioAtTick(params.tickUpper),
                params.liquidityDelta
            );
            //
            amount1 = SqrtPriceMath.getAmount1Delta(
                TickMath.getSqrtRatioAtTick(params.tickLower),
                _slot0.sqrtPriceX96,
                params.liquidityDelta
            );
			// 更新全局流动性
            liquidity = LiquidityMath.addDelta(liquidityBefore, params.liquidityDelta);
        } else {
            //当前价格高于区间; 流动性只能通过价格从左到右穿越区间时激活
            // 只需要提供token1（因为token1相对更有价值）
            amount1 = SqrtPriceMath.getAmount1Delta(
                TickMath.getSqrtRatioAtTick(params.tickLower),
                TickMath.getSqrtRatioAtTick(params.tickUpper),
                params.liquidityDelta
            );
        }
    }
}
```

先通过_updatePosition更新头寸信息，接着分别计算出 liquidityDelta 流动性需要提供的 token0 数量 amount0 和 token1 数量 amount1，流动性的计算公式在创建流动性时已经介绍了。



[_updatePosition](https://github.com/Uniswap/v3-core/blob/main/contracts/UniswapV3Pool.sol#L379) 方法负责更新用户的流动性头寸，包括：

1. 更新tick点的流动性状态
2. 计算和更新费用累积
3. 管理tick位图
4. 清理不再需要的tick数据

代码如下：

```
/// @dev Gets and updates a position with the given liquidity delta
/// @param owner the owner of the position
/// @param tickLower the lower tick of the position's tick range
/// @param tickUpper the upper tick of the position's tick range
/// @param tick the current tick, passed to avoid sloads
function _updatePosition(
    address owner,
    int24 tickLower,
    int24 tickUpper,
    int128 liquidityDelta,
    int24 tick
) private returns (Position.Info storage position) {
	//获取头寸和费用数据
    position = positions.get(owner, tickLower, tickUpper);

    uint256 _feeGrowthGlobal0X128 = feeGrowthGlobal0X128; // SLOAD for gas optimization
    uint256 _feeGrowthGlobal1X128 = feeGrowthGlobal1X128; // SLOAD for gas optimization

    //  更新Tick状态（当流动性变化时
    bool flippedLower;
    bool flippedUpper;
    if (liquidityDelta != 0) {
        uint32 time = _blockTimestamp();
        //tickCumulative: 累积tick值（用于计算时间加权平均价格） secondsPerLiquidityCumulativeX128: 累积的每流动性秒数
        (int56 tickCumulative, uint160 secondsPerLiquidityCumulativeX128) =
            observations.observeSingle(
                time,
                0,
                slot0.tick,
                slot0.observationIndex,
                liquidity,
                slot0.observationCardinality
            );
		//更新tick点的流动性总量
		//更新费用累积数据
        //更新预言机相关数据
        //返回是否"翻转"（从有流动性变为无流动性，或反之）
        flippedLower = ticks.update(
            tickLower,
            tick,
            liquidityDelta,
            _feeGrowthGlobal0X128,
            _feeGrowthGlobal1X128,
            secondsPerLiquidityCumulativeX128,
            tickCumulative,
            time,
            false,
            maxLiquidityPerTick
        );
        flippedUpper = ticks.update(
            tickUpper,
            tick,
            liquidityDelta,
            _feeGrowthGlobal0X128,
            _feeGrowthGlobal1X128,
            secondsPerLiquidityCumulativeX128,
            tickCumulative,
            time,
            true,
            maxLiquidityPerTick
        );
		//位图管理：
		//当tick从无流动性变为有流动性时，在位图中标记该tick
        //位图用于快速查找下一个活跃的tick点
        if (flippedLower) {
            tickBitmap.flipTick(tickLower, tickSpacing);
        }
        if (flippedUpper) {
            tickBitmap.flipTick(tickUpper, tickSpacing);
        }
    }
	//计算区间内费用累积
    (uint256 feeGrowthInside0X128, uint256 feeGrowthInside1X128) =
        ticks.getFeeGrowthInside(tickLower, tickUpper, tick, _feeGrowthGlobal0X128, _feeGrowthGlobal1X128);
     // 更新头寸信息
    position.update(liquidityDelta, feeGrowthInside0X128, feeGrowthInside1X128);

    /当tick从有流动性变为无流动性时，从位图中清除该tick
    if (liquidityDelta < 0) {
        if (flippedLower) {
            ticks.clear(tickLower);
        }
        if (flippedUpper) {
            ticks.clear(tickUpper);
        }
    }
}
```

ticktickCumulative 和 secondsPerLiquidityCumulativeX128 是预言机观察点相关的两个变量，这里不详细解释。

```
(int56 tickCumulative, uint160 secondsPerLiquidityCumulativeX128) =
    observations.observeSingle(
        time,
        0,
        slot0.tick,
        slot0.observationIndex,
        liquidity,
        slot0.observationCardinality
    );
```

接着使用ticks.update分别更新价格区间低点和价格区间高点的状态。如果对应 tick 的流动性从从无到有，或从有到无，则表示该 tick 需要被翻转。

```
flippedLower = ticks.update(
    tickLower,
    tick,
    liquidityDelta,
    _feeGrowthGlobal0X128,
    _feeGrowthGlobal1X128,
    secondsPerLiquidityCumulativeX128,
    tickCumulative,
    time,
    false,
    maxLiquidityPerTick
);
flippedUpper = ticks.update(
    tickUpper,
    tick,
    liquidityDelta,
    _feeGrowthGlobal0X128,
    _feeGrowthGlobal1X128,
    secondsPerLiquidityCumulativeX128,
    tickCumulative,
    time,
    true,
    maxLiquidityPerTick
);
```

随后计算该价格区间的累积的流动性手续费。

```
(uint256 feeGrowthInside0X128, uint256 feeGrowthInside1X128) =
    ticks.getFeeGrowthInside(tickLower, tickUpper, tick, _feeGrowthGlobal0X128, _feeGrowthGlobal1X128);
```

最后更新头寸信息，并判断是否 tick 被翻转，如果 tick 被翻转则调用ticks.clear清空 tick 状态。

```
position.update(liquidityDelta, feeGrowthInside0X128, feeGrowthInside1X128);
// clear any tick data that is no longer needed
if (liquidityDelta < 0) {
    if (flippedLower) {

    }
    if (flippedUpper) {
        ticks.clear(tickUpper);
    }
}
```

至此完成更新头寸流程。

#### swap

swap 也就指交易，是 Uniswap 中最常用的也是最核心的功能。对应 https://app.uniswap.org/swap 中的相关操作，接下来让我们看看 Uniswap 的合约是如何实现 swap 的。

SwapRouter合约包含了以下四个交换代币的方法：

- exactInput：多池交换，用户指定输入代币数量，尽可能多地获得输出代币；
- exactInputSingle：单池交换，用户指定输入代币数量，尽可能多地获得输出代币；
- exactOutput：多池交换，用户指定输出代币数量，尽可能少地提供输入代币；
- exactOutputSingle：单池交换，用户指定输出代币数量，尽可能少地提供输入代币。

这里分成"指定输入代币数量"和"指定输出代币数量"分别介绍。

##### 指定输入代币数量

[exactInput](https://github.com/Uniswap/v3-periphery/blob/main/contracts/SwapRouter.sol#L132) 方法负责**多池交换**，指定 swap 路径以及输入代币数量，尽可能多地获得输出代币。

参数如下：

```
struct ExactInputParams {
    bytes path; // swap 路径，可以解析成一个或多个交易池
    address recipient; //  最终接收输出代币的地址
    uint256 deadline; // 过期的区块号
    uint256 amountIn; // 你想要精确花费的输入代币数量。
    uint256 amountOutMinimum; // 你所能接受的最少的输出代币数量。这是应对滑点的保护措施
}
```

代码如下：

```
/// @inheritdoc ISwapRouter
function exactInput(ExactInputParams memory params)
    external
    payable
    override
    checkDeadline(params.deadline)
    returns (uint256 amountOut)
{
    address payer = msg.sender; // msg.sender pays for the first hop

    while (true) {
        //检查编码的路径中是否包含超过一个的流动性池（即是否还有后续的交换需要执行）。
        bool hasMultiplePools = params.path.hasMultiplePools();

        //执行单次交换（核心）
        //exactInputInternal 返回本次交换实际得到的输出代币数量，并更新到 params.amountIn 中，为下一次循环（如果有）做准备。
        params.amountIn = exactInputInternal(
            params.amountIn,// 本次交换的输入量 后续循环时，它是上一次交换的输出量，作为本次交换的输入量
            hasMultiplePools ? address(this) : params.recipient, //如果还有后续池子 那么本次交换的输出代币应该发送到当前合约的地址 
            0,
            SwapCallbackData({
                path: params.path.getFirstPool(), // 只获取当前第一个池子的路径信息 因为一次 exactInputInternal 调用只处理一个池子。
                payer: payer //谁应该提供本次交换的输入代币
            })
        );

        // decide whether to continue or terminate
  
        if (hasMultiplePools) {
            payer = address(this); // 从下一个池子开始，支付者是本合约（因为它托管了中间代币）
            params.path = params.path.skipToken();   // “消耗”掉路径中已经处理完的第一个代币，移动到下一个池子
        } else {
            amountOut = params.amountIn;   // 循环结束，最终的输出量就是最后一次交换的输出量
            break;
        }
    }
	//在函数最后，检查实际得到的输出代币数量 amountOut 是否大于等于用户设定的最小值
    require(amountOut >= params.amountOutMinimum, 'Too little received');
}
```

在多池 swap 中，会按照 swap 路径，拆成多个单池 swap，循环进行，直到路径结束。如果是第一步 swap。payer 为合约调用方，否则 payer 为当前SwapRouter合约。

[exactInputSingle](https://github.com/Uniswap/v3-periphery/blob/main/contracts/SwapRouter.sol#L115)方法负责单池交换，指定输入代币数量，尽可能多地获得输出代币。

参数如下，指定了输入代币地址和输出代币地址：

```
struct ExactInputSingleParams {
    address tokenIn; // 输入代币地址
    address tokenOut; // 输出代币地址
    uint24 fee; // 手续费费率
    address recipient; // 接收者地址
    uint256 deadline; // 过期的区块号
    uint256 amountIn; // 输入代币数量
    uint256 amountOutMinimum; // 最少输出代币数量
    uint160 sqrtPriceLimitX96; // 限定价格，值为0则不限价
}
```

代码如下：

```
/// @inheritdoc ISwapRouter
function exactInputSingle(ExactInputSingleParams calldata params)
    external
    payable
    override
    checkDeadline(params.deadline)
    returns (uint256 amountOut)
{
    amountOut = exactInputInternal(
        params.amountIn,
        params.recipient,
        params.sqrtPriceLimitX96,
        SwapCallbackData({path: abi.encodePacked(params.tokenIn, params.fee, params.tokenOut), payer: msg.sender})
    );
    require(amountOut >= params.amountOutMinimum, 'Too little received');
}
```

实际调用 [exactInputInternal](https://github.com/Uniswap/v3-periphery/blob/main/contracts/SwapRouter.sol#L87)，代码如下：

```
/// @dev Performs a single exact input swap
function exactInputInternal(
    uint256 amountIn,   // 本次交换要投入的确切输入代币数量
    address recipient,     // 本次交换输出的接收地址
    uint160 sqrtPriceLimitX96, // 价格限制，用于防止过度滑点
    SwapCallbackData memory data  // 回调数据，包含路径和支付者信息
) private returns (uint256 amountOut) {
    // allow swapping to the router address with address 0
    if (recipient == address(0)) recipient = address(this);
	
	//从编码的 path 中解析出本次交换所需的三个核心要素：
    (address tokenIn, address tokenOut, uint24 fee) = data.path.decodeFirstPool();
	//需要确定交换的方向
    bool zeroForOne = tokenIn < tokenOut;

    (int256 amount0, int256 amount1) =
        getPool(tokenIn, tokenOut, fee).swap( // 获取或创建池子合约，并调用其swap方法
            recipient, // 输出代币的接收者
            zeroForOne,// 交换方向
            amountIn.toInt256(),  // 精确的输入数量
            sqrtPriceLimitX96 == 0 // 计算价格限制：
                ? (zeroForOne ? TickMath.MIN_SQRT_RATIO + 1 : TickMath.MAX_SQRT_RATIO - 1)
                : sqrtPriceLimitX96,
            abi.encode(data) // 传递给池子的回调数据
        );
	//正数表示池子收到该代币，负数表示池子付出该代币。
	//如果是 zeroForOne，池子付出的是 amount1（为负值）。我们取它的负数 -amount1 并将其转换为 uint256，就得到了我们得到的 token1 的正数量。
    //如果是 !zeroForOne，池子付出的是 amount0（为负值）。我们取 -amount0 得到得到的 token0 的正数量。
    return uint256(-(zeroForOne ? amount1 : amount0));
}
```

如果没有指定接收者地址，则默认为当前SwapRouter合约地址。这个目的是在多池交易中，将中间代币保存在SwapRouter合约中。

```
if (recipient == address(0)) recipient = address(this);
```

接着解析出交易路由信息 tokenIn，tokenOut 和 fee。并比较 tokenIn 和 tokenOut 的地址得到 zeroForOne，表示在当前交易池是否是 token0 交换 token1。

- 这是 Uniswap V3 的一个核心概念。由于池子中的代币是按地址排序的（token0和token1,token0<token1)，我们需要确定交换的方向。
- zeroForOne:
  - 如果为true，表示用token0兑换token1。在这种情况下，tokenIn是token0，tokenOut是token1。
  - 如果为false，表示用token1兑换token0。在这种情况下，tokenIn是token1，tokenOut是token0。

```
(address tokenIn, address tokenOut, uint24 fee) = data.path.decodeFirstPool();

bool zeroForOne = tokenIn < tokenOut;
```

价格限制:

- 如果调用者没有设置限制 (sqrtPriceLimitX96 == 0)，则系统会设置一个默认的、极端但可达的限制。
  - 如果是zeroForOne（卖token0买token1），价格会下降。下限设置为TickMath.MIN_SQRT_RATIO + 1（最小价格 +1，避免溢出等问题）。
  - 如果是!zeroForOne（卖token1买token0），价格会上升。上限设置为TickMath.MAX_SQRT_RATIO - 1（最大价格 -1）。
- 如果调用者设置了限制，则使用该限制。

```
sqrtPriceLimitX96 == 0 // 计算价格限制：
                ? (zeroForOne ? TickMath.MIN_SQRT_RATIO + 1 : TickMath.MAX_SQRT_RATIO - 1)
                : sqrtPriceLimitX96,
```

最后调用交易池合约的swap方法，获取完成本次交换所需的 amount0 和 amount1，再根据 zeroForOne 返回 amountOut，进一步判断 amountOut 满足最少输出代币数量的要求，完成 swap。

swap方法相对复杂，放到后面专门讲。



##### 指定输出代币数量

[exactOutput](https://github.com/Uniswap/v3-periphery/blob/main/contracts/SwapRouter.sol#L224) 方法负责多池交换，指定 swap 路径以及输出代币数量，尽可能少地提供输入代币。与exactInput的正向路径处理不同，exactOutput采用了一种**反向处理**的方式：

- ```
  exactInput: A → B → C (正向处理)
  ```

- ```
  exactOutput: C → B → A (反向处理)
  ```

这种设计意味着exactOutput会先执行最后一步交换（得到精确的最终输出），然后逐步向前执行交换，直到得到最初需要的输入代币。

参数如下：

```
struct ExactOutputParams {
    bytes path; // 交换路径（编码的路径，顺序与 exactInput 相反）
    address recipient; // 接收者地址
    uint256 deadline; // 过期的区块号
    uint256 amountOut; // :期望得到的精确数量的输出代币
    uint256 amountInMaximum; //  愿意花费的最多的输入代币数量
}
```

代码如下：

```
/// @inheritdoc ISwapRouter
function exactOutput(ExactOutputParams calldata params)
    external
    payable
    override
    checkDeadline(params.deadline)
    returns (uint256 amountIn)
{
    // it's okay that the payer is fixed to msg.sender here, as they're only paying for the "final" exact output
    // swap, which happens first, and subsequent swaps are paid for within nested callback frames
    exactOutputInternal(
        params.amountOut,
        params.recipient,
        0,  // sqrtPriceLimitX96，这里设为0表示不使用价格限制 表示使用默认的价格限制
        SwapCallbackData({path: params.path, payer: msg.sender})  // 使用预编码的多跳路径
    );
 
    amountIn = amountInCached;  // 从全局状态读取 使用全局变量 amountInCached 来传递结果
    require(amountIn <= params.amountInMaximum, 'Too much requested');
    amountInCached = DEFAULT_AMOUNT_IN_CACHED; // 必须重置全局状态 这是为了支持多跳交换的复杂回调机制
}
```

在多池 swap 中，会按照 swap 路径，拆成多个单池 swap，循环进行，直到路径结束。如果是第一步 swap。payer 为合约调用方，否则 payer 为当前SwapRouter合约。

[exactOutputSingle](https://github.com/Uniswap/v3-periphery/blob/main/contracts/SwapRouter.sol#L203)方法负责单池交换，指定输出代币数量，尽可能少地提供输入代币。

参数如下，指定了输入代币地址和输出代币地址：

```
struct ExactOutputSingleParams {
    address tokenIn; // 输入代币地址
    address tokenOut; // 输出代币地址
    uint24 fee; // 手续费费率
    address recipient; // 接收者地址
    uint256 deadline; // 过期的区块号
    uint256 amountOut; // 输出代币数量
    uint256 amountInMaximum; // 最多输入代币数量
    uint160 sqrtPriceLimitX96; // 限定价格，值为0则不限价
}
```

代码如下：

```
/// @inheritdoc ISwapRouter
function exactOutputSingle(ExactOutputSingleParams calldata params)
    external
    payable
    override
    checkDeadline(params.deadline)
    returns (uint256 amountIn)
{
    // avoid an SLOAD by using the swap return data // 直接获取返回值
    amountIn = exactOutputInternal(
        params.amountOut,
        params.recipient,
        params.sqrtPriceLimitX96, // 允许用户自定义价格限制
        SwapCallbackData({path: abi.encodePacked(params.tokenOut, params.fee, params.tokenIn), payer: msg.sender}) // 现场编码单一路径
    );

    require(amountIn <= params.amountInMaximum, 'Too much requested'); 
    // has to be reset even though we don't use it in the single hop case
    amountInCached = DEFAULT_AMOUNT_IN_CACHED;
}
```

实际调用 [exactOutputInternal](https://github.com/Uniswap/v3-periphery/blob/main/contracts/SwapRouter.sol#L169)，代码如下：

```
/// @dev Performs a single exact output swap
function exactOutputInternal(
    uint256 amountOut, //期望获得的输出代币数量。
    address recipient,
    uint160 sqrtPriceLimitX96, //价格限制，用于防止过度滑点。
    SwapCallbackData memory data //包含路径和支付者信息的回调数据。
) private returns (uint256 amountIn) {
    // allow swapping to the router address with address 0
    if (recipient == address(0)) recipient = address(this);
	//这里解码出的顺序是tokenOut和tokenIn，与exactInputInternal中的顺序相反。这是因为exactOutput的路径是从输出代币到输入代币编码的。
    (address tokenOut, address tokenIn, uint24 fee) = data.path.decodeFirstPool();

    bool zeroForOne = tokenIn < tokenOut;

	//注意第三个参数是-amountOut.toInt256() exactInputInternal: 传入正的 amountIn exactOutputInternal: 传入负的 amountOut
    (int256 amount0Delta, int256 amount1Delta) =
        getPool(tokenIn, tokenOut, fee).swap(
            recipient,
            zeroForOne,
            -amountOut.toInt256(),
            sqrtPriceLimitX96 == 0
                ? (zeroForOne ? TickMath.MIN_SQRT_RATIO + 1 : TickMath.MAX_SQRT_RATIO - 1)
                : sqrtPriceLimitX96,
            abi.encode(data)
        );

    uint256 amountOutReceived;
    (amountIn, amountOutReceived) = zeroForOne
        ? (uint256(amount0Delta), uint256(-amount1Delta))
        : (uint256(amount1Delta), uint256(-amount0Delta));
    // it's technically possible to not receive the full output amount,
    // so if no price limit has been specified, require this possibility away
    if (sqrtPriceLimitX96 == 0) require(amountOutReceived == amountOut);
}
```

跟exactInputInternal的逻辑几乎完全一致，除了因为指定输出代币数量，调用交易池合约swap方法使用 -amountOut.toInt256() 作为参数。这是因为在exactOutput模式下，我们指定的是输出量，所以传入一个负值，表示我们希望池子付出指定数量的代币（即amountOut）

从池子返回的amount0Delta和amount1Delta中提取实际的输入量和输出量。注意，由于我们指定的是输出量，所以返回的输入量（amountIn）是池子实际收到的代币数量，而输出量（amountOutReceived）应该等于我们指定的amountOut（除非遇到价格限制）。

- 如果zeroForOne为true（用token0换token1），则：
  - 池子收到amount0Delta（正数）的token0，即输入量。
  - 池子付出amount1Delta（负数）的token1，我们取负数得到正数的输出量。
- 如果zeroForOne为false（用token1换token0），则：
  - 池子收到amount1Delta（正数）的token1，即输入量。
  - 池子付出amount0Delta（负数）的token0，我们取负数得到正数的输出量。

```
 uint256 amountOutReceived;
    (amountIn, amountOutReceived) = zeroForOne
        ? (uint256(amount0Delta), uint256(-amount1Delta))
        : (uint256(amount1Delta), uint256(-amount0Delta));
(int256 amount0Delta, int256 amount1Delta) =
    getPool(tokenIn, tokenOut, fee).swap(
        recipient,
        zeroForOne,
        -amountOut.toInt256(),
        sqrtPriceLimitX96 == 0
            ? (zeroForOne ? TickMath.MIN_SQRT_RATIO + 1 : TickMath.MAX_SQRT_RATIO - 1)
            : sqrtPriceLimitX96,
        abi.encode(data)
    );
```

返回的 amount0Delta 和 amount1Delta 为完成本次 swap 所需的 token0 数量和实际输出的 token1 数量，进一步判断 amountOut 满足最少输出代币数量的要求，完成 swap。

##### `swap`

一个通常的 V3 交易池存在很多互相重叠的价格区间的头寸，如下图所示：

图片加载失败: img/poolv3.png*poolv3*

每个交易池都会跟踪当前的价格，以及所有包含现价的价格区间提供的总流动性 liquidity。在每个区间的边界的 tick 上记录下 ΔL，当价格波动，穿过某个 tick 时，会根据价格波动方向进行流动性的增加或者减少。例如价格从左到右穿过区间，当穿过区间的第一个 tick 时，流动性需要增加 ΔL，穿出最后一个 tick 时，流动性需要减少 ΔL，中间的 tick 则流动性保持不变。

在一个 tick 内的流动性是常数， swap 公式如下：

Ptarget−Pcurrent=Δy/L*Pt**a**r**g**e**t*−*P**c**u**rre**n**t*=Δ*y*/*L*1/Ptarget−1/Pcurrent=Δx/L1/*Pt**a**r**g**e**t*−1/*P**c**u**rre**n**t*=Δ*x*/*L*

Pcurrent是 swap 前的价格， Ptarget是 swap 后的价格，L*L* 是 tick 内的流动性。

从上面公式，可以通过输入 token1 的数量 Δy推导出目标价格 Ptarget，进而推导出输出 token0 的数量 Δx；或者通过输入 token0 的数量 Δx推导出目标价格 Ptarget，进而推导出输出 token1 的数量 Δy。

如果是跨 tick 交易则需要拆解成多个 tick 内的交易：如果当前 tick 的流动性不能满足要求，价格会移动到当前区间的边界处。此时，使离开的区间休眠，并激活下一个区间。并且会开始下一个循环并且寻找下一个有流动性的 tick，直到用户需求的数量被满足。

讲完理论，回到代码。[swap](https://github.com/Uniswap/v3-core/blob/main/contracts/UniswapV3Pool.sol#L596) 方法是交易对 swap 最核心的方法，也是最复杂的方法。

参数为：

- recipient：接收者的地址；
- zeroForOne：如果从 token0 交换 token1 则为 true，从 token1 交换 token0 则为 false；
- amountSpecified：指定的代币数量，指定输入的代币数量则为正数，指定输出的代币数量则为负数；
- sqrtPriceLimitX96：限定价格，如果从 token0 交换 token1 则限定价格下限，从 token1 交换 token0 则限定价格上限；
- data：回调参数。

代码为：

```
/// @inheritdoc IUniswapV3PoolActions
function swap( 
    address recipient,
    bool zeroForOne,
    int256 amountSpecified, //指定的代币数量，指定输入的代币数量则为正数，指定输出的代币数量则为负数
    uint160 sqrtPriceLimitX96, //如果从 token0 交换 token1 则限定价格下限，从 token1 交换 token0 则限定价格上限
    bytes calldata data
) external override noDelegateCall returns (int256 amount0, int256 amount1) {
    require(amountSpecified != 0, 'AS');

    Slot0 memory slot0Start = slot0; //// 获取当前价格和tick状态

    require(slot0Start.unlocked, 'LOK'); // 确保池子未锁定（防重入）
    require(
     // 对于 zeroForOne 方向，价格限制必须低于当前价格但高于最小价格
    // 对于 !zeroForOne 方向，价格限制必须高于当前价格但低于最大价格
        zeroForOne
            ? sqrtPriceLimitX96 < slot0Start.sqrtPriceX96 && sqrtPriceLimitX96 > TickMath.MIN_SQRT_RATIO
            : sqrtPriceLimitX96 > slot0Start.sqrtPriceX96 && sqrtPriceLimitX96 < TickMath.MAX_SQRT_RATIO,
        'SPL'
    );

    slot0.unlocked = false; //// 锁定池子，防止重入

    SwapCache memory cache =
        SwapCache({
            liquidityStart: liquidity, // // 初始流动性
            blockTimestamp: _blockTimestamp(), 
            feeProtocol: zeroForOne ? (slot0Start.feeProtocol % 16) : (slot0Start.feeProtocol >> 4), // 协议费率
            secondsPerLiquidityCumulativeX128: 0, // 用于预言机的时间加权流动性累计值
            tickCumulative: 0,  // 用于预言机的tick累计值
            computedLatestObservation: false // 标记是否已计算最新观测值
        });

    bool exactInput = amountSpecified > 0; //判断是输入还是输出模式
    
    // 初始化交换状态，跟踪交换过程中的变化
    SwapState memory state =
        SwapState({
            amountSpecifiedRemaining: amountSpecified, // 剩余需要交换的数量
            amountCalculated: 0,  // 已计算出的数量
            sqrtPriceX96: slot0Start.sqrtPriceX96, // 当前价格
            tick: slot0Start.tick, //// 当前tick
            // 全局费用增长，根据方向选择 token0 或token1 的费用增长。
            feeGrowthGlobalX128: zeroForOne ? feeGrowthGlobal0X128 : feeGrowthGlobal1X128,  
            protocolFee: 0, // 累积的协议费用
            liquidity: cache.liquidityStart // 当前流动性
        });

    // 主循环：逐步处理交换，直到处理完所有数量或达到价格限制。
    while (state.amountSpecifiedRemaining != 0 && state.sqrtPriceX96 != sqrtPriceLimitX96) {
        StepComputations memory step;

        step.sqrtPriceStartX96 = state.sqrtPriceX96; //记录步骤开始时的价格。
		
		//查找下一个已初始化的 tick。tick 位图帮助高效找到有流动性的 tick。
        (step.tickNext, step.initialized) = tickBitmap.nextInitializedTickWithinOneWord(
            state.tick,
            tickSpacing,
            zeroForOne
        );
		
        //确保下一个 tick 在最小和最大 tick 范围内
        if (step.tickNext < TickMath.MIN_TICK) {
            step.tickNext = TickMath.MIN_TICK;
        } else if (step.tickNext > TickMath.MAX_TICK) {
            step.tickNext = TickMath.MAX_TICK;
        }

        // 计算下一个 tick 对应的价格
        step.sqrtPriceNextX96 = TickMath.getSqrtRatioAtTick(step.tickNext);

        // 调用 SwapMath.computeSwapStep 计算当前步骤的输入量、输出量、费用和新价格。
        // computeSwapStep 使用数学公式计算在恒定乘积曲线下如何交换。
        (state.sqrtPriceX96, step.amountIn, step.amountOut, step.feeAmount) = SwapMath.computeSwapStep(
            state.sqrtPriceX96,
            (zeroForOne ? step.sqrtPriceNextX96 < sqrtPriceLimitX96 : step.sqrtPriceNextX96 > sqrtPriceLimitX96)
                ? sqrtPriceLimitX96
                : step.sqrtPriceNextX96,
            state.liquidity,
            state.amountSpecifiedRemaining,
            fee
        );
		//根据精确输入或精确输出模式，更新剩余交换量和计算量。
		//精确输入: amountSpecifiedRemaining 减少（输入量 + 费用），amountCalculated 减少输出量（因为输出为负）。
        if (exactInput) {
            state.amountSpecifiedRemaining -= (step.amountIn + step.feeAmount).toInt256();
            state.amountCalculated = state.amountCalculated.sub(step.amountOut.toInt256());
        } else {
        //精确输出: amountSpecifiedRemaining 增加输出量（因为输出为负），amountCalculated 增加（输入量 + 费用）。
            state.amountSpecifiedRemaining += step.amountOut.toInt256();
            state.amountCalculated = state.amountCalculated.add((step.amountIn + step.feeAmount).toInt256());
        }

        // 处理协议费用: 如果协议费率大于零，计算协议费用并更新步骤费用和协议费用累积。
        if (cache.feeProtocol > 0) {
            uint256 delta = step.feeAmount / cache.feeProtocol; // 计算协议费用份额
            step.feeAmount -= delta;  // 减少交易费用，保留协议费用
            state.protocolFee += uint128(delta); // 累积协议费用
        }

        //更新全局费用增长: 根据步骤费用和流动性，计算费用增长并累加。
        if (state.liquidity > 0)
            state.feeGrowthGlobalX128 += FullMath.mulDiv(step.feeAmount, FixedPoint128.Q128, state.liquidity);

        // 如果价格达到下一个 tick 的价格，需要处理 tick 转换。
        if (state.sqrtPriceX96 == step.sqrtPriceNextX96) {
            // 如果 tick 已初始化，调用 ticks.cross 方法穿越该 tick 并获取流动性变化量 liquidityNet。
            if (step.initialized) {
                // check for the placeholder value, which we replace with the actual value the first time the swap
                 // 如果需要，计算最新预言机观测值（只计算一次）
                if (!cache.computedLatestObservation) {
                    (cache.tickCumulative, cache.secondsPerLiquidityCumulativeX128) = observations.observeSingle(
                        cache.blockTimestamp,
                        0,
                        slot0Start.tick,
                        slot0Start.observationIndex,
                        cache.liquidityStart,
                        slot0Start.observationCardinality
                    );
                    cache.computedLatestObservation = true;
                }
                 // 跨越tick：更新tick状态并获取流动性变化
                int128 liquidityNet =
                    ticks.cross(
                        step.tickNext,
                        (zeroForOne ? state.feeGrowthGlobalX128 : feeGrowthGlobal0X128),
                        (zeroForOne ? feeGrowthGlobal1X128 : state.feeGrowthGlobalX128),
                        cache.secondsPerLiquidityCumulativeX128,
                        cache.tickCumulative,
                        cache.blockTimestamp
                    );
                // 对于zeroForOne方向，流动性变化需要取反
                // safe because liquidityNet cannot be type(int128).min
                if (zeroForOne) liquidityNet = -liquidityNet;
				// 更新当前流动性
                state.liquidity = LiquidityMath.addDelta(state.liquidity, liquidityNet);
            }
			// 更新当前tick
            state.tick = zeroForOne ? step.tickNext - 1 : step.tickNext;
        } else if (state.sqrtPriceX96 != step.sqrtPriceStartX96) {
             // 如果价格变化但未到达下一个tick，重新计算tick (i.e. already transitioned ticks), and haven't moved
            state.tick = TickMath.getTickAtSqrtRatio(state.sqrtPriceX96);
        }
    }

   // 如果tick发生变化，需要更新预言机记录
    if (state.tick != slot0Start.tick) {
        (uint16 observationIndex, uint16 observationCardinality) =
         // 写入新的预言机观测值
            observations.write(
                slot0Start.observationIndex,
                cache.blockTimestamp,
                slot0Start.tick,
                cache.liquidityStart,
                slot0Start.observationCardinality,
                slot0Start.observationCardinalityNext
            );
            // 更新槽位状态
        (slot0.sqrtPriceX96, slot0.tick, slot0.observationIndex, slot0.observationCardinality) = (
            state.sqrtPriceX96,
            state.tick,
            observationIndex,
            observationCardinality
        );
    } else {
        // 如果tick未变化，只更新价格
        slot0.sqrtPriceX96 = state.sqrtPriceX96;
    }

    // 如果流动性发生变化，更新存储中的流动性值
    if (cache.liquidityStart != state.liquidity) liquidity = state.liquidity;

    // 更新全局费用增长和协议费用
    // overflow is acceptable, protocol has to withdraw before it hits type(uint128).max fees
    if (zeroForOne) {
        feeGrowthGlobal0X128 = state.feeGrowthGlobalX128;
        if (state.protocolFee > 0) protocolFees.token0 += state.protocolFee;
    } else {
        feeGrowthGlobal1X128 = state.feeGrowthGlobalX128;
        if (state.protocolFee > 0) protocolFees.token1 += state.protocolFee;
    }
	// 计算最终代币变化量
    (amount0, amount1) = zeroForOne == exactInput
        ? (amountSpecified - state.amountSpecifiedRemaining, state.amountCalculated)
        : (state.amountCalculated, amountSpecified - state.amountSpecifiedRemaining);

    // 执行代币转账和回调
    if (zeroForOne) {
        // 如果是token0 → token1，将token1转账给接收者
        if (amount1 < 0) TransferHelper.safeTransfer(token1, recipient, uint256(-amount1));
		// 记录当前余额，用于后续检查
        uint256 balance0Before = balance0();
         // 调用回调函数，要求调用者支付token0
        IUniswapV3SwapCallback(msg.sender).uniswapV3SwapCallback(amount0, amount1, data);
        // 检查余额变化，确保调用者支付了足够的token0
        require(balance0Before.add(uint256(amount0)) <= balance0(), 'IIA');
    } else {
     	// 如果是token1 → token0，将token0转账给接收者
        if (amount0 < 0) TransferHelper.safeTransfer(token0, recipient, uint256(-amount0));

        uint256 balance1Before = balance1();
        IUniswapV3SwapCallback(msg.sender).uniswapV3SwapCallback(amount0, amount1, data);
        // 检查余额变化，确保调用者支付了足够的token1
        require(balance1Before.add(uint256(amount1)) <= balance1(), 'IIA');
    }

    emit Swap(msg.sender, recipient, amount0, amount1, state.sqrtPriceX96, state.liquidity, state.tick);
    slot0.unlocked = true;
}
```

整体逻辑由一个 while 循环组成，将 swap 过程分解成多个小步骤，一点点的调整当前的 tick，直到满足用户所需的交易量或者价格触及限定价格（此时会部分成交）。

```
while (state.amountSpecifiedRemaining != 0 && state.sqrtPriceX96 != sqrtPriceLimitX96) {
```

使用 `tickBitmap.nextInitializedTickWithinOneWord`` 来找到下一个已初始化的 tick

```
(step.tickNext, step.initialized) = tickBitmap.nextInitializedTickWithinOneWord(
    state.tick,
    tickSpacing,
    zeroForOne
);
```

使用 SwapMath.computeSwapStep 进行 tick 内的 swap。这个方法会计算出当前区间可以满足的输入数量 amountIn，如果它比 amountRemaining 要小，我们会说现在的区间不能满足整个交易，因此下一个 sqrtPriceX96 就是当前区间的上界/下界，也就是说，我们消耗完了整个区间的流动性。如果 amountIn 大于 amountRemaining，我们计算的 sqrtPriceX96 仍然在现在区间内。

```
// compute values to swap to the target tick, price limit, or point where input/output amount is exhausted
(state.sqrtPriceX96, step.amountIn, step.amountOut, step.feeAmount) = SwapMath.computeSwapStep(
    state.sqrtPriceX96,
    (zeroForOne ? step.sqrtPriceNextX96 < sqrtPriceLimitX96 : step.sqrtPriceNextX96 > sqrtPriceLimitX96)
        ? sqrtPriceLimitX96
        : step.sqrtPriceNextX96,
    state.liquidity,
    state.amountSpecifiedRemaining,
    fee
);
```

保存本次交易的 amountIn 和 amountOut：

- 如果是指定输入代币数量。amountSpecifiedRemaining 表示剩余可用输入代币数量，amountCalculated 表示已输出代币数量（以负数表示）；
- 如果是指定输出代币数量。amountSpecifiedRemaining 表示剩余需要输出的代币数量（初始为负值，因此每次交换后需要加上 step.amountOut，直到为 0），amountCalculated 表示已使用的输入代币数量。

```
if (exactInput) {
    state.amountSpecifiedRemaining -= (step.amountIn + step.feeAmount).toInt256();
    state.amountCalculated = state.amountCalculated.sub(step.amountOut.toInt256());
} else {
    state.amountSpecifiedRemaining += step.amountOut.toInt256();
    state.amountCalculated = state.amountCalculated.add((step.amountIn + step.feeAmount).toInt256());
}
```

如果本次 swap 后的价格达到目标价格，如果该 tick 已经初始化，则通过 ticks.cross 方法穿越该 tick，返回新增的净流动性 liquidityNet 更新可用流动性 state.liquidity，移动当前 tick 到下一个 tick。

如果本次 swap 后的价格达到目标价格，但是又不等于初始价格，即表示此时 swap 结束，使用 swap 后的价格计算最新的 tick 值。

```
if (state.sqrtPriceX96 == step.sqrtPriceNextX96) {
    // if the tick is initialized, run the tick transition
    if (step.initialized) {
        // check for the placeholder value, which we replace with the actual value the first time the swap
        // crosses an initialized tick
        if (!cache.computedLatestObservation) {
            (cache.tickCumulative, cache.secondsPerLiquidityCumulativeX128) = observations.observeSingle(
                cache.blockTimestamp,
                0,
                slot0Start.tick,
                slot0Start.observationIndex,
                cache.liquidityStart,
                slot0Start.observationCardinality
            );
            cache.computedLatestObservation = true;
        }
        int128 liquidityNet =
            ticks.cross(
                step.tickNext,
                (zeroForOne ? state.feeGrowthGlobalX128 : feeGrowthGlobal0X128),
                (zeroForOne ? feeGrowthGlobal1X128 : state.feeGrowthGlobalX128),
                cache.secondsPerLiquidityCumulativeX128,
                cache.tickCumulative,
                cache.blockTimestamp
            );
        // if we're moving leftward, we interpret liquidityNet as the opposite sign
        // safe because liquidityNet cannot be type(int128).min
        if (zeroForOne) liquidityNet = -liquidityNet;

        state.liquidity = LiquidityMath.addDelta(state.liquidity, liquidityNet);
    }

    state.tick = zeroForOne ? step.tickNext - 1 : step.tickNext;
} else if (state.sqrtPriceX96 != step.sqrtPriceStartX96) {
    // recompute unless we're on a lower tick boundary (i.e. already transitioned ticks), and haven't moved
    state.tick = TickMath.getTickAtSqrtRatio(state.sqrtPriceX96);
}
```

重复上述步骤，直到 swap 完全结束。

完成 swap 后，更新 slot0 的状态和全局流动性。

```
// update tick and write an oracle entry if the tick change
if (state.tick != slot0Start.tick) {
    (uint16 observationIndex, uint16 observationCardinality) =
        observations.write(
            slot0Start.observationIndex,
            cache.blockTimestamp,
            slot0Start.tick,
            cache.liquidityStart,
            slot0Start.observationCardinality,
            slot0Start.observationCardinalityNext
        );
    (slot0.sqrtPriceX96, slot0.tick, slot0.observationIndex, slot0.observationCardinality) = (
        state.sqrtPriceX96,
        state.tick,
        observationIndex,
        observationCardinality
    );
} else {
    // otherwise just update the price
    slot0.sqrtPriceX96 = state.sqrtPriceX96;
}

// update liquidity if it changed
if (cache.liquidityStart != state.liquidity) liquidity = state.liquidity;
```

最后，计算本次 swap 需要的具体 amount0 和 amount1，调用 IUniswapV3SwapCallback接口。在回调之前已经把输出的 token 发送给了 recipient。

```
// do the transfers and collect payment
if (zeroForOne) {
    if (amount1 < 0) TransferHelper.safeTransfer(token1, recipient, uint256(-amount1));

    uint256 balance0Before = balance0();
    IUniswapV3SwapCallback(msg.sender).uniswapV3SwapCallback(amount0, amount1, data);
    require(balance0Before.add(uint256(amount0)) <= balance0(), 'IIA');
} else {
    if (amount0 < 0) TransferHelper.safeTransfer(token0, recipient, uint256(-amount0));

    uint256 balance1Before = balance1();
    IUniswapV3SwapCallback(msg.sender).uniswapV3SwapCallback(amount0, amount1, data);
    require(balance1Before.add(uint256(amount1)) <= balance1(), 'IIA');
}
```

IUniswapV3SwapCallback 的实现在 periphery 仓库的 SwapRouter.sol 中，负责支付输入的 token。

```
/// @inheritdoc IUniswapV3SwapCallback
function uniswapV3SwapCallback(
    int256 amount0Delta,
    int256 amount1Delta,
    bytes calldata _data
) external override {
    require(amount0Delta > 0 || amount1Delta > 0); // swaps entirely within 0-liquidity regions are not supported
    SwapCallbackData memory data = abi.decode(_data, (SwapCallbackData));
    (address tokenIn, address tokenOut, uint24 fee) = data.path.decodeFirstPool();
    CallbackValidation.verifyCallback(factory, tokenIn, tokenOut, fee);

    (bool isExactInput, uint256 amountToPay) =
        amount0Delta > 0
            ? (tokenIn < tokenOut, uint256(amount0Delta))
            : (tokenOut < tokenIn, uint256(amount1Delta));
    if (isExactInput) {
        pay(tokenIn, data.payer, msg.sender, amountToPay);
    } else {
        // either initiate the next swap or pay
        if (data.path.hasMultiplePools()) {
            data.path = data.path.skipToken();
            exactOutputInternal(amountToPay, msg.sender, 0, data);
        } else {
            amountInCached = amountToPay;
            tokenIn = tokenOut; // swap in/out because exact output swaps are reversed
            pay(tokenIn, data.payer, msg.sender, amountToPay);
        }
    }
}
```

至此，完成了整体 swap 流程。

#### 计算流动性和资产金额

白皮书中似乎提供了一种计算L、x和y的简单方法：

L=xvirtual⋅P=yvirtualP*L*=*x**v**i**r**t**u**a**l*⋅*P*=*P**y**v**i**r**t**u**a**l*

**Table 1: 词汇说明**

| Symbol Name                  | Whitepaper | Uniswap code     | Notes                                   |
| :--------------------------- | :--------- | :--------------- | :-------------------------------------- |
| Pice                         | P          | sqrtRatioX96     | Code tracks P*P* for efficiency reasons |
| Lower bound of a price range | pa*p**a*   | sqrtRatioAX96    | Code tracks pa*p**a*                    |
| Upper bound of a price range | pb*p**b*   | sqrtRatioBX96    | Code tracks pb*p**b*                    |
| The first asset              | X          | token0           |                                         |
| The second asset             | Y          | token1           |                                         |
| Amount of the first asset    | x*x*       | amount0          |                                         |
| Amount of the second asset   | y*y*       | amount1          |                                         |
| Virtual liquidity            | L          | liquidity amount |                                         |

然而，这里的x和y是虚拟代币数量，而不是真实数量！计算x和y的真实数量的数学公式在白皮书的最后给出，具体是在公式6.29和6.30中。这些数学公式的实现可以在文件LiquidityAmounts.sol中找到。

这些方程可以从白皮书中的关键方程2.2中推导出：

(xreal+LPb)(yreal+Lpa)=L2(*x**re**a**l*+*P**b**L*)(*y**re**a**l*+*L**p**a*)=*L*2

尝试直接解方程2.2以得到L会得到一个非常混乱的结果。相反，我们可以注意到，在价格范围之外，流动性完全由单个资产提供，要么是X，要么是Y，具体取决于当前价格在价格范围的哪一侧。我们有三个选择：

1. 假设P≤pa，则头寸完全在X中，因此y=0：

(x+Lpb)Lpa=L2(1)(*x*+*p**b**L*)*L**p**a*=*L*2(1)xpa+Lpapb=L(2)*x**p**a*+*L**p**b**p**a*=*L*(2)x=Lpa−Lpb(3)*x*=*p**a**L*−*p**b**L*(3)x=Lpb−papa⋅pb(4)*x*=*L**p**a*⋅*p**b**p**b*−*p**a*(4)

该头寸的流动性为：

L=xpa⋅pbpb−pa(5)*L*=*x**p**b*−*p**a**p**a*⋅*p**b*(5)

1. 假设P≥pb，则头寸完全在Y中，因此x=0：

Lpb(y+Lpa)=L2(6)*p**b**L*(*y*+*L**p**a*)=*L*2(6)ypb+Lpapb=L(7)*p**b**y*+*L**p**b**p**a*=*L*(7)y=L(pb−pa)(8)*y*=*L*(*p**b*−*p**a*)(8)

该头寸的流动性为：

L=ypb−pa(9)*L*=*p**b*−*p**a**y*(9)

3.当前价格在范围内：pa < P < pb。我认为应该这样考虑，即在最佳头寸，两种资产将平等地为流动性做出贡献。也就是说，在价格范围（P，pb）的一侧，资产x提供的流动性Lx必须等于在价格范围（pa，P）的另一侧资产y提供的流动性Ly。根据方程5和9，我们知道如何计算单资产范围的流动性。当P位于范围（pa，pb）内时，我们可以将（P，pb）视为X提供流动性的子范围，将（pa，P）视为Y提供流动性的子范围。将这些代入方程5和9，并要求Lx(P, pb) = Ly(pa, P)，我们得到：

xP⋅pbpb−P=yP−pa(10)*x**p**b*−*P**P*⋅*p**b*=*P*−*p**a**y*(10)

方程（第10式）很重要，因为它可以解出五个变量中的任意一个，包括x、y、P、pa、pb，而不需要涉及到流动性。然而，对于x和y，这是不必要的；等式4和8的简单修改就足够了：

x=Lpb−PP⋅pb(11)*x*=*L**P*⋅*p**b**p**b*−*P*(11)y=L(P−pa)(12)*y*=*L*(*P*−*p**a*)(12)

综上所述：

- 如果P≤pa，y=0，x可通过等式4计算。
- 如果P≥pb，则x=0，y可通过公式8计算。
- 否则，pa<P<pb，x和y可分别通过等式11和12计算。

从概念上讲，这一结果只是以略微不同的形式重述了白皮书中的等式6.29和6.30。然而，白皮书中∆的使用可能会让新用户感到困惑——确切地说，Δ是什么？如果是全新的头寸怎么办？出于这个原因，上面的方程4-12避免提及delta，目的是为了简单。当然，从更深入的角度来看，白皮书方程6.29和6.30仍然可以应用于新头寸：在这种情况下，我们只需要取∆L=L。
