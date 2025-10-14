# 预言机合约

本教程解释了什么是预言机,以及Chainlink如何将智能合约连接到真实世界数据。构建了一个模拟天气预言机和一个使用它根据降雨数据自动支付的农作物保险合约。

## 关键概念

### 什么是预言机?

预言机是连接区块链和外部数据源的桥梁。Chainlink是最流行的去中心化预言机网络。

### 模拟天气预言机
```solidity
import "@chainlink/contracts/src/v0.8/interfaces/AggregatorV3Interface.sol";
import "@openzeppelin/contracts/access/Ownable.sol";

contract MockWeatherOracle is AggregatorV3Interface, Ownable {
    uint8 private _decimals;
    string private _description;
    uint80 private _roundId;
    uint256 private _timestamp;
    uint256 private _lastUpdateBlock;
    
    constructor() Ownable(msg.sender) {
        _decimals = 0;
        _description = "Mock Rainfall Oracle";
        _roundId = 1;
        _timestamp = block.timestamp;
        _lastUpdateBlock = block.number;
    }
    
    function decimals() external view override returns (uint8) {
        return _decimals;
    }
    
    function description() external view override returns (string memory) {
        return _description;
    }
    
    function version() external pure override returns (uint256) {
        return 1;
    }
    
    function getRoundData(uint80 _roundId_) external view override returns (
        uint80 roundId,
        int256 answer,
        uint256 startedAt,
        uint256 updatedAt,
        uint80 answeredInRound
    ) {
        return (_roundId, _rainfall(), _timestamp, _timestamp, _roundId);
    }
    
    function latestRoundData() external view override returns (
        uint80 roundId,
        int256 answer,
        uint256 startedAt,
        uint256 updatedAt,
        uint80 answeredInRound
    ) {
        return (_roundId, _rainfall(), _timestamp, _timestamp, _roundId);
    }
    
    function _rainfall() public view returns (int256) {
        uint256 blocksSinceLastUpdate = block.number - _lastUpdateBlock;
        uint256 pseudoRandom = uint256(keccak256(abi.encodePacked(
            block.timestamp,
            block.coinbase,
            blocksSinceLastUpdate
        ))) % 1000;
        return int256(pseudoRandom);
    }
    
    function _updateRandomRainfall() private {
        _roundId++;
        _timestamp = block.timestamp;
        _lastUpdateBlock = block.number;
    }
    
    function updateRandomRainfall() external {
        _updateRandomRainfall();
    }
}
```

### 农作物保险合约
```solidity
contract CropInsurance is Ownable {
    AggregatorV3Interface private weatherOracle;
    AggregatorV3Interface private ethUsdPriceFeed;
    
    uint256 public constant RAINFALL_THRESHOLD = 500;
    uint256 public constant INSURANCE_PREMIUM_USD = 10;
    uint256 public constant INSURANCE_PAYOUT_USD = 50;
    
    mapping(address => bool) public hasInsurance;
    mapping(address => uint256) public lastClaimTimestamp;
    
    event InsurancePurchased(address indexed farmer, uint256 amount);
    event ClaimSubmitted(address indexed farmer);
    event ClaimPaid(address indexed farmer, uint256 amount);
    event RainfallChecked(address indexed farmer, uint256 rainfall);
    
    constructor(address _weatherOracle, address _ethUsdPriceFeed) payable Ownable(msg.sender) {
        weatherOracle = AggregatorV3Interface(_weatherOracle);
        ethUsdPriceFeed = AggregatorV3Interface(_ethUsdPriceFeed);
    }
    
    function purchaseInsurance() external payable {
        require(!hasInsurance[msg.sender], "Already insured");
        
        uint256 ethPrice = getEthPrice();
        uint256 premiumInEth = (INSURANCE_PREMIUM_USD * 1e18) / ethPrice;
        
        require(msg.value >= premiumInEth, "Insufficient premium");
        
        hasInsurance[msg.sender] = true;
        emit InsurancePurchased(msg.sender, msg.value);
    }
    
    function checkRainfallAndClaim() external {
        require(hasInsurance[msg.sender], "No insurance");
        require(block.timestamp >= lastClaimTimestamp[msg.sender] + 1 days, 
                "Must wait 24h between claims");
        
        (, int256 rainfall, , , ) = weatherOracle.latestRoundData();
        uint256 currentRainfall = uint256(rainfall);
        
        emit RainfallChecked(msg.sender, currentRainfall);
        
        if (currentRainfall < RAINFALL_THRESHOLD) {
            uint256 ethPrice = getEthPrice();
            uint256 payoutInEth = (INSURANCE_PAYOUT_USD * 1e18) / ethPrice;
            
            require(address(this).balance >= payoutInEth, "Insufficient funds");
            
            lastClaimTimestamp[msg.sender] = block.timestamp;
            
            (bool success, ) = msg.sender.call{value: payoutInEth}("");
            require(success, "Payment failed");
            
            emit ClaimPaid(msg.sender, payoutInEth);
        } else {
            emit ClaimSubmitted(msg.sender);
        }
    }
    
    function getEthPrice() public view returns (uint256) {
        (, int256 price, , , ) = ethUsdPriceFeed.latestRoundData();
        return uint256(price);
    }
    
    function getCurrentRainfall() public view returns (uint256) {
        (, int256 rainfall, , , ) = weatherOracle.latestRoundData();
        return uint256(rainfall);
    }
    
    function withdraw() external onlyOwner {
        payable(owner()).transfer(address(this).balance);
    }
    
    receive() external payable {}
    
    function getBalance() public view returns (uint256) {
        return address(this).balance;
    }
}
```

## 核心知识点

- **预言机**: 连接区块链和外部数据
- **Chainlink**: 去中心化预言机网络
- `AggregatorV3Interface`: Chainlink价格数据接口
- `latestRoundData()`: 获取最新数据
- **模拟随机性**: 使用 `keccak256` 和区块数据
- **价格转换**: USD到ETH的计算
- **时间锁**: 防止频繁索赔
- 智能合约可以基于真实世界数据自动执行

