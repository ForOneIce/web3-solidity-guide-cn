# 去中心化治理合约

本教程构建了一个基于ERC-20代币的去中心化治理系统(DAO)。支持加权投票、法定人数、提案押金、时间锁和链上执行提案。使用了 `IERC20`、`SafeCast` 和 `ReentrancyGuard`。

## 关键概念

### 提案结构
```solidity
struct Proposal {
    address proposer;
    string description;
    uint256 forVotes;
    uint256 againstVotes;
    uint256 startTime;
    uint256 endTime;
    bool executed;
    bool canceled;
    uint256 timelockEnd;
}
```

### 状态变量
```solidity
IERC20 public governanceToken;
mapping(uint256 => Proposal) public proposals;
mapping(uint256 => mapping(address => bool)) public hasVoted;

uint256 public nextProposalId;
uint256 public votingDuration;
uint256 public timelockDuration;
address public admin;
uint256 public quorumPercentage;
uint256 public proposalDepositAmount;
```

### 事件
```solidity
event ProposalCreated(
    uint256 indexed proposalId,
    address indexed proposer,
    string description,
    uint256 startTime,
    uint256 endTime
);

event Voted(
    uint256 indexed proposalId,
    address indexed voter,
    bool support,
    uint256 weight
);

event ProposalExecuted(uint256 indexed proposalId);
event QuorumNotMet(uint256 indexed proposalId);
event ProposalDepositPaid(uint256 indexed proposalId, address proposer, uint256 amount);
event ProposalDepositRefunded(uint256 indexed proposalId, address proposer, uint256 amount);
event TimelockSet(uint256 indexed proposalId, uint256 timelockEnd);
event ProposalTimelockStarted(uint256 indexed proposalId);
```

### 修饰符
```solidity
modifier onlyAdmin() {
    require(msg.sender == admin, "Not admin");
    _;
}
```

### 构造函数
```solidity
constructor(
    address _governanceToken,
    uint256 _votingDuration,
    uint256 _timelockDuration,
    uint256 _quorumPercentage,
    uint256 _proposalDepositAmount
) {
    require(_governanceToken != address(0), "Invalid token");
    require(_votingDuration > 0, "Invalid duration");
    require(_quorumPercentage > 0 && _quorumPercentage <= 100, "Invalid quorum");
    
    governanceToken = IERC20(_governanceToken);
    votingDuration = _votingDuration;
    timelockDuration = _timelockDuration;
    admin = msg.sender;
    quorumPercentage = _quorumPercentage;
    proposalDepositAmount = _proposalDepositAmount;
}
```

### 设置法定人数
```solidity
function setQuorumPercentage(uint256 _newQuorum) external onlyAdmin {
    require(_newQuorum > 0 && _newQuorum <= 100, "Invalid quorum");
    quorumPercentage = _newQuorum;
}
```

### 设置提案押金
```solidity
function setProposalDepositAmount(uint256 _newAmount) external onlyAdmin {
    proposalDepositAmount = _newAmount;
}
```

### 设置时间锁
```solidity
function setTimelockDuration(uint256 _newDuration) external onlyAdmin {
    timelockDuration = _newDuration;
}
```

### 创建提案
```solidity
function createProposal(string memory description) external returns (uint256) {
    require(bytes(description).length > 0, "Empty description");
    
    if (proposalDepositAmount > 0) {
        governanceToken.transferFrom(msg.sender, address(this), proposalDepositAmount);
        emit ProposalDepositPaid(nextProposalId, msg.sender, proposalDepositAmount);
    }
    
    uint256 proposalId = nextProposalId++;
    
    proposals[proposalId] = Proposal({
        proposer: msg.sender,
        description: description,
        forVotes: 0,
        againstVotes: 0,
        startTime: block.timestamp,
        endTime: block.timestamp + votingDuration,
        executed: false,
        canceled: false,
        timelockEnd: 0
    });
    
    emit ProposalCreated(
        proposalId,
        msg.sender,
        description,
        block.timestamp,
        block.timestamp + votingDuration
    );
    
    return proposalId;
}
```

### 投票
```solidity
function vote(uint256 proposalId, bool support) external {
    Proposal storage proposal = proposals[proposalId];
    
    require(block.timestamp >= proposal.startTime, "Not started");
    require(block.timestamp <= proposal.endTime, "Ended");
    require(!proposal.executed, "Already executed");
    require(!proposal.canceled, "Canceled");
    require(!hasVoted[proposalId][msg.sender], "Already voted");
    
    uint256 weight = governanceToken.balanceOf(msg.sender);
    require(weight > 0, "No voting power");
    
    hasVoted[proposalId][msg.sender] = true;
    
    if (support) {
        proposal.forVotes += weight;
    } else {
        proposal.againstVotes += weight;
    }
    
    emit Voted(proposalId, msg.sender, support, weight);
}
```

### 完成提案
```solidity
function finalizeProposal(uint256 proposalId) external {
    Proposal storage proposal = proposals[proposalId];
    
    require(block.timestamp > proposal.endTime, "Voting not ended");
    require(!proposal.executed, "Already executed");
    require(!proposal.canceled, "Canceled");
    
    uint256 totalVotes = proposal.forVotes + proposal.againstVotes;
    uint256 totalSupply = governanceToken.totalSupply();
    uint256 quorumRequired = (totalSupply * quorumPercentage) / 100;
    
    if (totalVotes >= quorumRequired && proposal.forVotes > proposal.againstVotes) {
        if (timelockDuration > 0) {
            proposal.timelockEnd = block.timestamp + timelockDuration;
            emit TimelockSet(proposalId, proposal.timelockEnd);
            emit ProposalTimelockStarted(proposalId);
        } else {
            proposal.executed = true;
            
            if (proposalDepositAmount > 0) {
                governanceToken.transfer(proposal.proposer, proposalDepositAmount);
                emit ProposalDepositRefunded(proposalId, proposal.proposer, proposalDepositAmount);
            }
            
            emit ProposalExecuted(proposalId);
        }
    } else {
        proposal.canceled = true;
        emit QuorumNotMet(proposalId);
    }
}
```

### 执行提案
```solidity
function executeProposal(uint256 proposalId) external {
    Proposal storage proposal = proposals[proposalId];
    
    require(proposal.timelockEnd > 0, "No timelock set");
    require(block.timestamp >= proposal.timelockEnd, "Timelock not ended");
    require(!proposal.executed, "Already executed");
    require(!proposal.canceled, "Canceled");
    
    proposal.executed = true;
    
    if (proposalDepositAmount > 0) {
        governanceToken.transfer(proposal.proposer, proposalDepositAmount);
        emit ProposalDepositRefunded(proposalId, proposal.proposer, proposalDepositAmount);
    }
    
    emit ProposalExecuted(proposalId);
}
```

### 获取提案结果
```solidity
function getProposalResult(uint256 proposalId) external view returns (
    bool passed,
    uint256 forVotes,
    uint256 againstVotes,
    bool executed
) {
    Proposal memory proposal = proposals[proposalId];
    
    passed = proposal.forVotes > proposal.againstVotes;
    return (passed, proposal.forVotes, proposal.againstVotes, proposal.executed);
}
```

### 获取提案详情
```solidity
function getProposalDetails(uint256 proposalId) external view returns (
    address proposer,
    string memory description,
    uint256 forVotes,
    uint256 againstVotes,
    uint256 startTime,
    uint256 endTime,
    bool executed,
    bool canceled
) {
    Proposal memory proposal = proposals[proposalId];
    return (
        proposal.proposer,
        proposal.description,
        proposal.forVotes,
        proposal.againstVotes,
        proposal.startTime,
        proposal.endTime,
        proposal.executed,
        proposal.canceled
    );
}
```

## 核心知识点

### DAO治理流程

1. **创建提案**: 任何人(通常需要代币或押金)
2. **投票期**: 代币持有者投票
3. **计票**: 检查法定人数和多数
4. **时间锁**: 延迟执行(可选)
5. **执行**: 如果通过,执行提案

### 加权投票

```
投票权 = 代币余额
```

持有更多治理代币 = 更大投票权

### 法定人数

```
法定人数 = 总供应量 × 法定人数百分比
```

例如: 20%法定人数,1000万代币总量 = 需要200万代币投票

### 时间锁

提案通过和执行之间的延迟:
- 给社区时间审查
- 允许退出机会
- 防止恶意提案

### 提案押金

- 防止垃圾提案
- 提案通过时退还
- 提案失败时可能罚没(未实现)

### 投票计数

```
通过条件:
1. totalVotes >= quorumRequired
2. forVotes > againstVotes
```

### 提案状态

- **Active**: 正在投票
- **Succeeded**: 通过,等待时间锁
- **Defeated**: 未达到法定人数或反对更多
- **Executed**: 已执行
- **Canceled**: 被取消

### 安全考虑

1. **双重投票防护**: `hasVoted` 映射
2. **时间检查**: 确保在正确时间段
3. **状态验证**: 检查执行和取消状态
4. **投票权验证**: 要求 balance > 0

### 改进方向

1. 委托投票
2. 动态法定人数
3. 提案类别
4. 链上执行(调用函数)
5. 投票NFT
6. 多签审批

