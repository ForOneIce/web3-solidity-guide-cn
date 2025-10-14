# 以太坊存钱罐

本教程构建了一个共享储蓄池,批准的成员可以存入、查看余额和提取ETH。介绍了 `payable` 和 `msg.value` 用于处理真实的ETH。

## 关键概念

### 管理员和成员
```solidity
address public bankManager;
address[] members;
mapping(address => bool) public registeredMembers;
mapping(address => uint256) balance;

constructor() {
    bankManager = msg.sender;
}
```

### 修饰符
```solidity
modifier onlyBankManager() {
    require(msg.sender == bankManager, "Not the bank manager");
    _;
}

modifier onlyRegisteredMember() {
    require(registeredMembers[msg.sender], "Not a registered member");
    _;
}
```

### 成员管理
```solidity
function addMembers(address _member) public onlyBankManager {
    require(!registeredMembers[_member], "Already registered");
    registeredMembers[_member] = true;
    members.push(_member);
}

function getMembers() public view returns (address[] memory) {
    return members;
}
```

### 存款功能
```solidity
function deposit(uint256 _amount) public onlyRegisteredMember {
    balance[msg.sender] += _amount;
}

function depositAmountEther() public payable onlyRegisteredMember {
    balance[msg.sender] += msg.value;
}
```

### 提取功能
```solidity
function withdraw(uint256 _amount) public onlyRegisteredMember {
    require(balance[msg.sender] >= _amount, "Insufficient balance");
    balance[msg.sender] -= _amount;
}
```

## 核心知识点

- `payable` 函数可以接收ETH
- `msg.value` 包含发送的ETH金额(以wei为单位)
- 映射用于跟踪每个成员的余额
- 数组用于存储成员列表
- 修饰符可以组合使用,创建多层访问控制

