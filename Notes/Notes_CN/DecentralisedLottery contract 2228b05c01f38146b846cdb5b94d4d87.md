# 去中心化彩票合约

本教程使用Chainlink VRF构建去中心化彩票合约,实现可证明公平的随机性。涵盖集成Chainlink VRF、管理彩票状态、处理参与、请求随机数,以及完成随机词以选择获胜者。

## 关键概念

### 彩票状态
```solidity
enum LOTTERY_STATE { OPEN, CLOSED, CALCULATING }

LOTTERY_STATE public lotteryState;
address payable[] public players;
address public recentWinner;
uint256 public entryFee;
```

### Chainlink VRF配置
```solidity
uint256 public subscriptionId;
bytes32 public keyHash;
uint32 public callbackGasLimit = 100000;
uint16 public requestConfirmations = 3;
uint32 public numWords = 1;
uint256 public latestRequestId;
```

### 构造函数
```solidity
constructor(
    address vrfCoordinator,
    uint256 _subscriptionId,
    bytes32 _keyHash,
    uint256 _entryFee
) VRFConsumerBaseV2Plus(vrfCoordinator) {
    subscriptionId = _subscriptionId;
    keyHash = _keyHash;
    entryFee = _entryFee;
    lotteryState = LOTTERY_STATE.CLOSED;
}
```

### 参与彩票
```solidity
function enter() public payable {
    require(lotteryState == LOTTERY_STATE.OPEN, "Lottery not open");
    require(msg.value >= entryFee, "Not enough ETH");
    players.push(payable(msg.sender));
}
```

### 开始彩票
```solidity
function startLottery() external onlyOwner {
    require(lotteryState == LOTTERY_STATE.CLOSED, "Can't start yet");
    lotteryState = LOTTERY_STATE.OPEN;
}
```

### 结束彩票并请求随机数
```solidity
function endLottery() external onlyOwner {
    require(lotteryState == LOTTERY_STATE.OPEN, "Lottery not open");
    lotteryState = LOTTERY_STATE.CALCULATING;
    
    VRFV2PlusClient.RandomWordsRequest memory req = VRFV2PlusClient.RandomWordsRequest({
        keyHash: keyHash,
        subId: subscriptionId,
        requestConfirmations: requestConfirmations,
        callbackGasLimit: callbackGasLimit,
        numWords: numWords,
        extraArgs: VRFV2PlusClient._argsToBytes(
            VRFV2PlusClient.ExtraArgsV1({nativePayment: false})
        )
    });
    
    latestRequestId = s_vrfCoordinator.requestRandomWords(req);
}
```

### 完成随机词(Chainlink回调)
```solidity
function fulfillRandomWords(uint256, uint256[] calldata randomWords) internal override {
    require(lotteryState == LOTTERY_STATE.CALCULATING, "Not ready to pick winner");
    
    uint256 winnerIndex = randomWords[0] % players.length;
    address payable winner = players[winnerIndex];
    recentWinner = winner;
    
    players = new address payable ;
    lotteryState = LOTTERY_STATE.CLOSED;
    
    (bool sent, ) = winner.call{value: address(this).balance}("");
    require(sent, "Failed to send ETH to winner");
}
```

### 获取参与者
```solidity
function getPlayers() external view returns (address payable[] memory) {
    return players;
}
```

## 核心知识点

### Chainlink VRF (可验证随机函数)

- **VRF**: 生成可证明公平的随机数
- **链下计算**: 随机数在链下生成
- **链上验证**: 可以在链上验证随机性
- **防篡改**: 无法被预测或操纵

### VRF流程

1. **请求随机数**: 调用 `requestRandomWords()`
2. **Chainlink处理**: VRF节点生成随机数
3. **回调**: Chainlink调用 `fulfillRandomWords()`
4. **使用随机数**: 选择获胜者

### 状态机模式

- `OPEN`: 可以参与
- `CALCULATING`: 正在请求随机数
- `CLOSED`: 彩票关闭

### 安全考虑

- 使用 `call` 发送ETH(推荐方式)
- 在发送ETH前重置状态
- 要求检查防止无效操作
- 只有所有者可以开始/结束彩票

### VRF参数

- `keyHash`: Gas lane标识符
- `subscriptionId`: Chainlink订阅ID
- `callbackGasLimit`: 回调函数的gas限制
- `requestConfirmations`: 等待的确认数
- `numWords`: 请求的随机数数量

