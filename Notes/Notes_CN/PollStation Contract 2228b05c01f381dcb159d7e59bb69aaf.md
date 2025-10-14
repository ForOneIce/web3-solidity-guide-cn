# 投票站合约

本教程通过构建一个投票系统来探索Solidity中的数组和映射,用户可以添加候选人、检索列表、投票和查看票数。

## 关键概念

### 候选人存储
```solidity
string[] public candidateNames;
mapping(string => uint256) voteCount;
```

### 添加候选人
```solidity
function addCandidateNames(string memory _candidateNames) public {
    candidateNames.push(_candidateNames);
}
```

### 获取候选人列表
```solidity
function getcandidateNames() public view returns (string[] memory) {
    return candidateNames;
}
```

### 投票功能
```solidity
function vote(string memory _candidateNames) public {
    voteCount[_candidateNames]++;
}
```

### 查看票数
```solidity
function getVote(string memory _candidateNames) public view returns (uint256) {
    return voteCount[_candidateNames];
}
```

## 核心知识点

- 数组(`string[]`)用于存储候选人列表
- 映射(`mapping`)用于跟踪每个候选人的票数
- `push()` 函数向动态数组添加元素
- 映射自动初始化为默认值(0)
- 可以通过返回数组来获取完整列表

