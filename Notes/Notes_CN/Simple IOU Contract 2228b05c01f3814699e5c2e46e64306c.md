# 简单欠条合约

本教程创建了一个链上群组账本,朋友们可以追踪债务、存储ETH并进行结算。介绍了嵌套映射和不同的ETH转账方法(`transfer()` vs `call()`)。

## 关键概念

### 状态变量
```solidity
address public owner;
mapping(address => bool) public registeredFriends;
address[] public friendList;
mapping(address => uint256) public balances;
mapping(address => mapping(address => uint256)) public debts;
```

### 构造函数和修饰符
```solidity
constructor() {
    owner = msg.sender;
}

modifier onlyOwner() {
    require(msg.sender == owner, "Not the owner");
    _;
}

modifier onlyRegistered() {
    require(registeredFriends[msg.sender], "Not registered");
    _;
}
```

### 添加朋友
```solidity
function addFriend(address _friend) public onlyOwner {
    require(!registeredFriends[_friend], "Already registered");
    registeredFriends[_friend] = true;
    friendList.push(_friend);
}
```

### 存入钱包
```solidity
function depositIntoWallet() public payable onlyRegistered {
    balances[msg.sender] += msg.value;
}
```

### 记录债务
```solidity
function recordDebt(address _debtor, uint256 _amount) public onlyRegistered {
    debts[_debtor][msg.sender] += _amount;
}
```

### 从钱包支付
```solidity
function payFromWallet(address _creditor, uint256 _amount) public onlyRegistered {
    require(balances[msg.sender] >= _amount, "Insufficient balance");
    require(debts[msg.sender][_creditor] >= _amount, "No debt to pay");
    
    balances[msg.sender] -= _amount;
    balances[_creditor] += _amount;
    debts[msg.sender][_creditor] -= _amount;
}
```

### ETH转账方式

#### 使用 transfer()
```solidity
function transferEther(address payable _to, uint256 _amount) public onlyRegistered {
    require(balances[msg.sender] >= _amount, "Insufficient balance");
    balances[msg.sender] -= _amount;
    _to.transfer(_amount);
}
```

#### 使用 call()
```solidity
function transferEtherViaCall(address payable _to, uint256 _amount) public onlyRegistered {
    require(balances[msg.sender] >= _amount, "Insufficient balance");
    balances[msg.sender] -= _amount;
    (bool success, ) = _to.call{value: _amount}("");
    require(success, "Transfer failed");
}
```

### 提取和查询
```solidity
function withdraw(uint256 _amount) public onlyRegistered {
    require(balances[msg.sender] >= _amount, "Insufficient balance");
    balances[msg.sender] -= _amount;
    payable(msg.sender).transfer(_amount);
}

function checkBalance() public view onlyRegistered returns (uint256) {
    return balances[msg.sender];
}
```

## 核心知识点

- 嵌套映射 `mapping(address => mapping(address => uint256))` 用于跟踪两方之间的关系
- `transfer()` 是传统的ETH转账方式,有2300 gas限制
- `call()` 是更灵活和推荐的转账方式
- `payable` 地址可以接收ETH
- 在转账前更新余额可以防止重入攻击

