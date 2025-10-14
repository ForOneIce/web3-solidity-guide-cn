# 安全保管箱合约

本教程使用接口、抽象合约和继承构建了一个模块化的保管箱系统。定义了一个 `IDepositBox` 接口、一个 `BaseDepositBox` 抽象合约,以及具体实现如 `BasicDepositBox`、`PremiumDepositBox` 和 `TimeLockedDepositBox`。`VaultManager` 合约协调创建和管理这些不同类型的保管箱。

## 关键概念

### 接口
```solidity
interface IDepositBox {
    function getOwner() external view returns (address);
    function transferOwnership(address newOwner) external;
    function storeSecret(string memory secret) external;
    function getSecret() external view returns (string memory);
    function getBoxType() external pure returns (string memory);
    function getDepositTime() external view returns (uint256);
}
```

### 抽象合约
```solidity
abstract contract BaseDepositBox is IDepositBox {
    address public owner;
    string public metadata;
    string private secret;
    uint256 public depositTime;
    
    constructor(string memory _metadata) {
        owner = msg.sender;
        metadata = _metadata;
        depositTime = block.timestamp;
    }
    
    modifier onlyOwner() {
        require(msg.sender == owner, "Not owner");
        _;
    }
    
    function getOwner() external view override returns (address) {
        return owner;
    }
    
    function transferOwnership(address newOwner) external override onlyOwner {
        require(newOwner != address(0), "Invalid address");
        owner = newOwner;
    }
    
    function storeSecret(string memory _secret) external override onlyOwner {
        secret = _secret;
    }
    
    function getSecret() external view override onlyOwner returns (string memory) {
        return secret;
    }
    
    function getDepositTime() external view override returns (uint256) {
        return depositTime;
    }
    
    // 抽象函数 - 必须由子合约实现
    function getBoxType() external pure virtual override returns (string memory);
}
```

### 基础保管箱
```solidity
contract BasicDepositBox is BaseDepositBox {
    constructor(string memory _metadata) BaseDepositBox(_metadata) {}
    
    function getBoxType() external pure override returns (string memory) {
        return "Basic";
    }
}
```

### 高级保管箱
```solidity
contract PremiumDepositBox is BaseDepositBox {
    constructor(string memory _metadata) BaseDepositBox(_metadata) {}
    
    function getBoxType() external pure override returns (string memory) {
        return "Premium";
    }
}
```

### 时间锁定保管箱
```solidity
contract TimeLockedDepositBox is BaseDepositBox {
    uint256 public unlockTime;
    
    constructor(string memory _metadata, uint256 _lockDuration) 
        BaseDepositBox(_metadata) 
    {
        unlockTime = block.timestamp + _lockDuration;
    }
    
    modifier timeUnlocked() {
        require(block.timestamp >= unlockTime, "Still locked");
        _;
    }
    
    function getSecret() external view override onlyOwner timeUnlocked returns (string memory) {
        return super.getSecret();
    }
    
    function getBoxType() external pure override returns (string memory) {
        return "Time-Locked";
    }
}
```

### 保管箱管理器
```solidity
contract VaultManager {
    struct BoxInfo {
        address boxAddress;
        string boxType;
        string metadata;
    }
    
    mapping(address => BoxInfo[]) public userBoxes;
    mapping(address => string) public boxNames;
    
    function createBasicBox(string memory _metadata) external returns (address) {
        BasicDepositBox newBox = new BasicDepositBox(_metadata);
        newBox.transferOwnership(msg.sender);
        
        userBoxes[msg.sender].push(BoxInfo({
            boxAddress: address(newBox),
            boxType: "Basic",
            metadata: _metadata
        }));
        
        return address(newBox);
    }
    
    function createPremiumBox(string memory _metadata) external returns (address) {
        PremiumDepositBox newBox = new PremiumDepositBox(_metadata);
        newBox.transferOwnership(msg.sender);
        
        userBoxes[msg.sender].push(BoxInfo({
            boxAddress: address(newBox),
            boxType: "Premium",
            metadata: _metadata
        }));
        
        return address(newBox);
    }
    
    function createTimeLockedBox(string memory _metadata, uint256 _lockDuration) 
        external returns (address) 
    {
        TimeLockedDepositBox newBox = new TimeLockedDepositBox(_metadata, _lockDuration);
        newBox.transferOwnership(msg.sender);
        
        userBoxes[msg.sender].push(BoxInfo({
            boxAddress: address(newBox),
            boxType: "Time-Locked",
            metadata: _metadata
        }));
        
        return address(newBox);
    }
    
    function nameBox(address boxAddress, string memory name) external {
        boxNames[boxAddress] = name;
    }
    
    function storeSecret(address boxAddress, string memory secret) external {
        IDepositBox(boxAddress).storeSecret(secret);
    }
    
    function transferBoxOwnership(address boxAddress, address newOwner) external {
        IDepositBox(boxAddress).transferOwnership(newOwner);
    }
    
    function getUserBoxes(address user) external view returns (BoxInfo[] memory) {
        return userBoxes[user];
    }
    
    function getBoxName(address boxAddress) external view returns (string memory) {
        return boxNames[boxAddress];
    }
    
    function getBoxInfo(address boxAddress) external view returns (
        address owner,
        string memory boxType,
        uint256 depositTime
    ) {
        IDepositBox box = IDepositBox(boxAddress);
        return (
            box.getOwner(),
            box.getBoxType(),
            box.getDepositTime()
        );
    }
}
```

## 核心知识点

### 接口 (Interface)

- 定义合约必须实现的函数
- 只有函数签名,没有实现
- 所有函数都是 `external`
- 用于标准化和互操作性

### 抽象合约 (Abstract Contract)

- 可以有实现的函数和未实现的函数
- 至少有一个未实现(abstract)函数
- 不能直接部署
- 用作其他合约的基类

### 继承

```solidity
contract Child is Parent {
    // 继承Parent的所有内容
}
```

### Virtual 和 Override

- `virtual`: 函数可以被重写
- `override`: 重写父合约的函数

### 修饰符叠加

```solidity
function getSecret() external view override onlyOwner timeUnlocked returns (string memory)
```

多个修饰符可以组合使用。

### 工厂模式

`VaultManager` 使用工厂模式动态创建保管箱:

```solidity
BasicDepositBox newBox = new BasicDepositBox(_metadata);
```

### 模块化设计优势

1. **代码重用**: 基类可被多个子类使用
2. **易于扩展**: 添加新类型很简单
3. **关注点分离**: 每个合约专注一个目的
4. **可维护性**: 修改局部不影响整体

### 使用场景

- NFT系列(不同稀有度)
- 会员系统(不同等级)
- 金融产品(不同风险级别)
- 游戏物品(不同类型)

