# 仅管理员合约

本教程解释了如何使用修饰符创建只有合约所有者才能调用的函数。它模拟了一个宝库系统,所有者可以控制添加宝藏、批准提取和转移所有权。

## 关键概念

### 所有者声明
```solidity
address public owner;

constructor() {
    owner = msg.sender;
}
```

### 仅所有者修饰符
```solidity
modifier onlyOwner() {
    require(msg.sender == owner, "Not the owner");
    _;
}
```

### 宝藏管理
```solidity
uint256 public treasureAmount;

function addTreasure(uint256 amount) public onlyOwner {
    treasureAmount += amount;
}
```

### 提取权限
```solidity
mapping(address => uint256) public withdrawalAllowance;

function approveWithdrawal(address recipient, uint256 amount) public onlyOwner {
    withdrawalAllowance[recipient] = amount;
}
```

### 提取功能
```solidity
mapping(address => bool) public hasWithdrawn;

function withdrawTreasure(uint256 amount) public {
    require(amount <= withdrawalAllowance[msg.sender], "Insufficient allowance");
    require(!hasWithdrawn[msg.sender], "Already withdrawn");
    
    hasWithdrawn[msg.sender] = true;
    withdrawalAllowance[msg.sender] -= amount;
}
```

### 重置状态
```solidity
function resetWithdrawalStatus(address user) public onlyOwner {
    hasWithdrawn[user] = false;
}
```

### 转移所有权
```solidity
function transferOwnership(address newOwner) public onlyOwner {
    owner = newOwner;
}
```

### 查看详情
```solidity
function getTreasureDetails() public view onlyOwner returns (uint256) {
    return treasureAmount;
}
```

## 核心知识点

- `modifier` 用于创建可重用的访问控制逻辑
- `msg.sender` 是调用函数的地址
- `onlyOwner` 确保只有所有者可以执行特定操作
- 映射可以跟踪多个用户的状态和权限

