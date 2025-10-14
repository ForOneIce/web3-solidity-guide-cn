# Oracle Contract

# 🌍 What Are Oracles — and Why Do Smart Contracts Need Them?

So picture this:

You’ve built a smart contract that lives on the blockchain. It's tamper-proof, transparent, and runs exactly as coded. Amazing, right?

But here’s the catch: smart contracts **can’t access the outside world on their own**. They live in a sealed box, and they don’t know what the weather is like, what ETH is worth in USD, or whether a football match ended in a win or loss.

That’s where **oracles** come in.

> Think of an oracle like a trusted delivery guy who brings real-world data into the blockchain world.
> 

They act as bridges — securely feeding off-chain data (like prices, weather, or scores) into smart contracts so they can make informed decisions.

---

## 🔗 Enter Chainlink: The Most Popular Oracle Network

[Chainlink](https://chain.link/) is the gold standard for decentralized oracles. It offers APIs for price feeds, weather, randomness, and even entire data networks that are secure, tamper-proof, and widely used by DeFi projects.

Chainlink has standard interfaces like `AggregatorV3Interface` that let us easily integrate their data feeds into our smart contracts.

In our case, we want **rainfall data**. Now, while Chainlink doesn’t yet offer live rainfall feeds for every location, we’ll **build a mock weather oracle** that behaves *like* Chainlink — perfect for testing and learning.

Later, you can swap it out for a real Chainlink weather oracle once it becomes available.

---

# 🌧️ What Are We Building?

Imagine this:

- Farmers often lose crops due to drought (not enough rain).
- Crop insurance usually takes weeks or months to pay out — and has middlemen.
- What if a farmer could buy **blockchain-powered crop insurance** and get paid automatically if rainfall drops below a threshold?

That’s what we’re building:

1. **`MockWeatherOracle.sol`** – A simulated Chainlink-style oracle that randomly generates rainfall values.
2. **`CropInsurance.sol`** – A smart contract that:
    - Lets farmers pay a premium (in ETH),
    - Monitors the rainfall,
    - And automatically pays out if rainfall is too low.

Let’s break both contracts down, nice and easy.

---

# 🛰️ Mock Oracle — `MockWeatherOracle.sol`

# MockWeatherOracle.sol — Full Contract Code

```solidity

// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

import "@chainlink/contracts/src/v0.8/interfaces/AggregatorV3Interface.sol";
import "@openzeppelin/contracts/access/Ownable.sol";

contract MockWeatherOracle is AggregatorV3Interface, Ownable {
    uint8 private _decimals;
    string private _description;
    uint80 private _roundId;
    uint256 private _timestamp;
    uint256 private _lastUpdateBlock;

    constructor() Ownable(msg.sender) {
        _decimals = 0; // Rainfall in whole millimeters
        _description = "MOCK/RAINFALL/USD";
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

    function getRoundData(uint80 _roundId_)
        external
        view
        override
        returns (uint80 roundId, int256 answer, uint256 startedAt, uint256 updatedAt, uint80 answeredInRound)
    {
        return (_roundId_, _rainfall(), _timestamp, _timestamp, _roundId_);
    }

    function latestRoundData()
        external
        view
        override
        returns (uint80 roundId, int256 answer, uint256 startedAt, uint256 updatedAt, uint80 answeredInRound)
    {
        return (_roundId, _rainfall(), _timestamp, _timestamp, _roundId);
    }

    // Function to get current rainfall with random variation
    function _rainfall() public view returns (int256) {
        // Use block information to generate pseudo-random variation
        uint256 blocksSinceLastUpdate = block.number - _lastUpdateBlock;
        uint256 randomFactor = uint256(keccak256(abi.encodePacked(
            block.timestamp,
            block.coinbase,
            blocksSinceLastUpdate
        ))) % 1000; // Random number between 0 and 999

        // Return random rainfall between 0 and 999mm
        return int256(randomFactor);
    }

    // Function to update random rainfall
    function _updateRandomRainfall() private {
        _roundId++;
        _timestamp = block.timestamp;
        _lastUpdateBlock = block.number;
    }

    // Function to force update rainfall (anyone can call)
    function updateRandomRainfall() external {
        _updateRandomRainfall();
    }
}

```

---

# 🔍 Let’s Break It Down — Line by Line

---

## ✅ License & Version

```solidity

// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

```

- `SPDX-License-Identifier`: A required comment that tells the compiler what license this code uses. Here it’s MIT — a permissive, open-source license.
- `pragma solidity ^0.8.19;`: This line locks the contract to be compiled with Solidity version **0.8.19 or newer**, but **not 0.9.0 or higher**. It ensures compatibility and avoids breaking changes from future versions.

---

## 📦 Imports

```solidity

import "@chainlink/contracts/src/v0.8/interfaces/AggregatorV3Interface.sol";
import "@openzeppelin/contracts/access/Ownable.sol";

```

- **AggregatorV3Interface**: This is Chainlink's standard oracle interface — used to fetch data like price feeds or, in our case, mock rainfall.
- **Ownable**: A helper from OpenZeppelin that gives us ownership functionality — including an `owner()` and the `onlyOwner` modifier.

💡 Analogy: Think of `Ownable` as giving the deployer admin access — useful when you want to restrict who can do certain things (like updating weather data, if needed).

---

## 🏗️ Contract Declaration

```solidity

contract MockWeatherOracle is AggregatorV3Interface, Ownable {

```

We are creating a contract called `MockWeatherOracle`.

It:

- **Inherits** from `AggregatorV3Interface` — meaning it must implement functions like `latestRoundData()`.
- **Inherits** from `Ownable` — so we get ownership features for free.

---

## 🧮 State Variables

```solidity

uint8 private _decimals;
string private _description;
uint80 private _roundId;
uint256 private _timestamp;
uint256 private _lastUpdateBlock;

```

Let’s break down each one:

- `_decimals`: Defines the precision of the data. Ours is `0` because rainfall is in full millimeters (e.g., "542mm").
- `_description`: A text label for the feed (like a name).
- `_roundId`: Used to simulate different data update cycles (each new round is a new reading).
- `_timestamp`: Records when the last update occurred.
- `_lastUpdateBlock`: Tracks the block when the last update happened, used to add randomness.

---

## 🧱 Constructor

```solidity

constructor() Ownable(msg.sender) {
    _decimals = 0;
    _description = "MOCK/RAINFALL/USD";
    _roundId = 1;
    _timestamp = block.timestamp;
    _lastUpdateBlock = block.number;
}

```

This sets initial values when the contract is first deployed.

- `Ownable(msg.sender)` — Sets the deployer as the owner.
- `_decimals = 0` — Rainfall doesn't need decimals (like 542.67mm? Nah.)
- `_description` — Just a readable label.
- `_roundId` — Start with round 1.
- `_timestamp` and `_lastUpdateBlock` — Store current time/block to simulate freshness of data.

---

## 📥 Chainlink Interface Functions

### 1. `decimals()`

```solidity

function decimals() external view override returns (uint8) {
    return _decimals;
}

```

Chainlink requires this. It tells apps how many decimal places to expect. We return `0`.

---

### 2. `description()`

```solidity

function description() external view override returns (string memory) {
    return _description;
}

```

Gives a human-readable description of the feed.

---

### 3. `version()`

```solidity

function version() external pure override returns (uint256) {
    return 1;
}

```

Says this is version `1` of our mock. This is mostly informational.

---

## 🔁 Round Data Functions

### 1. `getRoundData()`

```solidity

function getRoundData(uint80 _roundId_)
    external
    view
    override
    returns (uint80 roundId, int256 answer, uint256 startedAt, uint256 updatedAt, uint80 answeredInRound)
{
    return (_roundId_, _rainfall(), _timestamp, _timestamp, _roundId_);
}

```

This mimics Chainlink’s standard function for accessing historical data.

It returns:

- The round ID you asked for
- A mock rainfall value
- The same timestamp twice (no distinction here for simplicity)
- The same round ID for `answeredInRound`

In a real oracle, `startedAt` and `updatedAt` might be different. We simplify that here.

---

### 2. `latestRoundData()`

```solidity

function latestRoundData()
    external
    view
    override
    returns (uint80 roundId, int256 answer, uint256 startedAt, uint256 updatedAt, uint80 answeredInRound)
{
    return (_roundId, _rainfall(), _timestamp, _timestamp, _roundId);
}

```

This is the most important function — apps use it to get the **latest data**.

We return:

- Current round ID
- Random rainfall value
- Timestamps
- Round ID confirmation

💡 This function is what the `CropInsurance` contract will call to get current rainfall.

---

## 🌧️ `_rainfall()` — Simulated Rain Generator

```solidity

function _rainfall() public view returns (int256) {
    uint256 blocksSinceLastUpdate = block.number - _lastUpdateBlock;
    uint256 randomFactor = uint256(keccak256(abi.encodePacked(
        block.timestamp,
        block.coinbase,
        blocksSinceLastUpdate
    ))) % 1000;

    return int256(randomFactor);
}

```

Here's the magic behind random rainfall:

1. We calculate how many blocks have passed since last update.
2. We combine:
    - `block.timestamp` — current time
    - `block.coinbase` — address of the miner (some entropy)
    - `blocksSinceLastUpdate`
3. All that is hashed using `keccak256`, a secure hashing function.
4. The result is converted to an integer between 0–999 using `% 1000`.

So every time you call this function, you get a new-ish pseudo-random rainfall value — between **0 and 999mm**.

> ⚠️ Note: This is not secure randomness. But it's totally fine for a mock oracle.
> 

---

## 📅 `_updateRandomRainfall()`

```solidity

function _updateRandomRainfall() private {
    _roundId++;
    _timestamp = block.timestamp;
    _lastUpdateBlock = block.number;
}

```

A helper function to:

- Increase the round (to simulate new data)
- Record when the new data was created

This would be done by a Chainlink node in real life. Here, we simulate it manually.

---

## 🔁 `updateRandomRainfall()`

```solidity

function updateRandomRainfall() external {
    _updateRandomRainfall();
}

```

This is the **public** function anyone can call to update the "oracle" data.

> You can call this from a UI button like “Fetch Latest Rainfall” — useful for testing or simulating a new day.
> 

---

# ✅ Summary: What Did We Learn?

- We created a fake weather data oracle that acts like a Chainlink data feed.
- It returns pseudo-random rainfall values to simulate real-world inputs.
- It implements all required functions of the `AggregatorV3Interface`.
- You can use this in place of a real Chainlink oracle for testing insurance, games, or any logic that reacts to rainfall.

---

# 🌾 Crop Insurance — `CropInsurance.sol`

This contract simulates a blockchain-based crop insurance program. Farmers can pay a small premium, and if the rainfall is below a threshold, they’re automatically paid out — no middlemen, no waiting.

---

## 📜 Full Contract

```solidity

// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

import "@chainlink/contracts/src/v0.8/interfaces/AggregatorV3Interface.sol";
import "@openzeppelin/contracts/access/Ownable.sol";

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
        uint256 ethPrice = getEthPrice();
        uint256 premiumInEth = (INSURANCE_PREMIUM_USD * 1e18) / ethPrice;

        require(msg.value >= premiumInEth, "Insufficient premium amount");
        require(!hasInsurance[msg.sender], "Already insured");

        hasInsurance[msg.sender] = true;
        emit InsurancePurchased(msg.sender, msg.value);
    }

    function checkRainfallAndClaim() external {
        require(hasInsurance[msg.sender], "No active insurance");
        require(block.timestamp >= lastClaimTimestamp[msg.sender] + 1 days, "Must wait 24h between claims");

        (
            uint80 roundId,
            int256 rainfall,
            ,
            uint256 updatedAt,
            uint80 answeredInRound
        ) = weatherOracle.latestRoundData();

        require(updatedAt > 0, "Round not complete");
        require(answeredInRound >= roundId, "Stale data");

        uint256 currentRainfall = uint256(rainfall);
        emit RainfallChecked(msg.sender, currentRainfall);

        if (currentRainfall < RAINFALL_THRESHOLD) {
            lastClaimTimestamp[msg.sender] = block.timestamp;
            emit ClaimSubmitted(msg.sender);

            uint256 ethPrice = getEthPrice();
            uint256 payoutInEth = (INSURANCE_PAYOUT_USD * 1e18) / ethPrice;

            (bool success, ) = msg.sender.call{value: payoutInEth}("");
            require(success, "Transfer failed");

            emit ClaimPaid(msg.sender, payoutInEth);
        }
    }

    function getEthPrice() public view returns (uint256) {
        (
            ,
            int256 price,
            ,
            ,
        ) = ethUsdPriceFeed.latestRoundData();

        return uint256(price);
    }

    function getCurrentRainfall() public view returns (uint256) {
        (
            ,
            int256 rainfall,
            ,
            ,
        ) = weatherOracle.latestRoundData();

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

---

## 🧱 Constructor

```solidity

constructor(address _weatherOracle, address _ethUsdPriceFeed) payable Ownable(msg.sender) {
    weatherOracle = AggregatorV3Interface(_weatherOracle);
    ethUsdPriceFeed = AggregatorV3Interface(_ethUsdPriceFeed);
}

```

**Line-by-line explanation:**

- `constructor(...)`: This special function runs **once** when the contract is deployed.
- `address _weatherOracle`: This is the address of our rainfall oracle (like the mock we built earlier).
- `address _ethUsdPriceFeed`: This is the address of a Chainlink price feed that gives us ETH → USD conversion.
- `Ownable(msg.sender)`: Initializes the contract owner as the person who deployed it.
- We save both oracle addresses for use in later functions.

---

## 💸 `purchaseInsurance()`

```solidity

function purchaseInsurance() external payable {
    uint256 ethPrice = getEthPrice();
    uint256 premiumInEth = (INSURANCE_PREMIUM_USD * 1e18) / ethPrice;

    require(msg.value >= premiumInEth, "Insufficient premium amount");
    require(!hasInsurance[msg.sender], "Already insured");

    hasInsurance[msg.sender] = true;
    emit InsurancePurchased(msg.sender, msg.value);
}

```

**Line-by-line explanation:**

- `external payable`: This function can receive ETH directly from the user.
- `getEthPrice()`: We fetch the current price of ETH in USD using Chainlink.
- `premiumInEth`: We convert our $10 premium into ETH (multiplied by `1e18` for wei precision).
- `require(msg.value >= ...)`: Checks if the user sent **enough** ETH.
- `require(!hasInsurance[msg.sender])`: Prevents users from buying insurance twice.
- `hasInsurance[msg.sender] = true`: Marks the user as insured.
- `emit InsurancePurchased(...)`: Emits an event the frontend can listen to.

---

## ⛅ `checkRainfallAndClaim()`

```solidity

function checkRainfallAndClaim() external {
    require(hasInsurance[msg.sender], "No active insurance");
    require(block.timestamp >= lastClaimTimestamp[msg.sender] + 1 days, "Must wait 24h between claims");

```

- This function is **only for insured users**.
- We enforce a **1-day cooldown** between claims to avoid spamming.

```solidity

    (
        uint80 roundId,
        int256 rainfall,
        ,
        uint256 updatedAt,
        uint80 answeredInRound
    ) = weatherOracle.latestRoundData();

```

- Pulls latest rainfall data from our weather oracle.
- We use **destructuring** to ignore unneeded values.

```solidity

    require(updatedAt > 0, "Round not complete");
    require(answeredInRound >= roundId, "Stale data");

```

- Basic checks to make sure the oracle data is fresh and valid.

```solidity

    uint256 currentRainfall = uint256(rainfall);
    emit RainfallChecked(msg.sender, currentRainfall);

```

- Converts rainfall to unsigned format.
- Emits an event so users/frontends can track the rainfall they’re evaluated against.

---

## 💵 Claim and Payout Logic

```solidity

    if (currentRainfall < RAINFALL_THRESHOLD) {
        lastClaimTimestamp[msg.sender] = block.timestamp;
        emit ClaimSubmitted(msg.sender);

```

- If rainfall is **below the drought threshold**, the claim process continues.
- Records the time to prevent back-to-back claims.
- Emits a `ClaimSubmitted` event.

```solidity

        uint256 ethPrice = getEthPrice();
        uint256 payoutInEth = (INSURANCE_PAYOUT_USD * 1e18) / ethPrice;

```

- Converts the $50 payout to ETH using live conversion rate.

```solidity

        (bool success, ) = msg.sender.call{value: payoutInEth}("");
        require(success, "Transfer failed");

```

- Transfers ETH to the farmer.
- We check that the transfer succeeded.

```solidity

        emit ClaimPaid(msg.sender, payoutInEth);
    }
}

```

- Emits a `ClaimPaid` event for tracking.

---

## 📈 Utility Functions

### `getEthPrice()`

```solidity

function getEthPrice() public view returns (uint256) {
    (, int256 price, , , ) = ethUsdPriceFeed.latestRoundData();
    return uint256(price);
}

```

### 🧠 What it does (in plain English):

- This function talks to **Chainlink**, which gives us the **latest ETH price in USD**.
- The `price` it gives back isn’t directly the actual value — it comes with **8 extra digits**.

---

### 📦 Example:

Let’s say the Chainlink feed returns:

```solidity

price = 254000000000

```

That’s **$2,540.00000000** — basically `$2,540` with 8 decimals at the end.

So to read it properly, just **divide the output by 100,000,000** (aka `1e8`):

```

Actual ETH price = 254000000000 / 100000000 = $2,540
```

---

### 💵 How do we use it?

Let’s say someone sends **0.1 ETH** and you want to know how much that is in USD.

---

### Step-by-step:

1 ETH = $2,540

0.1 ETH = 0.1 × 2,540 = **$254**

In the contract:

```solidity
ethAmount = 0.1 ether = 100000000000000000 wei  // 18 decimals
ethPrice  = 254000000000                       // 8 decimals

usdValue = (ethAmount * ethPrice) / 1e18
         = (100000000000000000 * 254000000000) / 1e18
         = 25400000000
```

Now divide that by `1e8` (outside Solidity) to get:

```

usdValue = 25400000000 / 1e8 = $254
```

### `getCurrentRainfall()`

```solidity

function getCurrentRainfall() public view returns (uint256) {
    (, int256 rainfall, , , ) = weatherOracle.latestRoundData();
    return uint256(rainfall);
}

```

- Lets anyone view current rainfall — useful for dashboards or explorers.

---

## 🔐 Admin + Fallback

```solidity

function withdraw() external onlyOwner {
    payable(owner()).transfer(address(this).balance);
}

```

- Lets the contract owner withdraw all collected ETH (e.g., unused premiums)

```solidity

receive() external payable {}

```

- This function allows the contract to receive ETH **without calling a function**.

```solidity

function getBalance() public view returns (uint256) {
    return address(this).balance;
}

```

- Allows anyone to check how much ETH the contract currently holds.

---

# 🧪 How To Use It (Testing Workflow)

1. Deploy `MockWeatherOracle`
2. Deploy `CropInsurance` with:
    - Weather oracle address
    - Chainlink ETH/USD oracle address (or another mock)
3. Call `updateRandomRainfall()` to simulate weather
4. Use `purchaseInsurance()` to pay premium
5. Call `checkRainfallAndClaim()` and see if payout happens

---

# 🎉 Wrap Up

You now understand:

- What oracles are and how Chainlink connects smart contracts to the real world.
- How to simulate an oracle for testing.
- How to build a fully functional insurance dApp using Solidity and Chainlink-style feeds.
- Why events, `msg.sender`, mappings, and safe transfers are essential in dApp development.

Let me know if you want the frontend walkthrough, Chainlink mock deployment script, or test coverage using Hardhat or Foundry.