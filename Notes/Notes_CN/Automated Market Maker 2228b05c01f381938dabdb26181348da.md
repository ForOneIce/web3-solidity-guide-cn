# 自动化做市商(AMM)

欢迎回到 **Solidity 30天**——每一天,我们将一个复杂的概念转化为你能真正理解的代码。

今天,我们将揭开DeFi中最传奇机制之一的神秘面纱:

**自动化做市商** —— 或称AMM。

这是 **DeFi中最强大的发明之一** —— 这种东西帮助将智能合约变成了成熟的金融应用。

## 🏛️ 从前:人们如何交易加密货币

让我们回到交易最初的运作方式——即使在加密货币早期:

就像股票市场一样。

- 你有 **买家**,他们说:
  *"我想买1 DAI,我愿意支付0.95 ETH。"*

- 你有 **卖家**,他们说:
  *"我有1 DAI要卖,但我想要1 ETH。"*

- 中间有一个叫做 **订单簿** 的系统。
  想象它像一个巨大的列表,匹配买家和卖家。

当买家和卖家都同意一个价格时?砰——交易发生了。

这个系统在像Binance或Coinbase这样的中心化交易所上运行良好。

但有个问题...

## 🤕 为什么这在链上不起作用

在区块链上,一切都要花费 **gas**。

即使是像"我想买DAI"这样简单的事情也意味着:

- 发送交易
- 支付费用
- 等待确认

想象一下,必须为每一次对买入或卖出价格的微小改变都这样做。那是噩梦。

如果那时没有人想和你交易怎么办?

你被卡住了。你的订单就在那里,浪费gas和时间。

所以这是核心问题:

- 订单簿是 **中心化的**
- 它们需要人们一直保持活跃
- 在链上使用缓慢且昂贵

DeFi需要 **更好的** 东西。

## 💡 突破:如果我们不需要买家和卖家呢?

这是改变一切的绝妙想法:

> 如果人们不需要等待交易伙伴呢?
> 
> 如果智能合约就像代币的自动贩卖机呢?

所以,不依赖其他人来交易...

你会有两个代币的池子——比如DAI和ETH——锁定在任何人都可以使用的合约中。

想用DAI **购买ETH**?

合约进行数学计算,给你ETH,并保留你的DAI。

想用ETH **兑换DAI**?

同样的事情——合约给你DAI并存储你的ETH。

不需要买家。不需要卖家。只需要数学。

## 📐 好的...但它实际上是如何工作的?

AMM使用一个简单的公式:

> x × y = k

让我们分解一下。

- `x` = 池中代币A的数量
- `y` = 池中代币B的数量
- `k` = 某个常数(它永远不会改变)

所以 **两个代币储备的乘积必须始终保持不变**。

这是魔法所在:

如果有人想添加更多的代币A(如ETH),

保持 `k` 不变的唯一方法是合约给他们 *更少* 的代币B(如DAI)。

这就是价格如何自动调整——**自动地**。

随着人们交易更多,比率会改变——价格会自行更新。

这就是为什么它被称为 **自动化做市商**。

合约使用数学 *制造市场*,而不是人工订单。

## 🔁 你可以用AMM做什么?

你可以做三件大事:

1. **交换** 代币
   - 即时将代币A换成代币B(或反过来)
   - 不需要等待匹配
   - 只需发送代币,获得另一个

2. **添加流动性**
   - 存入等值的代币A和B
   - 你获得 **LP代币**(像收据)
   - 当你的代币在池中时,你赚取交易费用的一部分

3. **移除流动性**
   - 返还你的LP代币
   - 获得你两个代币的份额

现在你明白了这个想法...

在下一节中,我们将看一个实际的Solidity合约——并逐步了解如何 **从头编写你自己的AMM**。

## 🛠️ 你今天要构建什么

好的,所以今天的项目不仅仅是一个玩具合约。

你正在构建 **驱动去中心化交易中数十亿美元的核心引擎**。

但这里有个转折:

- **没有库**
- **没有捷径**
- **没有幕后魔法**

你将要手工编写自动化做市商的核心——这是 *真正* 理解它如何工作的最佳方式。

## 合约详解

### 导入ERC20标准
```solidity
import "@openzeppelin/contracts/token/ERC20/ERC20.sol";
```

### 合约声明
```solidity
contract AutomatedMarketMaker is ERC20 {
    IERC20 public tokenA;
    IERC20 public tokenB;
    
    uint256 public reserveA;
    uint256 public reserveB;
    
    address public owner;
}
```

### 事件
```solidity
event LiquidityAdded(address indexed provider, uint256 amountA, uint256 amountB, uint256 liquidity);
event LiquidityRemoved(address indexed provider, uint256 amountA, uint256 amountB, uint256 liquidity);
event TokensSwapped(address indexed trader, address tokenIn, uint256 amountIn, address tokenOut, uint256 amountOut);
```

### 构造函数
```solidity
constructor(address _tokenA, address _tokenB, string memory _name, string memory _symbol) 
    ERC20(_name, _symbol) 
{
    tokenA = IERC20(_tokenA);
    tokenB = IERC20(_tokenB);
    owner = msg.sender;
}
```

### 添加流动性
```solidity
function addLiquidity(uint256 amountA, uint256 amountB) external {
    require(amountA > 0 && amountB > 0, "Amounts must be > 0");
    
    tokenA.transferFrom(msg.sender, address(this), amountA);
    tokenB.transferFrom(msg.sender, address(this), amountB);
    
    uint256 liquidity;
    if (totalSupply() == 0) {
        liquidity = sqrt(amountA * amountB);
    } else {
        liquidity = min(
            amountA * totalSupply() / reserveA,
            amountB * totalSupply() / reserveB
        );
    }
    
    _mint(msg.sender, liquidity);
    
    reserveA += amountA;
    reserveB += amountB;
    
    emit LiquidityAdded(msg.sender, amountA, amountB, liquidity);
}
```

### 移除流动性
```solidity
function removeLiquidity(uint256 liquidityToRemove) external returns (uint256 amountAOut, uint256 amountBOut) {
    require(liquidityToRemove > 0, "Liquidity to remove must be > 0");
    require(balanceOf(msg.sender) >= liquidityToRemove, "Insufficient liquidity tokens");
    
    uint256 totalLiquidity = totalSupply();
    require(totalLiquidity > 0, "No liquidity in the pool");
    
    amountAOut = liquidityToRemove * reserveA / totalLiquidity;
    amountBOut = liquidityToRemove * reserveB / totalLiquidity;
    
    require(amountAOut > 0 && amountBOut > 0, "Insufficient reserves for requested liquidity");
    
    reserveA -= amountAOut;
    reserveB -= amountBOut;
    
    _burn(msg.sender, liquidityToRemove);
    
    tokenA.transfer(msg.sender, amountAOut);
    tokenB.transfer(msg.sender, amountBOut);
    
    emit LiquidityRemoved(msg.sender, amountAOut, amountBOut, liquidityToRemove);
    return (amountAOut, amountBOut);
}
```

### 交换A换B
```solidity
function swapAforB(uint256 amountAIn, uint256 minBOut) external {
    require(amountAIn > 0, "Amount must be > 0");
    require(reserveA > 0 && reserveB > 0, "Insufficient reserves");
    
    uint256 amountAInWithFee = amountAIn * 997 / 1000;
    uint256 amountBOut = reserveB * amountAInWithFee / (reserveA + amountAInWithFee);
    
    require(amountBOut >= minBOut, "Slippage too high");
    
    tokenA.transferFrom(msg.sender, address(this), amountAIn);
    tokenB.transfer(msg.sender, amountBOut);
    
    reserveA += amountAInWithFee;
    reserveB -= amountBOut;
    
    emit TokensSwapped(msg.sender, address(tokenA), amountAIn, address(tokenB), amountBOut);
}
```

### 交换B换A
```solidity
function swapBforA(uint256 amountBIn, uint256 minAOut) external {
    require(amountBIn > 0, "Amount must be > 0");
    require(reserveA > 0 && reserveB > 0, "Insufficient reserves");
    
    uint256 amountBInWithFee = amountBIn * 997 / 1000;
    uint256 amountAOut = reserveA * amountBInWithFee / (reserveB + amountBInWithFee);
    
    require(amountAOut >= minAOut, "Slippage too high");
    
    tokenB.transferFrom(msg.sender, address(this), amountBIn);
    tokenA.transfer(msg.sender, amountAOut);
    
    reserveB += amountBInWithFee;
    reserveA -= amountAOut;
    
    emit TokensSwapped(msg.sender, address(tokenB), amountBIn, address(tokenA), amountAOut);
}
```

### 工具函数
```solidity
function getReserves() external view returns (uint256, uint256) {
    return (reserveA, reserveB);
}

function min(uint256 a, uint256 b) internal pure returns (uint256) {
    return a < b ? a : b;
}

function sqrt(uint256 y) internal pure returns (uint256 z) {
    if (y > 3) {
        z = y;
        uint256 x = y / 2 + 1;
        while (x < z) {
            z = x;
            x = (y / x + x) / 2;
        }
    } else if (y != 0) {
        z = 1;
    }
}
```

## 核心知识点

### 常数乘积公式
```
x * y = k
```
这确保:
- 你交换得越多,价格变动越大(滑点)
- 池子永远不会完全耗尽
- 交换总是使 `x * y` 大致相同

### LP代币
- 代表你在池中的所有权份额
- 如果你拥有10%的LP,你拥有10%的代币
- 可以随时赎回你的份额

### 交易费用
0.3%的费用应用于每笔交易:
```solidity
uint256 amountAInWithFee = amountAIn * 997 / 1000;
```

### 滑点保护
```solidity
require(amountBOut >= minBOut, "Slippage too high");
```
保护用户免受意外价格变动的影响。

## 🎉 总结

你刚刚构建了一个DEX引擎!

现在你了解了:
- AMM如何使用数学而不是订单簿
- 流动性池如何工作
- LP代币为何重要
- 价格发现、滑点和费用如何融入数学中

这不仅仅是代码——这是真正理解DeFi运作方式的技能。

