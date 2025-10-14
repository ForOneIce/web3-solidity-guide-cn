# 稳定币合约

本教程构建了一个由抵押品支持的 `SimpleStablecoin` 合约。用户可以通过存入抵押品(ERC-20代币)基于Chainlink价格馈送和抵押率铸造sUSD。他们也可以赎回sUSD换取抵押品。使用了 `ERC20`、`Ownable`、`ReentrancyGuard`、`SafeERC20`、`AccessControl`、`IERC20Metadata` 和 `AggregatorV3Interface`。

## 关键概念

### 导入
```solidity
import "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import "@openzeppelin/contracts/access/Ownable.sol";
import "@openzeppelin/contracts/security/ReentrancyGuard.sol";
import "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";
import "@openzeppelin/contracts/access/AccessControl.sol";
import "@openzeppelin/contracts/token/ERC20/extensions/IERC20Metadata.sol";
import "@chainlink/contracts/src/v0.8/interfaces/AggregatorV3Interface.sol";
```

### 合约定义
```solidity
contract SimpleStablecoin is ERC20, Ownable, ReentrancyGuard, AccessControl {
    using SafeERC20 for IERC20;
    
    bytes32 public constant PRICE_FEED_MANAGER_ROLE = keccak256("PRICE_FEED_MANAGER_ROLE");
    
    IERC20 public collateralToken;
    uint8 public collateralDecimals;
    AggregatorV3Interface public priceFeed;
    uint256 public collateralizationRatio;  // 以基点表示 (150% = 15000)
}
```

### 事件
```solidity
event Minted(address indexed user, uint256 stablecoinAmount, uint256 collateralAmount);
event Redeemed(address indexed user, uint256 stablecoinAmount, uint256 collateralAmount);
event PriceFeedUpdated(address indexed newPriceFeed);
event CollateralizationRatioUpdated(uint256 newRatio);
```

### 自定义错误
```solidity
error InvalidCollateralTokenAddress();
error InvalidPriceFeedAddress();
error MintAmountIsZero();
error InsufficientStablecoinBalance();
error CollateralizationRatioTooLow();
```

### 构造函数
```solidity
constructor(
    address _collateralToken,
    address _priceFeed,
    uint256 _collateralizationRatio
) ERC20("Simple USD", "sUSD") Ownable(msg.sender) {
    if (_collateralToken == address(0)) revert InvalidCollateralTokenAddress();
    if (_priceFeed == address(0)) revert InvalidPriceFeedAddress();
    if (_collateralizationRatio < 10000) revert CollateralizationRatioTooLow();
    
    collateralToken = IERC20(_collateralToken);
    collateralDecimals = IERC20Metadata(_collateralToken).decimals();
    priceFeed = AggregatorV3Interface(_priceFeed);
    collateralizationRatio = _collateralizationRatio;
    
    _grantRole(DEFAULT_ADMIN_ROLE, msg.sender);
    _grantRole(PRICE_FEED_MANAGER_ROLE, msg.sender);
}
```

### 获取当前价格
```solidity
function getCurrentPrice() public view returns (uint256) {
    (, int256 price, , , ) = priceFeed.latestRoundData();
    require(price > 0, "Invalid price");
    
    uint8 priceFeedDecimals = priceFeed.decimals();
    
    // 归一化为18位小数
    if (priceFeedDecimals < 18) {
        return uint256(price) * (10 ** (18 - priceFeedDecimals));
    } else if (priceFeedDecimals > 18) {
        return uint256(price) / (10 ** (priceFeedDecimals - 18));
    }
    return uint256(price);
}
```

### 铸造稳定币
```solidity
function mint(uint256 stablecoinAmount) external nonReentrant {
    if (stablecoinAmount == 0) revert MintAmountIsZero();
    
    uint256 requiredCollateral = getRequiredCollateralForMint(stablecoinAmount);
    
    collateralToken.safeTransferFrom(msg.sender, address(this), requiredCollateral);
    
    _mint(msg.sender, stablecoinAmount);
    
    emit Minted(msg.sender, stablecoinAmount, requiredCollateral);
}
```

### 赎回稳定币
```solidity
function redeem(uint256 stablecoinAmount) external nonReentrant {
    if (balanceOf(msg.sender) < stablecoinAmount) {
        revert InsufficientStablecoinBalance();
    }
    
    uint256 collateralToReturn = getCollateralForRedeem(stablecoinAmount);
    
    _burn(msg.sender, stablecoinAmount);
    
    collateralToken.safeTransfer(msg.sender, collateralToReturn);
    
    emit Redeemed(msg.sender, stablecoinAmount, collateralToReturn);
}
```

### 设置抵押率
```solidity
function setCollateralizationRatio(uint256 _newRatio) external onlyOwner {
    if (_newRatio < 10000) revert CollateralizationRatioTooLow();
    collateralizationRatio = _newRatio;
    emit CollateralizationRatioUpdated(_newRatio);
}
```

### 设置价格馈送
```solidity
function setPriceFeedContract(address _newPriceFeed) 
    external 
    onlyRole(PRICE_FEED_MANAGER_ROLE) 
{
    if (_newPriceFeed == address(0)) revert InvalidPriceFeedAddress();
    priceFeed = AggregatorV3Interface(_newPriceFeed);
    emit PriceFeedUpdated(_newPriceFeed);
}
```

### 计算所需抵押品
```solidity
function getRequiredCollateralForMint(uint256 stablecoinAmount) 
    public 
    view 
    returns (uint256) 
{
    uint256 collateralPrice = getCurrentPrice();
    
    // 计算价值1 sUSD所需的抵押品
    uint256 baseCollateral = (stablecoinAmount * 10 ** collateralDecimals) / collateralPrice;
    
    // 应用抵押率
    uint256 requiredCollateral = (baseCollateral * collateralizationRatio) / 10000;
    
    return requiredCollateral;
}
```

### 计算赎回抵押品
```solidity
function getCollateralForRedeem(uint256 stablecoinAmount) 
    public 
    view 
    returns (uint256) 
{
    uint256 collateralPrice = getCurrentPrice();
    
    // 1:1赎回(不考虑超额抵押)
    uint256 collateralAmount = (stablecoinAmount * 10 ** collateralDecimals) / collateralPrice;
    
    return collateralAmount;
}
```

## 核心知识点

### 稳定币类型

1. **法币抵押**: 由USD等法币支持(USDC, USDT)
2. **加密抵押**: 由加密资产支持(DAI)
3. **算法稳定**: 通过算法维持挂钩(不在此例)

### 抵押率

```
150% = 15000 基点
存入 $150 价值的抵押品 → 铸造 $100 sUSD
```

### 价格馈送

使用Chainlink获取实时抵押品价格:
```solidity
(, int256 price, , , ) = priceFeed.latestRoundData();
```

### 铸造流程

1. 用户批准抵押品
2. 计算所需抵押品(基于价格和抵押率)
3. 转移抵押品到合约
4. 铸造sUSD给用户

### 赎回流程

1. 销毁用户的sUSD
2. 计算返还抵押品(1:1)
3. 转移抵押品给用户

### SafeERC20

```solidity
using SafeERC20 for IERC20;
collateralToken.safeTransferFrom(...);
collateralToken.safeTransfer(...);
```

安全地处理ERC-20代币转账,即使代币不返回布尔值也能正常工作。

### AccessControl

```solidity
bytes32 public constant PRICE_FEED_MANAGER_ROLE = keccak256("PRICE_FEED_MANAGER_ROLE");

_grantRole(PRICE_FEED_MANAGER_ROLE, msg.sender);

function setPriceFeedContract() external onlyRole(PRICE_FEED_MANAGER_ROLE) {}
```

基于角色的访问控制,更灵活than简单的所有者模式。

### 自定义错误

```solidity
error MintAmountIsZero();

if (stablecoinAmount == 0) revert MintAmountIsZero();
```

比字符串错误更节省gas。

### 小数处理

```solidity
// 归一化不同小数位数
if (priceFeedDecimals < 18) {
    return uint256(price) * (10 ** (18 - priceFeedDecimals));
}
```

确保所有计算使用一致的小数精度。

### 安全特性

1. **ReentrancyGuard**: 防止重入
2. **SafeERC20**: 安全的代币转账
3. **价格验证**: 确保价格 > 0
4. **抵押率最小值**: 至少100%
5. **权限控制**: 角色基础访问

