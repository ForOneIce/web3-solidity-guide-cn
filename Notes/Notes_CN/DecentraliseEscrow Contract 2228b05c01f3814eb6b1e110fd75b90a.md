# 去中心化托管合约

本教程实现了一个 `EnhancedSimpleEscrow` 合约,用于安全、无需信任的交易。它持有ETH直到确认交付,允许由仲裁者进行争议解决,包括买家取消的超时逻辑,并支持双方同意取消。使用枚举进行状态管理并发出详细事件。

## 关键概念

### 托管状态
```solidity
enum EscrowState {
    AWAITING_PAYMENT,
    AWAITING_DELIVERY,
    COMPLETE,
    DISPUTED,
    CANCELLED
}
```

### 状态变量
```solidity
address public buyer;
address public seller;
address public arbiter;
uint256 public amount;
EscrowState public state;
uint256 public depositTime;
uint256 public deliveryTimeout;
```

### 事件
```solidity
event PaymentDeposited(address indexed buyer, uint256 amount);
event DeliveryConfirmed(address indexed buyer, address indexed seller, uint256 amount);
event DisputeRaised(address indexed raisedBy);
event DisputeResolved(address indexed winner, uint256 amount);
event EscrowCancelled(address indexed canceledBy);
event DeliveryTimeoutReached(address indexed buyer);
event MutualCancellation(address indexed buyer, address indexed seller);
```

### 构造函数
```solidity
constructor(
    address _seller,
    address _arbiter,
    uint256 _deliveryTimeout
) {
    require(_seller != address(0), "Invalid seller");
    require(_arbiter != address(0), "Invalid arbiter");
    require(_deliveryTimeout > 0, "Invalid timeout");
    
    buyer = msg.sender;
    seller = _seller;
    arbiter = _arbiter;
    deliveryTimeout = _deliveryTimeout;
    state = EscrowState.AWAITING_PAYMENT;
}
```

### 阻止直接ETH转账
```solidity
receive() external payable {
    revert("Use deposit() function");
}
```

### 存款
```solidity
function deposit() external payable {
    require(msg.sender == buyer, "Only buyer");
    require(state == EscrowState.AWAITING_PAYMENT, "Wrong state");
    require(msg.value > 0, "Must send ETH");
    
    amount = msg.value;
    depositTime = block.timestamp;
    state = EscrowState.AWAITING_DELIVERY;
    
    emit PaymentDeposited(buyer, amount);
}
```

### 确认交付
```solidity
function confirmDelivery() external {
    require(msg.sender == buyer, "Only buyer");
    require(state == EscrowState.AWAITING_DELIVERY, "Wrong state");
    
    state = EscrowState.COMPLETE;
    
    payable(seller).transfer(amount);
    
    emit DeliveryConfirmed(buyer, seller, amount);
}
```

### 提出争议
```solidity
function raiseDispute() external {
    require(msg.sender == buyer || msg.sender == seller, "Only parties");
    require(state == EscrowState.AWAITING_DELIVERY, "Wrong state");
    
    state = EscrowState.DISPUTED;
    
    emit DisputeRaised(msg.sender);
}
```

### 解决争议
```solidity
function resolveDispute(address winner) external {
    require(msg.sender == arbiter, "Only arbiter");
    require(state == EscrowState.DISPUTED, "Not disputed");
    require(winner == buyer || winner == seller, "Invalid winner");
    
    state = EscrowState.COMPLETE;
    
    payable(winner).transfer(amount);
    
    emit DisputeResolved(winner, amount);
}
```

### 超时后取消
```solidity
function cancelAfterTimeout() external {
    require(msg.sender == buyer, "Only buyer");
    require(state == EscrowState.AWAITING_DELIVERY, "Wrong state");
    require(
        block.timestamp >= depositTime + deliveryTimeout,
        "Timeout not reached"
    );
    
    state = EscrowState.CANCELLED;
    
    payable(buyer).transfer(amount);
    
    emit DeliveryTimeoutReached(buyer);
    emit EscrowCancelled(buyer);
}
```

### 双方同意取消
```solidity
function cancelMutual() external {
    require(msg.sender == buyer || msg.sender == seller, "Only parties");
    require(state == EscrowState.AWAITING_DELIVERY, "Wrong state");
    
    // 简化版本: 任何一方都可以取消
    // 生产版本应该需要双方签名
    
    state = EscrowState.CANCELLED;
    
    payable(buyer).transfer(amount);
    
    emit MutualCancellation(buyer, seller);
    emit EscrowCancelled(msg.sender);
}
```

### 获取剩余时间
```solidity
function getTimeLeft() external view returns (uint256) {
    if (state != EscrowState.AWAITING_DELIVERY) {
        return 0;
    }
    
    uint256 deadline = depositTime + deliveryTimeout;
    if (block.timestamp >= deadline) {
        return 0;
    }
    
    return deadline - block.timestamp;
}
```

## 核心知识点

### 托管工作原理

1. **买家存款**: ETH锁定在合约中
2. **卖家交付**: 提供商品/服务
3. **买家确认**: 释放资金给卖家
4. **争议**: 如有问题,仲裁者决定

### 状态机

```
AWAITING_PAYMENT → AWAITING_DELIVERY → COMPLETE
                          ↓
                      DISPUTED → COMPLETE
                          ↓
                      CANCELLED
```

### 角色

- **买家**: 存入资金,确认交付,可取消
- **卖家**: 接收资金,可提出争议
- **仲裁者**: 解决争议的中立方

### 超时机制

```
如果 (当前时间 >= 存款时间 + 交付超时) {
    买家可以取消并获得退款
}
```

保护买家免受不交付的影响。

### 争议解决

- 买家或卖家可以提出争议
- 仲裁者决定谁获胜
- 获胜者获得全部资金

### 取消选项

#### 1. 超时取消
- 仅买家
- 超时后
- 自动退款

#### 2. 双方同意取消
- 买家或卖家(简化版本)
- 任何时候
- 退款给买家

### 安全特性

1. **状态检查**: 每个函数验证正确状态
2. **角色验证**: 只有授权方可以调用
3. **时间检查**: 超时逻辑
4. **防止直接转账**: `receive()` 回退
5. **先更新状态**: 防止重入

### 枚举优势

```solidity
enum EscrowState {
    AWAITING_PAYMENT,
    AWAITING_DELIVERY,
    COMPLETE,
    DISPUTED,
    CANCELLED
}
```

- 可读性强
- 类型安全
- 节省gas(vs 字符串)

### 改进方向

1. **多签取消**: 要求双方确认
2. **部分退款**: 争议中间解决方案
3. **多个里程碑**: 分阶段付款
4. **仲裁费用**: 支付仲裁者
5. **时间扩展**: 允许延长交付时间
6. **自动释放**: 超时后自动确认

