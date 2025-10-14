# 借贷池合约

本教程构建了一个基础的DeFi借贷平台。用户可以存入ETH、锁定抵押品、以利息借入ETH、偿还贷款和提取资金。介绍了DeFi机制、利率计算和抵押率。

## 关键概念

### 状态变量
```solidity
mapping(address => uint256) public depositBalances;
mapping(address => uint256) public borrowBalances;
mapping(address => uint256) public collateralBalances;

uint256 public interestRateBasisPoints = 500;  // 5%
uint256 public collateralFactorBasisPoints = 7500;  // 75%
uint256 public lastInterestAccrualTimestamp;
```

### 存款
```solidity
function deposit() external payable {
    require(msg.value > 0, "Must deposit something");
    depositBalances[msg.sender] += msg.value;
}
```

### 提款
```solidity
function withdraw(uint256 amount) external {
    require(depositBalances[msg.sender] >= amount, "Insufficient balance");
    depositBalances[msg.sender] -= amount;
    payable(msg.sender).transfer(amount);
}
```

### 存入抵押品
```solidity
function depositCollateral() external payable {
    require(msg.value > 0, "Must deposit collateral");
    collateralBalances[msg.sender] += msg.value;
}
```

### 提取抵押品
```solidity
function withdrawCollateral(uint256 amount) external {
    require(collateralBalances[msg.sender] >= amount, "Insufficient collateral");
    
    // 确保抵押率充足
    uint256 maxWithdraw = collateralBalances[msg.sender] - 
        (borrowBalances[msg.sender] * 10000) / collateralFactorBasisPoints;
    
    require(amount <= maxWithdraw, "Would under-collateralize loan");
    
    collateralBalances[msg.sender] -= amount;
    payable(msg.sender).transfer(amount);
}
```

### 借款
```solidity
function borrow(uint256 amount) external {
    uint256 maxBorrow = getMaxBorrowAmount(msg.sender);
    require(amount <= maxBorrow, "Insufficient collateral");
    
    borrowBalances[msg.sender] += amount;
    payable(msg.sender).transfer(amount);
}
```

### 偿还
```solidity
function repay() external payable {
    require(msg.value > 0, "Must repay something");
    require(borrowBalances[msg.sender] > 0, "No loan to repay");
    
    uint256 repayAmount = msg.value;
    if (repayAmount > borrowBalances[msg.sender]) {
        repayAmount = borrowBalances[msg.sender];
        payable(msg.sender).transfer(msg.value - repayAmount);
    }
    
    borrowBalances[msg.sender] -= repayAmount;
}
```

### 计算应计利息
```solidity
function calculateInterestAccrued(address user) public view returns (uint256) {
    if (lastInterestAccrualTimestamp == 0) return 0;
    
    uint256 timeElapsed = block.timestamp - lastInterestAccrualTimestamp;
    uint256 principal = borrowBalances[user];
    
    // 简化利息: principal * rate * time / (365 days * 10000)
    uint256 interest = (principal * interestRateBasisPoints * timeElapsed) / (365 days * 10000);
    return interest;
}
```

### 获取最大借款额
```solidity
function getMaxBorrowAmount(address user) public view returns (uint256) {
    uint256 collateral = collateralBalances[user];
    uint256 currentBorrow = borrowBalances[user];
    
    uint256 maxBorrow = (collateral * collateralFactorBasisPoints) / 10000;
    
    if (maxBorrow <= currentBorrow) return 0;
    return maxBorrow - currentBorrow;
}
```

### 获取总流动性
```solidity
function getTotalLiquidity() public view returns (uint256) {
    return address(this).balance;
}
```

## 核心知识点

### DeFi借贷机制

1. **存款**: 用户存入资产赚取利息
2. **抵押**: 用户锁定抵押品
3. **借款**: 基于抵押品价值借款
4. **利息**: 借款人支付利息
5. **清算**: 抵押不足时的保护机制(未实现)

### 关键比率

#### 利率
```solidity
500 basis points = 5% = 0.05
```

#### 抵押率
```solidity
7500 basis points = 75%
// 如果你有100 ETH抵押品,最多可借 75 ETH
```

### 基点(Basis Points)

- 1 basis point = 0.01%
- 100 basis points = 1%
- 10000 basis points = 100%

### 计算公式

#### 最大借款额
```
最大借款 = 抵押品 × 抵押率
```

#### 利息计算
```
利息 = 本金 × 利率 × 时间 / (365天 × 10000)
```

### 安全考虑

1. **过度抵押**: 要求超过100%的抵押
2. **利息计算**: 基于时间累积
3. **清算机制**: 保护协议免受坏账
4. **提款限制**: 确保充足抵押

### 改进方向

1. 添加清算功能
2. 动态利率
3. 多种代币支持
4. 预言机价格喂价
5. 治理代币

