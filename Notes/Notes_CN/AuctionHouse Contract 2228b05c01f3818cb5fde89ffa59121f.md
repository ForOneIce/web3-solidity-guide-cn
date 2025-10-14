# 拍卖行合约

本教程构建了一个完整的拍卖系统,涵盖数据需求、构造函数设置、出价、结束拍卖和查看结果。介绍了像 `msg.sender` 和 `block.timestamp` 这样的全局变量。

## 关键概念

### 状态变量
```solidity
address public owner;
string public item;
uint public auctionEndTime;
address private highestBidder;
uint private highestBid;
bool public ended;

mapping(address => uint) public bids;
address[] public bidders;
```

### 构造函数
```solidity
constructor(string memory _item, uint _biddingTime) {
    owner = msg.sender;
    item = _item;
    auctionEndTime = block.timestamp + _biddingTime;
}
```

### 出价功能
```solidity
function bid(uint amount) external {
    require(block.timestamp < auctionEndTime, "Auction ended");
    require(amount > highestBid, "Bid too low");
    
    if (bids[msg.sender] == 0) {
        bidders.push(msg.sender);
    }
    
    bids[msg.sender] += amount;
    
    if (bids[msg.sender] > highestBid) {
        highestBid = bids[msg.sender];
        highestBidder = msg.sender;
    }
}
```

### 结束拍卖
```solidity
function endAuction() external {
    require(!ended, "Auction already ended");
    require(block.timestamp >= auctionEndTime, "Auction not yet ended");
    require(msg.sender == owner, "Only owner can end");
    
    ended = true;
}
```

### 查看获胜者
```solidity
function getWinner() external view returns (address, uint) {
    require(ended, "Auction not ended");
    return (highestBidder, highestBid);
}
```

### 获取所有出价者
```solidity
function getAllBidders() external view returns (address[] memory) {
    return bidders;
}
```

## 核心知识点

- `block.timestamp` 返回当前区块时间戳
- `msg.sender` 是调用函数的地址
- `require()` 用于验证条件,失败时回退交易
- 映射和数组可以结合使用来跟踪参与者
- `private` 变量不能从外部访问

