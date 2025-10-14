# 🎉 第30天 – 最终构建:你自己的迷你DEX

**你做到了。**

整整三十天深入Solidity。一次一个合约。一次一个概念。从你的第一个 `uint` 到你的第一个代币销售,你一直在坚持、构建、学习——现在?你已经准备好创建驱动一切的东西,从DeFi巨鲸到收益农场狂热者:

一个 **去中心化交易所** —— 你自己的 **迷你DEX**。

### 🧡 让我们花点时间欣赏这一点

认真地。回顾第1天。

你从基本存储、结构体和函数开始。你学习了全局变量、控制流、修饰符、继承和接口。你动手实践了NFT、DAO、AMM、稳定币、预言机、随机性、可升级合约等等。

你不仅仅是阅读它们——

**你构建了它们。**

### 🚀 迷你去中心化交易所

不是模拟。不是复制粘贴的Uniswap克隆。我们说的是一个干净、简单、精简的 **从零开始实现** 的DEX——刚好足够理解交换、流动性和LP代币的真正工作原理。

为了让这一切点击,我们将这最后的挑战分成 **两个合约**:

---

## 📦 核心池:`MiniDexPair.sol`

### 完整合约代码

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "@openzeppelin/contracts/security/ReentrancyGuard.sol";

contract MiniDexPair is ReentrancyGuard {
    address public immutable tokenA;
    address public immutable tokenB;

    uint256 public reserveA;
    uint256 public reserveB;
    uint256 public totalLPSupply;

    mapping(address => uint256) public lpBalances;

    event LiquidityAdded(address indexed provider, uint256 amountA, uint256 amountB, uint256 lpMinted);
    event LiquidityRemoved(address indexed provider, uint256 amountA, uint256 amountB, uint256 lpBurned);
    event Swapped(address indexed user, address inputToken, uint256 inputAmount, address outputToken, uint256 outputAmount);

    constructor(address _tokenA, address _tokenB) {
        require(_tokenA != _tokenB, "Identical tokens");
        require(_tokenA != address(0) && _tokenB != address(0), "Zero address");

        tokenA = _tokenA;
        tokenB = _tokenB;
    }

    // 工具函数
    function sqrt(uint y) internal pure returns (uint z) {
        if (y > 3) {
            z = y;
            uint x = y / 2 + 1;
            while (x < z) {
                z = x;
                x = (y / x + x) / 2;
            }
        } else if (y != 0) {
            z = 1;
        }
    }

    function min(uint256 a, uint256 b) internal pure returns (uint256) {
        return a < b ? a : b;
    }

    function _updateReserves() private {
        reserveA = IERC20(tokenA).balanceOf(address(this));
        reserveB = IERC20(tokenB).balanceOf(address(this));
    }

    function addLiquidity(uint256 amountA, uint256 amountB) external nonReentrant {
        require(amountA > 0 && amountB > 0, "Invalid amounts");

        IERC20(tokenA).transferFrom(msg.sender, address(this), amountA);
        IERC20(tokenB).transferFrom(msg.sender, address(this), amountB);

        uint256 lpToMint;
        if (totalLPSupply == 0) {
            lpToMint = sqrt(amountA * amountB);
        } else {
            lpToMint = min(
                (amountA * totalLPSupply) / reserveA,
                (amountB * totalLPSupply) / reserveB
            );
        }

        require(lpToMint > 0, "Zero LP minted");

        lpBalances[msg.sender] += lpToMint;
        totalLPSupply += lpToMint;

        _updateReserves();

        emit LiquidityAdded(msg.sender, amountA, amountB, lpToMint);
    }

    function removeLiquidity(uint256 lpAmount) external nonReentrant {
        require(lpAmount > 0 && lpAmount <= lpBalances[msg.sender], "Invalid LP amount");

        uint256 amountA = (lpAmount * reserveA) / totalLPSupply;
        uint256 amountB = (lpAmount * reserveB) / totalLPSupply;

        lpBalances[msg.sender] -= lpAmount;
        totalLPSupply -= lpAmount;

        IERC20(tokenA).transfer(msg.sender, amountA);
        IERC20(tokenB).transfer(msg.sender, amountB);

        _updateReserves();

        emit LiquidityRemoved(msg.sender, amountA, amountB, lpAmount);
    }

    function getAmountOut(uint256 inputAmount, address inputToken) public view returns (uint256 outputAmount) {
        require(inputToken == tokenA || inputToken == tokenB, "Invalid input token");

        bool isTokenA = inputToken == tokenA;
        (uint256 inputReserve, uint256 outputReserve) = isTokenA ? (reserveA, reserveB) : (reserveB, reserveA);

        uint256 inputWithFee = inputAmount * 997;
        uint256 numerator = inputWithFee * outputReserve;
        uint256 denominator = (inputReserve * 1000) + inputWithFee;

        outputAmount = numerator / denominator;
    }

    function swap(uint256 inputAmount, address inputToken) external nonReentrant {
        require(inputAmount > 0, "Zero input");
        require(inputToken == tokenA || inputToken == tokenB, "Invalid token");

        address outputToken = inputToken == tokenA ? tokenB : tokenA;
        uint256 outputAmount = getAmountOut(inputAmount, inputToken);

        require(outputAmount > 0, "Insufficient output");

        IERC20(inputToken).transferFrom(msg.sender, address(this), inputAmount);
        IERC20(outputToken).transfer(msg.sender, outputAmount);

        _updateReserves();

        emit Swapped(msg.sender, inputToken, inputAmount, outputToken, outputAmount);
    }

    // 查看函数
    function getReserves() external view returns (uint256, uint256) {
        return (reserveA, reserveB);
    }

    function getLPBalance(address user) external view returns (uint256) {
        return lpBalances[user];
    }

    function getTotalLPSupply() external view returns (uint256) {
        return totalLPSupply;
    }
}
```

## 🏗️ 池创建者:`MiniDexFactory.sol`

### 完整合约代码

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts/access/Ownable.sol";
import "./MiniDexPair.sol";

contract MiniDexFactory is Ownable {
    event PairCreated(address indexed tokenA, address indexed tokenB, address pairAddress, uint);

    mapping(address => mapping(address => address)) public getPair;
    address[] public allPairs;

    constructor(address _owner) Ownable(_owner) {}

    function createPair(address _tokenA, address _tokenB) external onlyOwner returns (address pair) {
        require(_tokenA != address(0) && _tokenB != address(0), "Invalid token address");
        require(_tokenA != _tokenB, "Identical tokens");
        require(getPair[_tokenA][_tokenB] == address(0), "Pair already exists");

        // 排序代币以保持一致性
        (address token0, address token1) = _tokenA < _tokenB ? (_tokenA, _tokenB) : (_tokenB, _tokenA);

        pair = address(new MiniDexPair(token0, token1));
        getPair[token0][token1] = pair;
        getPair[token1][token0] = pair;

        allPairs.push(pair);
        emit PairCreated(token0, token1, pair, allPairs.length - 1);
    }

    function allPairsLength() external view returns (uint) {
        return allPairs.length;
    }

    function getPairAtIndex(uint index) external view returns (address) {
        require(index < allPairs.length, "Index out of bounds");
        return allPairs[index];
    }
}
```

## 核心知识点

### MiniDexPair关键特性

1. **不可变代币**: 一旦设置,池中的代币永远不能改变
2. **LP代币**: 内部追踪,代表所有权份额
3. **常数乘积公式**: `x * y = k` 用于定价
4. **0.3%费用**: 与Uniswap V2相同
5. **重入保护**: 所有外部函数受保护

### MiniDexFactory关键特性

1. **动态池创建**: 按需创建新交易对
2. **双向映射**: `getPair[A][B]` 和 `getPair[B][A]` 都工作
3. **防重复**: 每对只能创建一次
4. **代币排序**: 确保 `tokenA < tokenB`
5. **所有者控制**: 只有所有者可以创建池

### 工作流程

#### 1. 部署
```solidity
1. 部署两个ERC-20代币(TokenA, TokenB)
2. 部署 MiniDexFactory
3. 调用 factory.createPair(TokenA, TokenB)
```

#### 2. 添加流动性
```solidity
1. 批准 pair 合约消费你的代币
2. 调用 pair.addLiquidity(amountA, amountB)
3. 接收 LP 代币
```

#### 3. 交换
```solidity
1. 批准 pair 合约
2. 调用 pair.swap(inputAmount, inputToken)
3. 接收输出代币
```

#### 4. 移除流动性
```solidity
1. 调用 pair.removeLiquidity(lpAmount)
2. 接收两个代币back
```

## 🎉 总结

你现在拥有了一个完全功能的DEX!

- ✅ 支持ERC-20代币
- ✅ 让用户交换、添加和移除流动性
- ✅ 使用常数乘积AMM模型计算价格
- ✅ 发出清晰的事件并公开有用的查看函数
- ✅ 工厂模式用于可扩展性

**恭喜完成30天Solidity之旅!** 🎊

你已经从基础到高级DeFi,现在你有了构建真实Web3应用的技能。

