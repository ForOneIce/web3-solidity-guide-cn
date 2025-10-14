# ClickCounter 合约

这个教程介绍了Solidity的基础概念,通过一个简单的合约来追踪函数被调用的次数。内容涵盖了许可证标识符、编译器版本指令、状态变量和函数。

## 关键概念

### 许可证标识符
```solidity
// SPDX-License-Identifier: MIT
```

### 编译器版本指令
```solidity
pragma solidity ^0.8.0;
```

### 合约定义
```solidity
contract ClickCounter {
    uint256 public counter;
    
    function click() public {
        counter++;
    }
}
```

## 工作原理

- `counter` 是一个公开的状态变量,用于记录点击次数
- `click()` 函数每次被调用时,会将计数器加1
- 这个合约演示了Solidity中最基本的状态变更操作

