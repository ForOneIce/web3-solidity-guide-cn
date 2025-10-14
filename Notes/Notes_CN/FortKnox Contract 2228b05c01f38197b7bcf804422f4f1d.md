# 金库堡垒合约

本教程探索重入漏洞及其防护。构建了一个带有易受攻击和安全提取函数的 `GoldVault` 合约,以及一个 `GoldThief` 合约来演示攻击及其对安全版本的失败。

## 关键概念

### 金库合约
```solidity
contract GoldVault {
    mapping(address => uint256) public goldBalance;
    
    uint256 private _status;
    uint256 private constant _NOT_ENTERED = 1;
    uint256 private constant _ENTERED = 2;
    
    constructor() {
        _status = _NOT_ENTERED;
    }
    
    modifier nonReentrant() {
        require(_status != _ENTERED, "Reentrant call blocked");
        _status = _ENTERED;
        _;
        _status = _NOT_ENTERED;
    }
    
    function deposit() external payable {
        goldBalance[msg.sender] += msg.value;
    }
}
```

### 易受攻击的提取
```solidity
function vulnerableWithdraw() external {
    uint256 amount = goldBalance[msg.sender];
    require(amount > 0, "No balance");
    
    // 错误: 先发送ETH,后更新余额
    (bool success, ) = msg.sender.call{value: amount}("");
    require(success, "Transfer failed");
    
    goldBalance[msg.sender] = 0;  // 太晚了!
}
```

### 安全的提取
```solidity
function safeWithdraw() external nonReentrant {
    uint256 amount = goldBalance[msg.sender];
    require(amount > 0, "No balance");
    
    // 正确: 先更新状态,后发送ETH
    goldBalance[msg.sender] = 0;
    
    (bool sent, ) = msg.sender.call{value: amount}("");
    require(sent, "Transfer failed");
}
```

### 攻击合约接口
```solidity
interface IVault {
    function deposit() external payable;
    function vulnerableWithdraw() external;
    function safeWithdraw() external;
}
```

### 盗贼合约
```solidity
contract GoldThief {
    IVault public targetVault;
    address public owner;
    uint public attackCount;
    bool public attackingSafe;
    
    constructor(address _vaultAddress) {
        targetVault = IVault(_vaultAddress);
        owner = msg.sender;
    }
    
    function attackVulnerable() external payable {
        attackingSafe = false;
        attackCount = 0;
        
        targetVault.deposit{value: msg.value}();
        targetVault.vulnerableWithdraw();
    }
    
    function attackSafe() external payable {
        attackingSafe = true;
        attackCount = 0;
        
        targetVault.deposit{value: msg.value}();
        targetVault.safeWithdraw();
    }
    
    receive() external payable {
        attackCount++;
        
        if (!attackingSafe && address(targetVault).balance >= 1 ether && attackCount < 5) {
            targetVault.vulnerableWithdraw();
        }
        
        if (attackingSafe) {
            targetVault.safeWithdraw();
        }
    }
    
    function stealLoot() external {
        require(msg.sender == owner, "Not owner");
        payable(owner).transfer(address(this).balance);
    }
    
    function getBalance() external view returns (uint256) {
        return address(this).balance;
    }
}
```

## 核心知识点

### 重入攻击原理

1. **攻击者调用** `vulnerableWithdraw()`
2. **金库发送ETH** 到攻击者合约
3. **触发receive()** 攻击者的receive函数
4. **再次调用withdraw()** 余额还未更新!
5. **重复抽取** 直到金库耗尽

### Checks-Effects-Interactions模式

正确顺序:
1. **Checks**: 验证条件 (`require`)
2. **Effects**: 更新状态 (余额 = 0)
3. **Interactions**: 外部调用 (发送ETH)

### 防御措施

#### 1. ReentrancyGuard
```solidity
modifier nonReentrant() {
    require(_status != _ENTERED, "Reentrant call blocked");
    _status = _ENTERED;
    _;
    _status = _NOT_ENTERED;
}
```

#### 2. 先更新状态
```solidity
goldBalance[msg.sender] = 0;  // 先做这个
(bool sent, ) = msg.sender.call{value: amount}("");  // 后做这个
```

### DAO黑客事件

2016年,The DAO因重入攻击损失360万ETH。这导致了:
- 以太坊硬分叉(ETH vs ETC)
- 对智能合约安全的重视
- OpenZeppelin等安全库的诞生

### 最佳实践

1. **使用ReentrancyGuard**: 来自OpenZeppelin
2. **遵循CEI模式**: Checks-Effects-Interactions
3. **最小化外部调用**: 减少攻击面
4. **审计代码**: 专业安全审计
5. **测试攻击场景**: 主动查找漏洞

### OpenZeppelin的ReentrancyGuard

```solidity
import "@openzeppelin/contracts/security/ReentrancyGuard.sol";

contract MyContract is ReentrancyGuard {
    function withdraw() external nonReentrant {
        // 受保护的代码
    }
}
```

更简单,经过审计,推荐使用!

