# NFT市场合约

本教程开发了一个完全链上的NFT市场合约。允许用户以自定义价格和版税挂单出售NFT,通过自动手续费和版税分配购买NFT,取消挂单,以及所有者更新手续费设置。集成了 `IERC721` 和 `ReentrancyGuard`。

## 关键概念

### 状态变量
```solidity
address public owner;
uint256 public marketplaceFeePercent;
address public feeRecipient;

struct Listing {
    address seller;
    address nftContract;
    uint256 tokenId;
    uint256 price;
    uint256 royaltyPercent;
}

mapping(bytes32 => Listing) public listings;
```

### 事件
```solidity
event Listed(
    bytes32 indexed listingId,
    address indexed seller,
    address indexed nftContract,
    uint256 tokenId,
    uint256 price,
    uint256 royaltyPercent
);

event Purchase(
    bytes32 indexed listingId,
    address indexed buyer,
    address indexed seller,
    uint256 price
);

event Unlisted(bytes32 indexed listingId);
event FeeUpdated(uint256 newFeePercent);
```

### 构造函数
```solidity
constructor(uint256 _feePercent, address _feeRecipient) {
    owner = msg.sender;
    marketplaceFeePercent = _feePercent;
    feeRecipient = _feeRecipient;
}
```

### 修饰符
```solidity
modifier onlyOwner() {
    require(msg.sender == owner, "Not owner");
    _;
}
```

### 更新手续费
```solidity
function setMarketplaceFeePercent(uint256 _newFeePercent) external onlyOwner {
    require(_newFeePercent <= 100, "Fee too high");
    marketplaceFeePercent = _newFeePercent;
    emit FeeUpdated(_newFeePercent);
}

function setFeeRecipient(address _newRecipient) external onlyOwner {
    require(_newRecipient != address(0), "Invalid address");
    feeRecipient = _newRecipient;
}
```

### 挂单NFT
```solidity
function listNFT(
    address nftContract,
    uint256 tokenId,
    uint256 price,
    uint256 royaltyPercent
) external {
    require(price > 0, "Price must be > 0");
    require(royaltyPercent <= 100, "Royalty too high");
    
    IERC721 nft = IERC721(nftContract);
    require(nft.ownerOf(tokenId) == msg.sender, "Not owner");
    require(
        nft.getApproved(tokenId) == address(this) || 
        nft.isApprovedForAll(msg.sender, address(this)),
        "Not approved"
    );
    
    bytes32 listingId = keccak256(abi.encodePacked(nftContract, tokenId, msg.sender));
    
    listings[listingId] = Listing({
        seller: msg.sender,
        nftContract: nftContract,
        tokenId: tokenId,
        price: price,
        royaltyPercent: royaltyPercent
    });
    
    emit Listed(listingId, msg.sender, nftContract, tokenId, price, royaltyPercent);
}
```

### 购买NFT
```solidity
function buyNFT(bytes32 listingId) external payable nonReentrant {
    Listing memory listing = listings[listingId];
    require(listing.price > 0, "Not listed");
    require(msg.value == listing.price, "Incorrect price");
    
    delete listings[listingId];
    
    // 计算费用分配
    uint256 marketplaceFee = (listing.price * marketplaceFeePercent) / 100;
    uint256 royaltyFee = (listing.price * listing.royaltyPercent) / 100;
    uint256 sellerProceeds = listing.price - marketplaceFee - royaltyFee;
    
    // 转移NFT
    IERC721(listing.nftContract).transferFrom(
        listing.seller,
        msg.sender,
        listing.tokenId
    );
    
    // 分配付款
    if (marketplaceFee > 0) {
        payable(feeRecipient).transfer(marketplaceFee);
    }
    
    if (royaltyFee > 0) {
        payable(listing.seller).transfer(royaltyFee);
    }
    
    payable(listing.seller).transfer(sellerProceeds);
    
    emit Purchase(listingId, msg.sender, listing.seller, listing.price);
}
```

### 取消挂单
```solidity
function cancelListing(bytes32 listingId) external {
    Listing memory listing = listings[listingId];
    require(listing.seller == msg.sender, "Not seller");
    
    delete listings[listingId];
    emit Unlisted(listingId);
}
```

### 获取挂单
```solidity
function getListing(bytes32 listingId) external view returns (
    address seller,
    address nftContract,
    uint256 tokenId,
    uint256 price,
    uint256 royaltyPercent
) {
    Listing memory listing = listings[listingId];
    return (
        listing.seller,
        listing.nftContract,
        listing.tokenId,
        listing.price,
        listing.royaltyPercent
    );
}
```

### Fallback函数
```solidity
receive() external payable {
    revert("Direct payments not accepted");
}

fallback() external payable {
    revert("Invalid function call");
}
```

## 核心知识点

### NFT市场机制

1. **挂单**: 卖家授权市场并列出NFT
2. **购买**: 买家发送ETH,获得NFT
3. **费用**: 平台收取手续费
4. **版税**: 创作者获得版税

### 费用分配

```
总价 = 100 ETH
├─ 平台费 (2%) = 2 ETH → 平台
├─ 版税 (10%) = 10 ETH → 卖家(创作者)
└─ 卖家收入 = 88 ETH → 卖家
```

### 批准流程

买家必须先批准市场合约:
```solidity
nft.approve(marketplaceAddress, tokenId);
// 或
nft.setApprovalForAll(marketplaceAddress, true);
```

### ListingID生成

```solidity
bytes32 listingId = keccak256(abi.encodePacked(
    nftContract,
    tokenId,
    seller
));
```

唯一标识每个挂单。

### 安全措施

1. **ReentrancyGuard**: 防止重入攻击
2. **删除挂单**: 购买前删除,防止双重消费
3. **所有权检查**: 验证卖家拥有NFT
4. **批准检查**: 确保市场可以转移NFT
5. **价格验证**: 确保付款正确

### IERC721集成

```solidity
IERC721 nft = IERC721(nftContract);
nft.ownerOf(tokenId);
nft.getApproved(tokenId);
nft.isApprovedForAll(owner, operator);
nft.transferFrom(from, to, tokenId);
```

### 改进方向

1. 支持ERC-1155
2. 拍卖功能
3. 批量购买
4. 出价系统
5. 白名单NFT集合
6. 元数据缓存

