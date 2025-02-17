# UniswapV3 合约导读

> 本文旨在帮助大家熟悉 UniswapV3 的合约结构，梳理流程。以下内容主要参考自 [@paco0x](https://github.com/paco0x) 的系列博客[《Uniswap v3 详解》](https://liaoph.com/uniswap-v3-1/)。感谢 paco 的精彩分享，强烈推荐大家去读一读他的博客！

## 基本架构

Uniswap v3 在代码层面的架构和 v2 基本保持一致，将合约分成了两个仓库：

- uniswap-v3-core
- uniswap-v3-periphery

### core

- **UniswapV3Factory**: 提供创建 pool 的接口，并且追踪所有的 pool
- **UniswapV3Pool**: 实现代币交易，流动性管理，交易手续费的收取，oracle 数据管理。接口的实现粒度比较低，不适合普通用户使用，错误的调用其中的接口可能会造成经济上的损失。

### periphery

- **SwapRouter**: 提供代币交易的接口，它是对 UniswapV3Pool 合约中交易相关接口的进一步封装，前端界面主要与这个合约来进行对接。
- **NonfungiblePositionManager**: 用来增加/移除/修改 Pool 的流动性，并且通过 NFT token 将流动性代币化。使用 ERC721 token（v2 使用的是 ERC20）的原因是同一个池的多个流动性并不能等价替换（v3 的集中流性动功能）。

### 相关图示

- 合约关系图
  ![合约关系图](./img/contracts-relationship.webp)
- [合约结构图](../img/640.png)
- [Factory 结构图](./img/UniswapV3_ContractMap_Factory.png)
- [NonFungiblePositionManager 结构图](./img/UniswapV3_ContractMap_NonFungiblePositionManager.png)
- [Pool 结构图](./img/UniswapV3_ContractMap_Pool.png)

## 流程梳理

### CreatePool

![创建交易对流程图](./img/create-pool.png)

用户首先调用 `NonfungiblePositionManager` 合约的 `createAndInitializePoolIfNecessary` 方法创建交易对，传入的参数为交易对的 token0, token1, fee 和初始价格 sqrtPrice.

- 调用`Factory.getPool(tokenA, tokenB, fee)`获取 Pool 地址

- 如果 Pool 地址为 0，说明 Pool 还未创建

  - 调用`Factory.createPool(tokenA, tokenB, fee)`，创建 Pool
    - Factory调用 `Pool.deploy` 部署Pool合约
  - 调用`Pool.initialize(sqrtPriceX96)`对 Pool 初始化

- 如果 Pool 地址不为 0 ，说明 Pool 已存在

  - 检查 Pool 的价格，若为 0,调用`Pool.initialize(sqrtPriceX96)`对 Pool 初始化

相关代码

- [createAndInitializePoolIfNecessary](./NonfungiblePositionManager.md#createAndInitializePoolIfNecessary)
- [UniswapV3Factory.getPool](./UniswapV3Factory.md#getPool)
- [UniswapV3Factory.createPool](./UniswapV3Factory.md#createPool)
- [UniswapV3Factory.deploy](./UniswapV3Factory.md#deploy)
- [UniswapV3Pool.initialize](./UniswapV3Pool.md#initialize)

### addLiquidity

在合约内，v3 会保存所有用户的流动性，代码内称作 Position，提供流动性的调用流程如下：

![添加流动性对流程图](./img/add-liquidity.png)

用户调用 `Manager.mint`添加流动性，其内部流程如下：

- Manager内部调用 `Manager.addLiquidity`
- Manager调用`Pool.mint`
  - 修改用户的position状态
  - 调用manager的mint回调函数，进行token的转帐操作
- Manager内部调用`Manager.mint`
  - 将代表相关流动性postion的ERC721代币返回给用户
  - 创建流动性头寸存入Manager
  - 广播 `IncreaseLiquidity(tokenId, liquidity, amount0, amount1)`

相关代码

- [Pool.mint](./UniswapV3Pool.md#mint)
- [struct AddLiquidityParams](./NonfungiblePositionManager.md#AddLiquidityParams)
- [Manager.addLiquidity](./NonfungiblePositionManager.md#addLiquidity)
- [Manager.mint](./NonfungiblePositionManager.md#mint)
