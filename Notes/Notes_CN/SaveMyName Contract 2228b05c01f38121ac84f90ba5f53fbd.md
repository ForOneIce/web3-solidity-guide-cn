# SaveMyName 合约

本教程专注于在区块链上存储和检索文本数据(姓名和简介)。介绍了字符串、内存存储、函数返回类型和 `view` 关键字。

## 关键概念

### 字符串存储
```solidity
string name;
string bio;
```

### 添加数据函数
```solidity
function add(string memory _name, string memory _bio) public {
    name = _name;
    bio = _bio;
}
```

### 检索数据
```solidity
function retrieve() public view returns (string memory, string memory) {
    return (name, bio);
}
```

### 保存并返回
```solidity
function saveAndRetrieve(string memory _name, string memory _bio) public returns (string memory, string memory) {
    name = _name;
    bio = _bio;
    return (name, bio);
}
```

## 核心知识点

- `string memory` 关键字用于临时存储字符串数据
- `view` 函数不修改状态,只读取数据
- 函数可以返回多个值
- 字符串数据存储在区块链上是永久的

