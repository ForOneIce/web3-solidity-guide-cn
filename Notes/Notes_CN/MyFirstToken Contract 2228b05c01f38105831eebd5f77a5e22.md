# 我的第一个代币合约

本教程解释了ERC-20代币标准,并指导从头开始构建一个最小化的ERC-20代币。涵盖了代币元数据、供应量、余额、授权额度、转账和批准。

## 关键概念

### ERC-20 标准

ERC-20是以太坊上代币的标准接口,定义了所有代币必须实现的函数。

### 代币元数据
```solidity
string public name = "SimpleToken";
string public symbol = "SIM";
uint8 public decimals = 18;
uint256 public totalSupply;
```

### 余额和授权
```solidity
mapping(address => uint256) public balanceOf;
mapping(address => mapping(address => uint256)) public allowance;
```

### 事件
```solidity
event Transfer(address indexed from, address indexed to, uint256 value);
event Approval(address indexed owner, address indexed spender, uint256 value);
```

### 构造函数
```solidity
constructor(uint256 _initialSupply) {
    totalSupply = _initialSupply * (10 ** uint256(decimals));
    balanceOf[msg.sender] = totalSupply;
    emit Transfer(address(0), msg.sender, totalSupply);
}
```

### 转账函数
```solidity
function transfer(address _to, uint256 _value) public returns (bool) {
    require(_to != address(0), "Invalid address");
    _transfer(msg.sender, _to, _value);
    return true;
}
```

### 内部转账逻辑
```solidity
function _transfer(address _from, address _to, uint256 _value) internal {
    require(balanceOf[_from] >= _value, "Insufficient balance");
    
    balanceOf[_from] -= _value;
    balanceOf[_to] += _value;
    
    emit Transfer(_from, _to, _value);
}
```

### 批准函数
```solidity
function approve(address _spender, uint256 _value) public returns (bool) {
    require(_spender != address(0), "Invalid address");
    
    allowance[msg.sender][_spender] = _value;
    emit Approval(msg.sender, _spender, _value);
    return true;
}
```

### 授权转账
```solidity
function transferFrom(address _from, address _to, uint256 _value) public returns (bool) {
    require(_value <= allowance[_from][msg.sender], "Allowance exceeded");
    
    allowance[_from][msg.sender] -= _value;
    _transfer(_from, _to, _value);
    return true;
}
```

## 核心知识点

- **ERC-20标准**: 定义了代币的基本接口
- `decimals`: 通常为18,表示代币可分割性
- `totalSupply`: 代币总供应量
- `balanceOf`: 映射,跟踪每个地址的余额
- `allowance`: 嵌套映射,跟踪授权额度
- `transfer()`: 直接转账
- `approve()`: 授权其他地址花费你的代币
- `transferFrom()`: 使用授权额度进行转账
- `_transfer()`: 内部函数,包含实际转账逻辑
- 事件用于通知外部应用转账和批准操作

