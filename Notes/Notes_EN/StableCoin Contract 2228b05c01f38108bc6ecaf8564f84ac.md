# StableCoin Contract

Alright, builder —

Take a second and look back at how far you’ve come.

In just a few days, you’ve gone from writing simple variables…

to creating **your own ERC-20 tokens**,

**locking treasure**,

**setting up access controls**,

**guarding contracts against attacks**,

and even **handling money safely** inside smart contracts.

**You didn’t just read about Solidity — you *built with it*.**

And now, you're ready for the next step.

Today, we’re moving from "toy contracts" to **building a real-world, essential tool** that powers the biggest apps in crypto:

> Stablecoins.
> 

---

## 🌍 Why Stablecoins Matter So Much

Imagine trying to live your daily life —

but the value of the money in your wallet keeps *jumping* 30% up or down every few hours.

- Yesterday, your coffee cost $3.
- Today, it’s $6.
- Tomorrow, maybe it’s $1.50.

That’s how wild cryptocurrencies like Bitcoin and Ethereum can be.

**Fun for investors?** Maybe.

**Practical for payments?** Absolutely not.

That’s where **stablecoins** come in.

🔹 They're like an anchor in the stormy seas of crypto.

🔹 They're designed to stay *stable*, typically pegged 1:1 to a major currency like the **US Dollar**.

🔹 They're the silent heroes behind **trading**, **saving**, **borrowing**, **lending**, and **everyday transactions** in Web3.

In fact, if you look around DeFi today —

**almost every dApp**, every NFT marketplace, every lending pool, every DEX —

they all rely heavily on stablecoins behind the scenes.

No stablecoins?

The entire DeFi world would feel like trying to shop in a market where prices change every five seconds.

---

## 🛠️ But Aren’t Real Stablecoins Super Complicated?

Yes — absolutely.

Real-world stablecoins like USDC, DAI, and USDT are **incredibly complex systems**.

They involve:

- Collateralized reserves
- On-chain and off-chain oracles
- Dynamic supply adjustments
- Governance votes
- Full audits and legal compliance

And that's just scratching the surface!

**But here’s the thing:**

You don’t need to dive into that complexity to *understand* how the core idea works.

---

## 🚀 What We’re Building Today

Today, we’re building your very own **SimpleStablecoin**.

A **beginner-friendly** version.

**Carefully designed** to strip away all the overwhelming complexity —

and **zoom in** on what actually matters:

- How do you **mint** stablecoins?
- How do you **redeem** them?
- How do you **back** them with **real collateral**?
- How do you **calculate** safe margins and protect the system?

👉 This is not a production-grade stablecoin.

👉 This is **your training ground** —

a clean, understandable playground where you’ll learn the *mechanics* that power the biggest stablecoins in the world.

By the time we’re done, you'll not only know how a stablecoin contract is built —

you’ll have written your **own**.

And trust me —

**That’s a big leap toward becoming a real Web3 developer.**

---

**Ready?**

Let’s dive into the contract — and start unpacking how your SimpleStablecoin actually works!

---

# 🧠 Quick Overview: How Our SimpleStablecoin Works

Before we dive into the code, here’s a quick overview of what’s happening behind the scenes:

Users will deposit a trusted collateral token (like ETH, USDC, or any ERC-20 token we decide), and based on the latest price fetched from a price feed, the contract calculates how much stablecoin they can mint. To keep the system safe, we make sure that users always deposit *more* collateral than the value of stablecoins they’re getting — thanks to a 150% collateralization ratio. Later, when someone wants their collateral back, they can redeem their stablecoins by burning them. The contract also includes basic owner controls (like updating the price feed or collateralization settings) and protects against common vulnerabilities like reentrancy attacks.

This is a **beginner-friendly** version — built to focus purely on the core mechanics like **minting**, **redeeming**, and **collateral management**, so you can really understand how a stablecoin works under the hood without getting overwhelmed by production-level complexity.

---

# 📝 Here’s the Full Contract

Now that you know the high-level idea,

let’s jump into the code and see how everything fits together!

👇

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import "@openzeppelin/contracts/access/Ownable.sol";
import "@openzeppelin/contracts/security/ReentrancyGuard.sol";
import "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";
import "@openzeppelin/contracts/access/AccessControl.sol";
import "@openzeppelin/contracts/interfaces/IERC20Metadata.sol";
import "@chainlink/contracts/src/v0.8/interfaces/AggregatorV3Interface.sol";

contract SimpleStablecoin is ERC20, Ownable, ReentrancyGuard, AccessControl {
    using SafeERC20 for IERC20;

    bytes32 public constant PRICE_FEED_MANAGER_ROLE = keccak256("PRICE_FEED_MANAGER_ROLE");
    IERC20 public immutable collateralToken;
    uint8 public immutable collateralDecimals;
    AggregatorV3Interface public priceFeed;
    uint256 public collateralizationRatio = 150; // Expressed as a percentage (150 = 150%)

    event Minted(address indexed user, uint256 amount, uint256 collateralDeposited);
    event Redeemed(address indexed user, uint256 amount, uint256 collateralReturned);
    event PriceFeedUpdated(address newPriceFeed);
    event CollateralizationRatioUpdated(uint256 newRatio);

    error InvalidCollateralTokenAddress();
    error InvalidPriceFeedAddress();
    error MintAmountIsZero();
    error InsufficientStablecoinBalance();
    error CollateralizationRatioTooLow();

    constructor(
        address _collateralToken,
        address _initialOwner,
        address _priceFeed
    ) ERC20("Simple USD Stablecoin", "sUSD") Ownable(_initialOwner) {
        if (_collateralToken == address(0)) revert InvalidCollateralTokenAddress();
        if (_priceFeed == address(0)) revert InvalidPriceFeedAddress();

        collateralToken = IERC20(_collateralToken);
        collateralDecimals = IERC20Metadata(_collateralToken).decimals();
        priceFeed = AggregatorV3Interface(_priceFeed);

        _grantRole(DEFAULT_ADMIN_ROLE, _initialOwner);
        _grantRole(PRICE_FEED_MANAGER_ROLE, _initialOwner);
    }

    function getCurrentPrice() public view returns (uint256) {
        (, int256 price, , , ) = priceFeed.latestRoundData();
        require(price > 0, "Invalid price feed response");
        return uint256(price);
    }

    function mint(uint256 amount) external nonReentrant {
        if (amount == 0) revert MintAmountIsZero();

        uint256 collateralPrice = getCurrentPrice();
        uint256 requiredCollateralValueUSD = amount * (10 ** decimals()); // 18 decimals assumed for sUSD
        uint256 requiredCollateral = (requiredCollateralValueUSD * collateralizationRatio) / (100 * collateralPrice);
        uint256 adjustedRequiredCollateral = (requiredCollateral * (10 ** collateralDecimals)) / (10 ** priceFeed.decimals());

        collateralToken.safeTransferFrom(msg.sender, address(this), adjustedRequiredCollateral);
        _mint(msg.sender, amount);

        emit Minted(msg.sender, amount, adjustedRequiredCollateral);
    }

    function redeem(uint256 amount) external nonReentrant {
        if (amount == 0) revert MintAmountIsZero();
        if (balanceOf(msg.sender) < amount) revert InsufficientStablecoinBalance();

        uint256 collateralPrice = getCurrentPrice();
        uint256 stablecoinValueUSD = amount * (10 ** decimals());
        uint256 collateralToReturn = (stablecoinValueUSD * 100) / (collateralizationRatio * collateralPrice);
        uint256 adjustedCollateralToReturn = (collateralToReturn * (10 ** collateralDecimals)) / (10 ** priceFeed.decimals());

        _burn(msg.sender, amount);
        collateralToken.safeTransfer(msg.sender, adjustedCollateralToReturn);

        emit Redeemed(msg.sender, amount, adjustedCollateralToReturn);
    }

    function setCollateralizationRatio(uint256 newRatio) external onlyOwner {
        if (newRatio < 100) revert CollateralizationRatioTooLow();
        collateralizationRatio = newRatio;
        emit CollateralizationRatioUpdated(newRatio);
    }

    function setPriceFeedContract(address _newPriceFeed) external onlyRole(PRICE_FEED_MANAGER_ROLE) {
        if (_newPriceFeed == address(0)) revert InvalidPriceFeedAddress();
        priceFeed = AggregatorV3Interface(_newPriceFeed);
        emit PriceFeedUpdated(_newPriceFeed);
    }

    function getRequiredCollateralForMint(uint256 amount) public view returns (uint256) {
        if (amount == 0) return 0;

        uint256 collateralPrice = getCurrentPrice();
        uint256 requiredCollateralValueUSD = amount * (10 ** decimals());
        uint256 requiredCollateral = (requiredCollateralValueUSD * collateralizationRatio) / (100 * collateralPrice);
        uint256 adjustedRequiredCollateral = (requiredCollateral * (10 ** collateralDecimals)) / (10 ** priceFeed.decimals());

        return adjustedRequiredCollateral;
    }

    function getCollateralForRedeem(uint256 amount) public view returns (uint256) {
        if (amount == 0) return 0;

        uint256 collateralPrice = getCurrentPrice();
        uint256 stablecoinValueUSD = amount * (10 ** decimals());
        uint256 collateralToReturn = (stablecoinValueUSD * 100) / (collateralizationRatio * collateralPrice);
        uint256 adjustedCollateralToReturn = (collateralToReturn * (10 ** collateralDecimals)) / (10 ** priceFeed.decimals());

        return adjustedCollateralToReturn;
    }
    
}

```

---

# 📦 Imports in `SimpleStablecoin.sol`

Before we dive into the logic of our stablecoin, we first pull in a few powerful libraries and interfaces. These imports bring in battle-tested tools for token behavior, ownership, security, role management — and for the first time, live price data from Chainlink.

Here’s everything we’re bringing into our contract:

---

```solidity
 
import "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import "@openzeppelin/contracts/access/Ownable.sol";
import "@openzeppelin/contracts/security/ReentrancyGuard.sol";
import "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";
import "@openzeppelin/contracts/access/AccessControl.sol";
import "@openzeppelin/contracts/interfaces/IERC20Metadata.sol";
import "@chainlink/contracts/src/v0.8/interfaces/AggregatorV3Interface.sol";

```

---

# 🧩 Breaking it Down

We start by importing **ERC20** from OpenZeppelin, which gives us all the standard token functionality — minting, transferring, and managing balances — without writing it from scratch.

Then, **Ownable** steps in to introduce a simple ownership system. Sensitive settings, like updating collateral ratios, should only be changed by the contract owner, and this module handles that for us.

**ReentrancyGuard** is imported next to protect important functions like `mint` and `redeem` from a type of attack called "reentrancy", where a malicious contract tries to repeatedly call a function before the first call finishes.

**SafeERC20** is a safety net for dealing with other ERC-20 tokens. Sometimes, token contracts behave badly (like failing transfers without errors), and SafeERC20 ensures all token operations either complete successfully or fail cleanly.

We also bring in **AccessControl**, which allows the contract to define custom roles. In this stablecoin, we have a special **Price Feed Manager** role — so specific accounts can update the price feed without having total admin control.

**IERC20Metadata** is there to help us fetch extra information about the collateral token, like how many decimals it uses. Different tokens have different decimal setups, and it’s important our math accounts for that.

Finally, and most importantly, we import **AggregatorV3Interface** from Chainlink.

This is what lets our stablecoin **see the real-world price** of the collateral token.

Instead of guessing or relying on manual updates, the contract can automatically fetch trusted, decentralized price data — keeping the system accurate, up-to-date, and secure against manipulation.

---

# 🏗️ Contract Declaration

After setting up all our tools through imports, we now officially start building our contract.

Here’s the very first line that kicks everything off:

---

```solidity
 
contract SimpleStablecoin is ERC20, Ownable, ReentrancyGuard, AccessControl {

```

---

# 🧩 Breaking it Down

We declare our contract and give it a name: **SimpleStablecoin**.

But we’re not building everything from scratch — we’re **extending** four powerful OpenZeppelin contracts to inherit ready-to-use features:

- **ERC20** — This gives our stablecoin all the basic token behaviors, like transferring tokens, approving spenders, and checking balances.
- **Ownable** — Adds a simple ownership model, so we can control who has permission to perform sensitive operations like updating system settings.
- **ReentrancyGuard** — Protects critical functions from reentrancy attacks, making operations like minting and redeeming much safer.
- **AccessControl** — Allows us to define custom roles (like a Price Feed Manager), and finely control who can call certain functions.

By inheriting all of these at once, we build a strong foundation where security, access control, and token behavior are already professionally handled — and we can focus purely on the stablecoin-specific logic.

---

# 

# 🏗️ State Variables

Now that the contract has been declared, we move on to setting up the **key variables** that define how our stablecoin will work internally.

These variables will store everything from the collateral token to the current price feed, making sure our system knows what assets it’s managing and under what rules.

Here’s the code:

---

```solidity
 
using SafeERC20 for IERC20;

bytes32 public constant PRICE_FEED_MANAGER_ROLE = keccak256("PRICE_FEED_MANAGER_ROLE");
IERC20 public immutable collateralToken;
uint8 public immutable collateralDecimals;
AggregatorV3Interface public priceFeed;
uint256 public collateralizationRatio = 150; // Expressed as a percentage (150 = 150%)

```

---

# 🧩 Breaking it Down

---

```solidity
 
using SafeERC20 for IERC20;

```

This line activates **SafeERC20** for all `IERC20` tokens we interact with.

It automatically wraps methods like `transferFrom` and `transfer` with safety checks — making sure token operations either succeed properly or revert safely if anything goes wrong.

---

```solidity
bytes32 public constant PRICE_FEED_MANAGER_ROLE = keccak256("PRICE_FEED_MANAGER_ROLE");

```

We create a special role called `PRICE_FEED_MANAGER_ROLE`, which controls who can update the price feed.

You’ll notice it’s stored as a `bytes32` `constant`.
Here’s why:

`bytes32` is used because role identifiers in AccessControl are always expected to be exactly 32 bytes — it’s a standardized format that makes role checking fast and efficient inside the Ethereum Virtual Machine (EVM).

constant means this value never changes once deployed. It’s hardcoded into the contract forever, making it cheaper on gas and safer — there’s no chance of someone accidentally (or maliciously) modifying the role identifier later.

We generate it using `keccak256("PRICE_FEED_MANAGER_ROLE")` so that the role has a unique, cryptographically strong identifier instead of just using a plain string.

---

```solidity
 
IERC20 public immutable collateralToken;

```

This variable stores the **address of the ERC-20 token** users must deposit as collateral.

By making it `immutable`, we ensure it can only be set once — at the time of deployment — and can’t ever be changed later, adding an extra layer of safety.

---

```solidity
 
uint8 public immutable collateralDecimals;

```

Different ERC-20 tokens can have different numbers of decimals (e.g., USDC uses 6, ETH uses 18).

We store the collateral token’s decimals here when the contract is deployed, so later calculations for minting and redeeming stay accurate.

---

```solidity
 
AggregatorV3Interface public priceFeed;

```

This variable points to the **Chainlink price feed** contract.

We’ll use this to fetch the real-time price of our collateral token whenever someone mints or redeems stablecoins.

---

```solidity
 
uint256 public collateralizationRatio = 150; // Expressed as a percentage (150 = 150%)

```

This sets the **collateralization ratio** — meaning users must always deposit **150%** worth of collateral for the stablecoins they mint.

It’s a built-in safety buffer to protect the system against sudden drops in collateral value.

---

# 📣 Events

Smart contracts can’t just "talk" to a website or a user directly.

Instead, they **emit events** — little signals that are recorded on the blockchain.

Apps like frontends, block explorers, and wallets can listen to these events and show users what’s happening in real time.

Here’s the set of events we’re emitting from our SimpleStablecoin:

---

```solidity
 
event Minted(address indexed user, uint256 amount, uint256 collateralDeposited);
event Redeemed(address indexed user, uint256 amount, uint256 collateralReturned);
event PriceFeedUpdated(address newPriceFeed);
event CollateralizationRatioUpdated(uint256 newRatio);

```

---

# 🧩 Breaking it Down

---

```solidity
 
event Minted(address indexed user, uint256 amount, uint256 collateralDeposited);

```

This event fires whenever someone successfully mints new stablecoins.

It records:

- **Who minted** (`user`)
- **How many stablecoins** they received (`amount`)
- **How much collateral** they deposited (`collateralDeposited`)

The `indexed` keyword on `user` allows you to easily filter and search for all minting actions by a specific user address on the blockchain.

---

```solidity
 
event Redeemed(address indexed user, uint256 amount, uint256 collateralReturned);

```

This one triggers whenever someone redeems stablecoins back for collateral.

It tells us:

- **Who redeemed** (`user`)
- **How much stablecoin** they burned (`amount`)
- **How much collateral** they got back (`collateralReturned`)

Again, `user` is `indexed` so that frontend apps can quickly show a user’s personal redemption history.

---

```solidity
 
event PriceFeedUpdated(address newPriceFeed);

```

This event signals that the **price feed address** has been updated.

It records the new feed address, so that everything remains fully transparent —

users can trust that any changes to critical data sources are publicly visible.

---

```solidity
 
event CollateralizationRatioUpdated(uint256 newRatio);

```

Finally, this event announces whenever the **collateralization ratio** has been changed.

Users and dApps monitoring the system can immediately react if the required safety buffer is raised or lowered.

---

# 🚨 Custom Errors

When something goes wrong inside a smart contract, like an invalid input or a failed check, we want the transaction to **fail gracefully** and explain *why*.

In older Solidity code, this was usually done with `require()` statements and error strings — but that approach is expensive on gas.

**Custom errors** are a newer, cleaner way to handle failures.

They’re cheaper, easier to read, and make debugging way simpler for users and developers.

Here are the custom errors we’ve declared:

---

```solidity
 
error InvalidCollateralTokenAddress();
error InvalidPriceFeedAddress();
error MintAmountIsZero();
error InsufficientStablecoinBalance();
error CollateralizationRatioTooLow();

```

---

# 🧩 Breaking it Down

---

```solidity
 
error InvalidCollateralTokenAddress();

```

This error is thrown if someone tries to deploy the contract with an invalid (zero) collateral token address.

Without a real token backing the system, the whole stablecoin wouldn’t work — so we stop it right at the start.

---

```solidity
 
error InvalidPriceFeedAddress();

```

This error triggers if the provided price feed address is invalid.

Since we depend heavily on live price data, connecting to a bad feed would make the system unsafe — so we block it immediately.

---

```solidity
 
error MintAmountIsZero();

```

If a user tries to mint **zero** stablecoins, we throw this error.

Minting zero tokens doesn’t make sense and would just waste gas, so we prevent it cleanly.

---

```solidity
 
error InsufficientStablecoinBalance();

```

This error is used when a user tries to **redeem** more stablecoins than they actually have in their balance.

It protects against accidental mistakes and potential abuse.

---

```solidity
 
error CollateralizationRatioTooLow();

```

If someone tries to set the collateralization ratio below 100%, this error is thrown.

Allowing a ratio under 100% would mean stablecoins aren’t fully backed — a huge risk — so we enforce a strict minimum.

---

# 🏗️ Constructor

The constructor is the **launch sequence** for our contract.

It runs **once** — at the moment the contract is deployed — and sets up all the key pieces: what token we accept as collateral, where we get price data from, and who controls the contract.

Here’s the code:

---

```solidity
 
constructor(
    address _collateralToken,
    address _initialOwner,
    address _priceFeed
) ERC20("Simple USD Stablecoin", "sUSD") Ownable(_initialOwner) {
    if (_collateralToken == address(0)) revert InvalidCollateralTokenAddress();
    if (_priceFeed == address(0)) revert InvalidPriceFeedAddress();

    collateralToken = IERC20(_collateralToken);
    collateralDecimals = IERC20Metadata(_collateralToken).decimals();
    priceFeed = AggregatorV3Interface(_priceFeed);

    _grantRole(DEFAULT_ADMIN_ROLE, _initialOwner);
    _grantRole(PRICE_FEED_MANAGER_ROLE, _initialOwner);
}

```

---

# 🧩 Breaking it Down

---

```solidity
 
constructor(
    address _collateralToken,
    address _initialOwner,
    address _priceFeed
)

```

When deploying the contract, we must provide:

- The address of the **collateral token** (an ERC-20 like USDC, WETH, etc.),
- The address of the **initial owner** (the admin),
- The address of the **Chainlink price feed** (to fetch live collateral prices).

---

```solidity
 
ERC20("Simple USD Stablecoin", "sUSD")

```

Right here inside the constructor, we call the ERC-20 constructor from OpenZeppelin —

giving our token a **name** ("Simple USD Stablecoin") and a **symbol** ("sUSD") that will appear in wallets and explorers.

---

```solidity
 
Ownable(_initialOwner)

```

We also call the `Ownable` constructor, setting the initial owner of the contract —

the one who will have special admin privileges like adjusting settings later.

---

```solidity
 
if (_collateralToken == address(0)) revert InvalidCollateralTokenAddress();

```

Before doing anything else, we double-check that the provided collateral token address is valid (not the zero address).

If it’s invalid, we **revert** the deployment immediately with a clear custom error.

---

```solidity
 
if (_priceFeed == address(0)) revert InvalidPriceFeedAddress();

```

Same idea here — we make sure the Chainlink price feed address is valid before proceeding.

No price feed, no stablecoin logic.

---

```solidity
 
collateralToken = IERC20(_collateralToken);

```

We save the address of the collateral token so the contract knows **what token** users will deposit when minting stablecoins.

---

```solidity
 
collateralDecimals = IERC20Metadata(_collateralToken).decimals();

```

Different tokens have different decimal formats.

We fetch and store the collateral token’s decimals right now, so we can handle all math correctly later.

---

```solidity
 
priceFeed = AggregatorV3Interface(_priceFeed);

```

We connect our contract to the **Chainlink price feed**, giving it the ability to fetch live price data on demand.

---

```solidity
 
_grantRole(DEFAULT_ADMIN_ROLE, _initialOwner);
_grantRole(PRICE_FEED_MANAGER_ROLE, _initialOwner);

```

Finally, we grant two important roles:

- **DEFAULT_ADMIN_ROLE** — gives the owner full control over the contract’s role system,
- **PRICE_FEED_MANAGER_ROLE** — lets the owner update the price feed if needed in the future.

By assigning these roles right at deployment, we make sure the system is secure and manageable from day one.

---

# 🧮 Getting the Live Price from Chainlink

This function is a **helper** that we use throughout the contract — in both minting and redeeming logic, as well as in view functions.

Instead of repeating the same price-fetching logic everywhere, we wrap it neatly in `getCurrentPrice()` so it’s reusable and easy to maintain.

Here’s the code:

---

```solidity
 
function getCurrentPrice() public view returns (uint256) {
    (, int256 price, , , ) = priceFeed.latestRoundData();
    require(price > 0, "Invalid price feed response");
    return uint256(price);
}

```

---

# 🧩 Breaking it Down

---

```solidity
 
function getCurrentPrice() public view returns (uint256)

```

This is a **public view** function — anyone can call it to get the latest price, and it doesn’t change any state.

---

```solidity
 
(, int256 price, , , ) = priceFeed.latestRoundData();

```

We call `latestRoundData()` from the Chainlink Aggregator interface.

This gives us the **latest reported price** — but we only care about the second value in the tuple, which is the actual price (returned as an `int256`).

---

```solidity
 
require(price > 0, "Invalid price feed response");

```

Just to be safe, we check that the returned price is greater than zero.

If the oracle ever fails or returns bad data, we block the transaction to prevent faulty collateral calculations.

---

```solidity
 
return uint256(price);

```

Since Solidity math works best with unsigned integers (`uint256`), we convert the `int256` value into a positive number and return it.

---

This function is called any time we need a live price — which happens **a lot** in a stablecoin.

By centralizing the logic here, we keep things clean, safe, and easy to update in one place if needed.

---

# 🏗️ Minting Stablecoins

The mint function is where users **deposit collateral** into the contract and receive **freshly minted stablecoins** in return.

This is the entry point for anyone who wants to create new sUSD tokens based on the value of their deposited assets.

Here’s the code:

---

```solidity
 
function mint(uint256 amount) external nonReentrant {
    if (amount == 0) revert MintAmountIsZero();

    uint256 collateralPrice = getCurrentPrice();
    uint256 requiredCollateralValueUSD = amount * (10 ** decimals()); // 18 decimals assumed for sUSD
    uint256 requiredCollateral = (requiredCollateralValueUSD * collateralizationRatio) / (100 * collateralPrice);
    uint256 adjustedRequiredCollateral = (requiredCollateral * (10 ** collateralDecimals)) / (10 ** priceFeed.decimals());

    collateralToken.safeTransferFrom(msg.sender, address(this), adjustedRequiredCollateral);
    _mint(msg.sender, amount);

    emit Minted(msg.sender, amount, adjustedRequiredCollateral);
}

```

---

# 🧩 Breaking it Down

---

```solidity
 
function mint(uint256 amount) external nonReentrant {

```

This is a **public-facing function** — any user can call it to mint stablecoins.

We also protect it with `nonReentrant` to prevent sneaky attacks during the minting process.

---

```solidity
 
if (amount == 0) revert MintAmountIsZero();

```

We immediately block users from minting **zero** stablecoins.

Minting nothing would be pointless and would just waste gas.

---

```solidity
 
uint256 collateralPrice = getCurrentPrice();

```

We fetch the **current live price** of the collateral token using our connected Chainlink price feed.

This ensures that the required collateral calculation is always based on **real-world data**, not outdated assumptions.

---

```solidity
 
uint256 requiredCollateralValueUSD = amount * (10 ** decimals());

```

We calculate the **USD value** of the stablecoins the user wants to mint.

Since sUSD uses 18 decimals (like ETH), we multiply by `10 ** 18` to handle the scaling properly.

---

```solidity
 
uint256 requiredCollateral = (requiredCollateralValueUSD * collateralizationRatio) / (100 * collateralPrice);

```

Next, we calculate how much **collateral value** (in USD terms) the user needs to deposit, based on the collateralization ratio (like 150%).

If you want to mint $100 of sUSD at 150%, you should provide $150 worth of collateral.

---

```solidity
 
uint256 adjustedRequiredCollateral = (requiredCollateral * (10 ** collateralDecimals)) / (10 ** priceFeed.decimals());

```

Since tokens and price feeds often use different numbers of decimals (e.g., 6 vs 18),

we adjust the numbers here to make sure the calculation is **precision-correct** for both the collateral token and the price feed.

---

```solidity
 
collateralToken.safeTransferFrom(msg.sender, address(this), adjustedRequiredCollateral);

```

We safely transfer the required amount of collateral from the user into the contract.

`safeTransferFrom` makes sure the operation either succeeds fully or fails cleanly — no silent errors.

---

```solidity
 
_mint(msg.sender, amount);

```

Once the collateral is safely deposited, we **mint** the requested amount of sUSD stablecoins directly into the user’s wallet.

---

```solidity
 
emit Minted(msg.sender, amount, adjustedRequiredCollateral);

```

Finally, we fire the **Minted** event — recording who minted, how much they minted, and how much collateral they deposited —

giving dApps and users full visibility into what just happened.

---

# 🔄 Redeeming Stablecoins

If minting is how users create sUSD by depositing collateral,

**redeeming** is how they burn their sUSD and **get their collateral back**.

This function does the opposite of `mint`: it ensures the user has enough stablecoins, calculates how much collateral they should get back based on the latest price, and safely returns it to them.

Here’s the code:

---

```solidity
 
function redeem(uint256 amount) external nonReentrant {
    if (amount == 0) revert MintAmountIsZero();
    if (balanceOf(msg.sender) < amount) revert InsufficientStablecoinBalance();

    uint256 collateralPrice = getCurrentPrice();
    uint256 stablecoinValueUSD = amount * (10 ** decimals());
    uint256 collateralToReturn = (stablecoinValueUSD * 100) / (collateralizationRatio * collateralPrice);
    uint256 adjustedCollateralToReturn = (collateralToReturn * (10 ** collateralDecimals)) / (10 ** priceFeed.decimals());

    _burn(msg.sender, amount);
    collateralToken.safeTransfer(msg.sender, adjustedCollateralToReturn);

    emit Redeemed(msg.sender, amount, adjustedCollateralToReturn);
}

```

---

# 🧩 Breaking it Down

---

```solidity
 
if (amount == 0) revert MintAmountIsZero();

```

We block zero-value redemptions right away.

Redeeming nothing is just a waste of gas.

---

```solidity
 
if (balanceOf(msg.sender) < amount) revert InsufficientStablecoinBalance();

```

We check that the user actually owns enough sUSD to redeem.

If they try to burn more than they hold, the transaction is reverted — clean and clear.

---

```solidity
 
uint256 collateralPrice = getCurrentPrice();

```

We fetch the **latest real-world price** of the collateral token from Chainlink.

This ensures that the amount of collateral the user receives is based on up-to-date, trusted market data.

---

```solidity
 
uint256 stablecoinValueUSD = amount * (10 ** decimals());

```

We calculate the USD value of the sUSD tokens being redeemed, properly scaled using 18 decimals (since that’s what sUSD uses).

---

```solidity
 
uint256 collateralToReturn = (stablecoinValueUSD * 100) / (collateralizationRatio * collateralPrice);

```

We then figure out **how much collateral value** should be returned —

based on the collateralization ratio and the current market price.

> Example: If you're redeeming $100 and the ratio is 150%, you should get back around $66.66 worth of collateral.
> 

---

```solidity
 
uint256 adjustedCollateralToReturn = (collateralToReturn * (10 ** collateralDecimals)) / (10 ** priceFeed.decimals());

```

Here, we adjust for any differences between the number of decimals used by the collateral token and the price feed —

so the amount returned is **precise and accurate**.

---

```solidity
 
_burn(msg.sender, amount);

```

We **burn** the stablecoins being redeemed — permanently removing them from circulation.

---

```solidity
 
collateralToken.safeTransfer(msg.sender, adjustedCollateralToReturn);

```

Once the sUSD is burned, the calculated amount of collateral is **safely sent back** to the user’s wallet.

---

```solidity
 
emit Redeemed(msg.sender, amount, adjustedCollateralToReturn);

```

We emit a **Redeemed** event, recording:

- who redeemed,
- how much sUSD they burned,
- how much collateral they received.

This keeps everything transparent and traceable on-chain.

---

# ⚙️ Updating the Collateralization Ratio

This function allows the **owner** to adjust the system’s safety buffer — the percentage of overcollateralization required when minting new stablecoins.

Tuning this ratio can help respond to changing market conditions (like high volatility), but it’s tightly controlled to avoid risky settings.

Here’s the function:

---

```solidity
 
function setCollateralizationRatio(uint256 newRatio) external onlyOwner {
    if (newRatio < 100) revert CollateralizationRatioTooLow();
    collateralizationRatio = newRatio;
    emit CollateralizationRatioUpdated(newRatio);
}

```

---

# 🧩 Breaking it Down

---

```solidity
 
function setCollateralizationRatio(uint256 newRatio) external onlyOwner {

```

This is an **owner-only function** — no one else can call it.

It gives the owner the ability to update how much collateral users must deposit when minting.

---

```solidity
 
if (newRatio < 100) revert CollateralizationRatioTooLow();

```

We enforce a **strict lower bound**: the ratio can never go below 100%.

That ensures every stablecoin is always backed by at least its full value in collateral — no fractional reserves allowed.

---

```solidity
 
collateralizationRatio = newRatio;

```

If the new ratio is valid, we store it.

This new value will be used in all future `mint` and `redeem` calculations.

---

```solidity
 
emit CollateralizationRatioUpdated(newRatio);

```

We emit an event so that the change is **visible on-chain**.

This keeps the system transparent — dApps and users can see when (and how) the rules have changed.

---

# 🔄 Updating the Price Feed

Price feeds are the backbone of any collateral-backed stablecoin.

If the data source breaks or needs an upgrade, we need a way to **safely update it** without redeploying the entire contract.

This function lets a trusted role — not just the owner — update the Chainlink feed when needed.

---

```solidity
 
function setPriceFeedContract(address _newPriceFeed) external onlyRole(PRICE_FEED_MANAGER_ROLE) {
    if (_newPriceFeed == address(0)) revert InvalidPriceFeedAddress();
    priceFeed = AggregatorV3Interface(_newPriceFeed);
    emit PriceFeedUpdated(_newPriceFeed);
}

```

---

# 🧩 Breaking it Down

---

```solidity
 
function setPriceFeedContract(address _newPriceFeed) external onlyRole(PRICE_FEED_MANAGER_ROLE) {

```

This function can only be called by someone with the **PRICE_FEED_MANAGER_ROLE**.

That way, we don’t give all price feed control to the owner — it’s a separate, tightly scoped role.

---

```solidity
 
if (_newPriceFeed == address(0)) revert InvalidPriceFeedAddress();

```

We check that the new address is valid (not zero).

If it's not, we stop the transaction immediately with a clean custom error.

---

```solidity
 
priceFeed = AggregatorV3Interface(_newPriceFeed);

```

Once verified, we update the internal `priceFeed` reference — connecting the contract to a new Chainlink feed.

---

```solidity
 
emit PriceFeedUpdated(_newPriceFeed);

```

We emit an event so that the update is publicly visible.

This keeps the system transparent and makes sure any dApp or frontend using this contract can respond to the change.

---

# 📊 Previewing Required Collateral

Before a user calls `mint`, it’s helpful to know exactly **how much collateral** they’ll need to deposit.

This function makes that easy — it lets frontends and users **preview the required amount** based on the live price feed and current ratio, without actually sending a transaction.

Here’s the code:

---

```solidity
 
function getRequiredCollateralForMint(uint256 amount) public view returns (uint256) {
    if (amount == 0) return 0;

    uint256 collateralPrice = getCurrentPrice();
    uint256 requiredCollateralValueUSD = amount * (10 ** decimals());
    uint256 requiredCollateral = (requiredCollateralValueUSD * collateralizationRatio) / (100 * collateralPrice);
    uint256 adjustedRequiredCollateral = (requiredCollateral * (10 ** collateralDecimals)) / (10 ** priceFeed.decimals());

    return adjustedRequiredCollateral;
}

```

---

# 🧩 Breaking it Down

---

```solidity
 
function getRequiredCollateralForMint(uint256 amount) public view returns (uint256)

```

This is a **view function**, meaning it doesn’t modify state — it only reads from the contract.

Anyone can call it to check how much collateral they’d need to mint a given amount of sUSD.

---

```solidity
 
if (amount == 0) return 0;

```

If the user is asking how much collateral is needed for minting zero tokens, the answer is simple: none.

---

```solidity
 
uint256 collateralPrice = getCurrentPrice();

```

We fetch the **current market price** of the collateral using the live Chainlink feed.

---

```solidity
 
uint256 requiredCollateralValueUSD = amount * (10 ** decimals());

```

We scale the amount of stablecoins to 18 decimals to express it in full USD value terms.

This gives us the total USD value the collateral must cover.

---

```solidity
 
uint256 requiredCollateral = (requiredCollateralValueUSD * collateralizationRatio) / (100 * collateralPrice);

```

We apply the **collateralization ratio** (e.g., 150%) to determine how much more than the USD value the user must provide.

---

```solidity
 
uint256 adjustedRequiredCollateral = (requiredCollateral * (10 ** collateralDecimals)) / (10 ** priceFeed.decimals());

```

We adjust for any **decimal mismatch** between the collateral token and the price feed.

This ensures that the final number is correctly scaled in the units of the collateral token.

---

```solidity
 
return adjustedRequiredCollateral;

```

Finally, we return the exact number of **collateral tokens (in smallest units)** the user needs to send to mint the desired sUSD amount.

---

# 💵 Previewing Collateral Returned on Redemption

This function is the **counterpart to `getRequiredCollateralForMint`** — it helps users check **how much collateral** they’ll receive if they redeem a certain amount of sUSD.

It’s a pure, read-only function that’s super useful for frontends and wallets to give users clarity before they take any action.

---

```solidity
 
function getCollateralForRedeem(uint256 amount) public view returns (uint256) {
    if (amount == 0) return 0;

    uint256 collateralPrice = getCurrentPrice();
    uint256 stablecoinValueUSD = amount * (10 ** decimals());
    uint256 collateralToReturn = (stablecoinValueUSD * 100) / (collateralizationRatio * collateralPrice);
    uint256 adjustedCollateralToReturn = (collateralToReturn * (10 ** collateralDecimals)) / (10 ** priceFeed.decimals());

    return adjustedCollateralToReturn;
}

```

---

# 🧩 Breaking it Down

---

```solidity
 
if (amount == 0) return 0;

```

If the user asks to redeem **zero** stablecoins, the answer is obviously zero — no need to run the rest of the calculation.

---

```solidity
 
uint256 collateralPrice = getCurrentPrice();

```

We fetch the **live price** of the collateral token using the same helper we defined earlier.

That ensures all math is based on up-to-date market data.

---

```solidity
 
uint256 stablecoinValueUSD = amount * (10 ** decimals());

```

We convert the amount of sUSD the user wants to redeem into its **USD value**, using 18 decimal scaling since our stablecoin uses that format.

---

```solidity
 
uint256 collateralToReturn = (stablecoinValueUSD * 100) / (collateralizationRatio * collateralPrice);

```

This tells us the **USD value of collateral** the user is entitled to receive.

Since the system is overcollateralized (e.g., 150%), users only get back a portion of the full value they originally locked.

---

```solidity
 
uint256 adjustedCollateralToReturn = (collateralToReturn * (10 ** collateralDecimals)) / (10 ** priceFeed.decimals());

```

Again, we handle the difference in **decimal formats** between the collateral token and the price feed — so the result is correctly scaled in token units.

---

```solidity
 
return adjustedCollateralToReturn;

```

Finally, we return the amount of collateral (in smallest units like wei) the user would receive if they redeemed the given sUSD amount.

---

With this, users have full visibility into both sides of the system:

- How much collateral they need to mint.
- How much collateral they’ll get back when they redeem.

---

# 🧰 Step-by-Step Guide to Deploying SimpleStablecoin on Remix with Mainnet Fork

### 1. Open Remix IDE

- Navigate to Remix IDE in your browser.

### 2. Set Up the Mainnet Fork Environment

1. Click on the **"Deploy & Run Transactions"** tab (🚀 icon) on the left sidebar.
2. In the **"Environment"** dropdown, select **"Remix VM - Mainnet fork"**.
    - This forks the Ethereum mainnet and loads it into the Remix VM, allowing you to interact with deployed mainnet contracts.
    - Remix provides 10 accounts, each preloaded with 100 ETH for testing purposes

*Note: Be cautious when using the mainnet fork, as it simulates the mainnet environment. Ensure you're not sending real transactions unless intended.*

### 3. Create and Compile Your Custom ERC20 Token

1. In the **File Explorer** (📁 icon), create a new file named `MyToken.sol`.
2. Paste the following code into `MyToken.sol`:
    
    ```solidity
    // SPDX-License-Identifier: MIT
    pragma solidity ^0.8.20;
    
    import "@openzeppelin/contracts/token/ERC20/ERC20.sol";
    
    contract MyToken is ERC20 {
        constructor() ERC20("MyToken", "MTK") {
            _mint(msg.sender, 1_000_000 * 10 ** decimals());
        }
    }
    
    ```
    
3. Click on the **"Solidity Compiler"** tab (🛠️ icon).
4. Ensure the compiler version matches the pragma in your contract (e.g., `0.8.20`).
5. Click **"Compile MyToken.sol"**.

### 4. Deploy Your Custom ERC20 Token

1. In the **"Deploy & Run Transactions"** tab:
    - Ensure the **Environment** is set to **"Remix VM - Mainnet fork"**.
    - Select the `MyToken` contract from the dropdown.
2. Click **"Deploy"**.
3. After deployment, you'll see your contract under **Deployed Contracts**. Expand it to view available functions.

### 5. Identify Chainlink Price Feed Address

Since you're using a mainnet fork, you can interact with actual deployed contracts. For example:

- **Chainlink ETH/USD Price Feed (Ethereum Mainnet):**
    - Address: `0x5f4ec3df9cbd43714fe2740f5e3616155c5b8419`
    - Decimals: 8

*Note: These addresses are specific to the Ethereum mainnet. Ensure they are accurate and up-to-date by referencing [Chainlink's official documentation](https://docs.chain.link/data-feeds/price-feeds/addresses).*

### 6. Create and Compile the SimpleStablecoin Contract

1. In the **File Explorer**, create a new file named `SimpleStablecoin.sol`.
2. Paste your complete **SimpleStablecoin** contract code into this file.
    - Ensure all necessary imports from OpenZeppelin are included.
3. Click on the **"Solidity Compiler"** tab.
4. Ensure the compiler version matches the pragma in your contract.
5. Click **"Compile SimpleStablecoin.sol"**.

### 7. Deploy the SimpleStablecoin Contract

1. In the **"Deploy & Run Transactions"** tab:
    - Ensure the **Environment** is set to **"Remix VM - Mainnet fork"**.
    - Select the `SimpleStablecoin` contract from the dropdown.
2. Input the constructor parameters:
    - `_collateralToken`: Paste the address of your deployed `MyToken` contract.
    - `_initialOwner`: Use one of the pre-funded account addresses provided by Remix.
    - `_priceFeedContract`: Paste the Chainlink ETH/USD price feed address.
3. Click **"Deploy"**.

### 8. Interact with the Deployed Contracts

### Approve the SimpleStablecoin Contract to Spend Your Tokens

1. In the **Deployed Contracts** section, expand your `MyToken` contract.
2. Call the `approve` function:
    - `_spender`: Address of your deployed `SimpleStablecoin` contract.
    - `_value`: Amount of tokens to approve (e.g., `1000000000000000000000` for 1000 tokens with 18 decimals).

### Mint Stablecoins

1. Expand your `SimpleStablecoin` contract in the **Deployed Contracts** section.
2. Call the `mint` function:
    - `amount`: Number of stablecoins you wish to mint ( 100 tokens).

### Redeem Stablecoins

1. Call the `redeem` function:
    - `amount`: Number of stablecoins you wish to redeem.

*Note: Ensure you have sufficient balances and approvals before performing these actions.*