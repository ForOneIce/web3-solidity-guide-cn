# 主密钥合约

本教程介绍了Solidity中的继承,通过创建一个用于访问控制的 `Ownable` 合约,以及继承其所有权逻辑的 `VaultMaster`。还涵盖了事件和OpenZeppelin的 `Ownable` 合约。

## 关键概念

### Ownable 合约
```solidity
contract Ownable {
    address private owner;
    
    event OwnershipTransferred(address indexed previousOwner, address indexed newOwner);
    
    constructor() {
        owner = msg.sender;
        emit OwnershipTransferred(address(0), msg.sender);
    }
    
    modifier onlyOwner() {
        require(msg.sender == owner, "Not the owner");
        _;
    }
    
    function ownerAddress() public view returns (address) {
        return owner;
    }
    
    function transferOwnership(address _newOwner) public onlyOwner {
        require(_newOwner != address(0), "Invalid address");
        address previousOwner = owner;
        owner = _newOwner;
        emit OwnershipTransferred(previousOwner, _newOwner);
    }
}
```

### VaultMaster 合约(继承)
```solidity
import "./ownable.sol";

contract VaultMaster is Ownable {
    event DepositSuccessful(address indexed account, uint256 value);
    event WithdrawSuccessful(address indexed recipient, uint256 value);
    
    function getBalance() public view returns (uint256) {
        return address(this).balance;
    }
    
    function deposit() public payable {
        require(msg.value > 0, "Must send ETH");
        emit DepositSuccessful(msg.sender, msg.value);
    }
    
    function withdraw(address _to, uint256 _amount) public onlyOwner {
        require(_amount <= address(this).balance, "Insufficient balance");
        payable(_to).transfer(_amount);
        emit WithdrawSuccessful(_to, _amount);
    }
}
```

### 使用 OpenZeppelin 的 Ownable
```solidity
import "@openzeppelin/contracts/access/Ownable.sol";

contract VaultMaster is Ownable {
    constructor() Ownable(msg.sender) {}
    
    // ... 其余代码相同
}
```

## 核心知识点

- **继承**: 使用 `is` 关键字继承另一个合约的功能
- **事件**: 使用 `event` 声明,用 `emit` 触发,便于前端监听
- `indexed` 参数可以被过滤和搜索
- **修饰符继承**: 子合约可以使用父合约的修饰符
- **OpenZeppelin**: 提供经过审计的标准合约实现
- `private` vs `public`: 私有变量只能在合约内部访问
- 继承使代码可重用,减少重复

