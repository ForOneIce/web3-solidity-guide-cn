# 收益农场合约

本教程实现了一个收益农场平台,用户可以质押ERC-20代币以随时间赚取奖励。包括质押、取消质押、领取奖励、紧急提款和管理员控制的奖励充值功能。使用了 `IERC20`、`ReentrancyGuard`、`SafeCast` 和自定义 `IERC20Metadata` 接口。

## 关键概念

### 导入和接口
```solidity
import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "@openzeppelin/contracts/security/ReentrancyGuard.sol";
import "@openzeppelin/contracts/utils/math/SafeCast.sol";

interface IERC20Metadata {
    function decimals() external view returns (uint8);
}
```

### 状态变量
```solidity
IERC20 public stakingToken;
IERC20 public rewardToken;
uint256 public rewardRatePerSecond;
address public owner;

uint8 public stakingTokenDecimals;

struct StakerInfo {
    uint256 stakedAmount;
    uint256 rewardDebt;
    uint256 lastUpdateTime;
}

mapping(address => StakerInfo) public stakers;
```

### 事件
```solidity
event Staked(address indexed user, uint256 amount);
event Unstaked(address indexed user, uint256 amount);
event RewardClaimed(address indexed user, uint256 reward);
event EmergencyWithdraw(address indexed user, uint256 amount);
event RewardRefilled(uint256 amount);
```

### 修饰符
```solidity
modifier onlyOwner() {
    require(msg.sender == owner, "Not owner");
    _;
}
```

### 构造函数
```solidity
constructor(
    address _stakingToken,
    address _rewardToken,
    uint256 _rewardRatePerSecond
) {
    require(_stakingToken != address(0), "Invalid staking token");
    require(_rewardToken != address(0), "Invalid reward token");
    require(_rewardRatePerSecond > 0, "Invalid reward rate");
    
    stakingToken = IERC20(_stakingToken);
    rewardToken = IERC20(_rewardToken);
    rewardRatePerSecond = _rewardRatePerSecond;
    owner = msg.sender;
    
    stakingTokenDecimals = IERC20Metadata(_stakingToken).decimals();
}
```

### 质押
```solidity
function stake(uint256 amount) external nonReentrant {
    require(amount > 0, "Cannot stake 0");
    
    updateRewards(msg.sender);
    
    stakingToken.transferFrom(msg.sender, address(this), amount);
    
    stakers[msg.sender].stakedAmount += amount;
    
    emit Staked(msg.sender, amount);
}
```

### 取消质押
```solidity
function unstake(uint256 amount) external nonReentrant {
    require(amount > 0, "Cannot unstake 0");
    require(stakers[msg.sender].stakedAmount >= amount, "Insufficient balance");
    
    updateRewards(msg.sender);
    
    stakers[msg.sender].stakedAmount -= amount;
    
    stakingToken.transfer(msg.sender, amount);
    
    emit Unstaked(msg.sender, amount);
}
```

### 领取奖励
```solidity
function claimRewards() external nonReentrant {
    updateRewards(msg.sender);
    
    uint256 reward = stakers[msg.sender].rewardDebt;
    require(reward > 0, "No rewards");
    
    stakers[msg.sender].rewardDebt = 0;
    
    rewardToken.transfer(msg.sender, reward);
    
    emit RewardClaimed(msg.sender, reward);
}
```

### 紧急提款
```solidity
function emergencyWithdraw() external nonReentrant {
    uint256 amount = stakers[msg.sender].stakedAmount;
    require(amount > 0, "No stake");
    
    stakers[msg.sender].stakedAmount = 0;
    stakers[msg.sender].rewardDebt = 0;
    stakers[msg.sender].lastUpdateTime = 0;
    
    stakingToken.transfer(msg.sender, amount);
    
    emit EmergencyWithdraw(msg.sender, amount);
}
```

### 充值奖励
```solidity
function refillRewards(uint256 amount) external onlyOwner {
    require(amount > 0, "Cannot refill 0");
    
    rewardToken.transferFrom(msg.sender, address(this), amount);
    
    emit RewardRefilled(amount);
}
```

### 更新奖励(内部)
```solidity
function updateRewards(address user) internal {
    StakerInfo storage staker = stakers[user];
    
    if (staker.stakedAmount > 0) {
        uint256 pending = pendingRewards(user);
        staker.rewardDebt += pending;
    }
    
    staker.lastUpdateTime = block.timestamp;
}
```

### 查看待领取奖励
```solidity
function pendingRewards(address user) public view returns (uint256) {
    StakerInfo memory staker = stakers[user];
    
    if (staker.stakedAmount == 0) return 0;
    
    uint256 timeElapsed = block.timestamp - staker.lastUpdateTime;
    if (timeElapsed == 0) return 0;
    
    // 奖励 = 质押量 × 奖励率 × 时间
    uint256 reward = (staker.stakedAmount * rewardRatePerSecond * timeElapsed) 
        / (10 ** stakingTokenDecimals);
    
    return reward;
}
```

### 获取质押代币小数
```solidity
function getStakingTokenDecimals() external view returns (uint8) {
    return stakingTokenDecimals;
}
```

## 核心知识点

### 收益农场机制

1. **质押**: 用户锁定代币
2. **赚取**: 随时间累积奖励
3. **领取**: 提取累积的奖励
4. **取消质押**: 取回质押的代币

### 奖励计算

```
奖励 = 质押量 × 奖励率 × 时间
```

例如:
- 质押: 1000代币
- 奖励率: 0.1 代币/秒
- 时间: 100秒
- 奖励 = 1000 × 0.1 × 100 / 10^18 = 10代币

### StakerInfo结构

```solidity
struct StakerInfo {
    uint256 stakedAmount;      // 质押数量
    uint256 rewardDebt;        // 累积的未领取奖励
    uint256 lastUpdateTime;    // 上次更新时间
}
```

### 奖励更新策略

每次交互时(质押/取消质押/领取):
1. 计算自上次更新以来的奖励
2. 累加到 `rewardDebt`
3. 更新 `lastUpdateTime`

### 紧急提款

- 放弃所有奖励
- 立即取回质押代币
- 用于紧急情况

### SafeCast使用

虽然导入了,但在简化版本中可能未使用。用于安全类型转换:
```solidity
using SafeCast for uint256;
uint128 value = amount.toUint128();
```

### ReentrancyGuard

保护所有修改状态的公共函数:
```solidity
function stake(uint256 amount) external nonReentrant {
    // 安全的
}
```

### 小数处理

```solidity
// 归一化计算
uint256 reward = (staker.stakedAmount * rewardRatePerSecond * timeElapsed) 
    / (10 ** stakingTokenDecimals);
```

确保正确处理不同小数位数的代币。

### 安全考虑

1. **重入保护**: 所有外部函数使用 `nonReentrant`
2. **零地址检查**: 构造函数验证
3. **余额检查**: 取消质押前验证
4. **先更新后转账**: 避免重入
5. **紧急退出**: 用户可以随时退出

### 改进方向

1. 时间锁定质押
2. 多级奖励
3. 提升机制
4. 推荐奖励
5. 复利
6. 惩罚机制(早期提款)

