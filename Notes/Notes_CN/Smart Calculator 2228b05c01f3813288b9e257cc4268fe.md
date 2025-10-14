# 智能计算器

本教程演示了合约间通信,构建了一个基础计算器,它将高级数学运算(幂运算、平方根)委托给单独的科学计算器合约。使用高级和低级调用方法。

## 关键概念

### 科学计算器合约
```solidity
import "./ScientificCalculator.sol";

contract ScientificCalculator {
    function power(uint256 base, uint256 exponent) public pure returns (uint256) {
        uint256 result = 1;
        for (uint256 i = 0; i < exponent; i++) {
            result *= base;
        }
        return result;
    }
    
    function squareRoot(uint256 number) public pure returns (uint256) {
        if (number == 0) return 0;
        uint256 z = (number + 1) / 2;
        uint256 y = number;
        while (z < y) {
            y = z;
            z = (number / z + z) / 2;
        }
        return y;
    }
}
```

### 计算器合约

#### 状态变量和设置
```solidity
contract Calculator {
    address public owner;
    address public scientificCalculatorAddress;
    
    constructor() {
        owner = msg.sender;
    }
    
    modifier onlyOwner() {
        require(msg.sender == owner, "Not the owner");
        _;
    }
    
    function setScientificCalculator(address _address) public onlyOwner {
        scientificCalculatorAddress = _address;
    }
}
```

#### 高级调用
```solidity
function power(uint256 base, uint256 exponent) public view returns (uint256) {
    require(scientificCalculatorAddress != address(0), "Calculator not set");
    
    ScientificCalculator scientificCalc = ScientificCalculator(scientificCalculatorAddress);
    return scientificCalc.power(base, exponent);
}
```

#### 低级调用
```solidity
function squareRoot(uint256 number) public view returns (uint256) {
    require(scientificCalculatorAddress != address(0), "Calculator not set");
    
    bytes memory data = abi.encodeWithSignature("squareRoot(uint256)", number);
    (bool success, bytes memory returnData) = scientificCalculatorAddress.call(data);
    require(success, "Call failed");
    
    return abi.decode(returnData, (uint256));
}
```

## 核心知识点

- `import` 用于导入其他Solidity文件
- 合约可以调用其他合约的函数
- **高级调用**: 直接实例化合约并调用函数
- **低级调用**: 使用 `call()` 和 `abi.encodeWithSignature()`
- `abi.decode()` 用于解析返回数据
- `pure` 函数不读取或修改状态
- 合约间通信是DeFi的基础

