# Gas高效投票合约

本教程专注于投票合约中的gas优化。比较了一个未优化版本和一个gas高效版本,演示了如何使用更小的数据类型、映射代替动态数组,以及位运算来追踪投票者。

## 关键概念

### Gas优化策略

1. 使用更小的数据类型 (`uint8`, `uint32` 代替 `uint256`)
2. 使用映射代替动态数组
3. 使用位运算进行投票者追踪
4. 使用 `bytes32` 代替 `string`
5. 结构体打包

### 提案结构
```solidity
struct Proposal {
    bytes32 name;
    uint32 voteCount;
    uint32 startTime;
    uint32 endTime;
    bool executed;
}
```

### 状态变量
```solidity
uint8 public proposalCount;
mapping(uint8 => Proposal) public proposals;
mapping(address => uint256) private voterRegistry;  // 位图存储投票
mapping(uint8 => uint32) public proposalVoterCount;
```

### 事件
```solidity
event ProposalCreated(uint8 indexed proposalId, bytes32 name);
event Voted(address indexed voter, uint8 indexed proposalId);
event ProposalExecuted(uint8 indexed proposalId);
```

### 创建提案
```solidity
function createProposal(bytes32 name, uint32 duration) external {
    uint8 proposalId = proposalCount;
    proposalCount++;
    
    Proposal memory newProposal = Proposal({
        name: name,
        voteCount: 0,
        startTime: uint32(block.timestamp),
        endTime: uint32(block.timestamp + duration),
        executed: false
    });
    
    proposals[proposalId] = newProposal;
    emit ProposalCreated(proposalId, name);
}
```

### 投票(使用位运算)
```solidity
function vote(uint8 proposalId) external {
    require(proposalId < proposalCount, "Invalid proposal");
    require(block.timestamp >= proposals[proposalId].startTime, "Not started");
    require(block.timestamp <= proposals[proposalId].endTime, "Ended");
    require(!proposals[proposalId].executed, "Already executed");
    
    uint256 voterData = voterRegistry[msg.sender];
    uint256 mask = 1 << proposalId;
    
    require((voterData & mask) == 0, "Already voted");
    
    voterRegistry[msg.sender] = voterData | mask;
    proposals[proposalId].voteCount++;
    proposalVoterCount[proposalId]++;
    
    emit Voted(msg.sender, proposalId);
}
```

### 执行提案
```solidity
function executeProposal(uint8 proposalId) external {
    require(proposalId < proposalCount, "Invalid proposal");
    require(block.timestamp > proposals[proposalId].endTime, "Not ended");
    require(!proposals[proposalId].executed, "Already executed");
    
    proposals[proposalId].executed = true;
    emit ProposalExecuted(proposalId);
}
```

### 检查投票状态
```solidity
function hasVoted(address voter, uint8 proposalId) external view returns (bool) {
    return (voterRegistry[voter] & (1 << proposalId)) != 0;
}
```

### 获取提案详情
```solidity
function getProposal(uint8 proposalId) external view returns (
    bytes32 name,
    uint32 voteCount,
    uint32 startTime,
    uint32 endTime,
    bool executed
) {
    Proposal memory p = proposals[proposalId];
    return (p.name, p.voteCount, p.startTime, p.endTime, p.executed);
}
```

## 核心知识点

### Gas优化技术

1. **更小的数据类型**: `uint8` 比 `uint256` 便宜
2. **映射 vs 数组**: 映射的查找成本恒定
3. **位运算**: 
   - `1 << proposalId` 创建掩码
   - `voterData & mask` 检查特定位
   - `voterData | mask` 设置特定位
4. **bytes32 vs string**: `bytes32` 是固定大小,更便宜
5. **结构体打包**: 将小类型组合在一起以节省存储槽
6. **Memory vs Storage**: 使用 `memory` 进行临时操作

### 位运算解释

- `1 << proposalId`: 将1左移 proposalId 位
- `voterData & mask`: 按位与,检查该位是否为1
- `voterData | mask`: 按位或,将该位设置为1

这允许在单个 `uint256` 中存储256个不同的投票状态!

