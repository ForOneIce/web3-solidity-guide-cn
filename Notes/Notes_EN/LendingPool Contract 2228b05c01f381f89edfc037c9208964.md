# LendingPool Contract

Welcome back to **30 Days of Solidity** — where every day, we unlock a new superpower of smart contracts.

So far, you’ve built contracts that send and receive ETH, guarded them with access control, and even dabbled in NFTs. But today?

**Today, we build a bank.**

Not the kind with long queues, paperwork, and hidden fees.

Not the kind that closes at 5PM.

We’re talking about **DeFi** — short for **Decentralized Finance**.

And it’s not just a buzzword. DeFi is one of the most game-changing movements in the blockchain space. It's about **rebuilding the entire financial system** — lending, borrowing, saving, trading — but doing it without centralized institutions.

Imagine this:

- No banks freezing your account.
- No loan officers judging your credit.
- No approval wait times.

Just **code** — open, transparent, unstoppable code — that anyone can interact with.

That’s what makes DeFi so powerful. You take the rules of traditional finance, **turn them into smart contract logic**, and let the blockchain do the rest.

And here’s the magic: once deployed, the contract doesn’t care *who* you are.

If the numbers add up — you can lend, you can borrow, you can earn interest.

So what are we building today?

We're creating a **simple yet powerful lending pool**.

A mini version of what platforms like **Aave** or **Compound** offer — but way easier to understand.

You’ll learn how to:

- Accept ETH deposits from users 💰
- Let them lock collateral to unlock borrowing 🛡️
- Charge interest over time 📈
- Let users repay loans and withdraw their funds ✅

This is your first hands-on taste of **DeFi mechanics** — and trust me, it’s going to change how you see smart contracts forever.

Let’s dive in.

---

## 🔍 A Quick Look at What We’re Building

The `SimpleLending` contract is your very first **DeFi bank** on-chain.

It lets users:

- **Deposit** ETH into a pool
- **Lock up collateral** to access loans
- **Borrow** ETH based on that collateral
- **Repay** loans with interest
- And **withdraw** funds when they’re done

We’re keeping it minimal — no ERC20s, no external price feeds — just pure ETH-based lending logic to help you **understand how DeFi works at the core**.

Now let’s see the full code 👇

---

## 🧾 Full Contract: `SimpleLending.sol`

```solidity
 
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

/**
 * @title SimpleLending
 * @dev A basic DeFi lending and borrowing platform
 */
contract SimpleLending {
    // Token balances for each user
    mapping(address => uint256) public depositBalances;

    // Borrowed amounts for each user
    mapping(address => uint256) public borrowBalances;

    // Collateral provided by each user
    mapping(address => uint256) public collateralBalances;

    // Interest rate in basis points (1/100 of a percent)
    // 500 basis points = 5% interest
    uint256 public interestRateBasisPoints = 500;

    // Collateral factor in basis points (e.g., 7500 = 75%)
    // Determines how much you can borrow against your collateral
    uint256 public collateralFactorBasisPoints = 7500;

    // Timestamp of last interest accrual
    mapping(address => uint256) public lastInterestAccrualTimestamp;

    // Events
    event Deposit(address indexed user, uint256 amount);
    event Withdraw(address indexed user, uint256 amount);
    event Borrow(address indexed user, uint256 amount);
    event Repay(address indexed user, uint256 amount);
    event CollateralDeposited(address indexed user, uint256 amount);
    event CollateralWithdrawn(address indexed user, uint256 amount);

    function deposit() external payable {
        require(msg.value > 0, "Must deposit a positive amount");
        depositBalances[msg.sender] += msg.value;
        emit Deposit(msg.sender, msg.value);
    }

    function withdraw(uint256 amount) external {
        require(amount > 0, "Must withdraw a positive amount");
        require(depositBalances[msg.sender] >= amount, "Insufficient balance");
        depositBalances[msg.sender] -= amount;
        payable(msg.sender).transfer(amount);
        emit Withdraw(msg.sender, amount);
    }

    function depositCollateral() external payable {
        require(msg.value > 0, "Must deposit a positive amount as collateral");
        collateralBalances[msg.sender] += msg.value;
        emit CollateralDeposited(msg.sender, msg.value);
    }

    function withdrawCollateral(uint256 amount) external {
        require(amount > 0, "Must withdraw a positive amount");
        require(collateralBalances[msg.sender] >= amount, "Insufficient collateral");

        uint256 borrowedAmount = calculateInterestAccrued(msg.sender);
        uint256 requiredCollateral = (borrowedAmount * 10000) / collateralFactorBasisPoints;

        require(
            collateralBalances[msg.sender] - amount >= requiredCollateral,
            "Withdrawal would break collateral ratio"
        );

        collateralBalances[msg.sender] -= amount;
        payable(msg.sender).transfer(amount);
        emit CollateralWithdrawn(msg.sender, amount);
    }

    function borrow(uint256 amount) external {
        require(amount > 0, "Must borrow a positive amount");
        require(address(this).balance >= amount, "Not enough liquidity in the pool");

        uint256 maxBorrowAmount = (collateralBalances[msg.sender] * collateralFactorBasisPoints) / 10000;
        uint256 currentDebt = calculateInterestAccrued(msg.sender);

        require(currentDebt + amount <= maxBorrowAmount, "Exceeds allowed borrow amount");

        borrowBalances[msg.sender] = currentDebt + amount;
        lastInterestAccrualTimestamp[msg.sender] = block.timestamp;

        payable(msg.sender).transfer(amount);
        emit Borrow(msg.sender, amount);
    }

    function repay() external payable {
        require(msg.value > 0, "Must repay a positive amount");

        uint256 currentDebt = calculateInterestAccrued(msg.sender);
        require(currentDebt > 0, "No debt to repay");

        uint256 amountToRepay = msg.value;
        if (amountToRepay > currentDebt) {
            amountToRepay = currentDebt;
            payable(msg.sender).transfer(msg.value - currentDebt);
        }

        borrowBalances[msg.sender] = currentDebt - amountToRepay;
        lastInterestAccrualTimestamp[msg.sender] = block.timestamp;

        emit Repay(msg.sender, amountToRepay);
    }

    function calculateInterestAccrued(address user) public view returns (uint256) {
        if (borrowBalances[user] == 0) {
            return 0;
        }

        uint256 timeElapsed = block.timestamp - lastInterestAccrualTimestamp[user];
        uint256 interest = (borrowBalances[user] * interestRateBasisPoints * timeElapsed) / (10000 * 365 days);

        return borrowBalances[user] + interest;
    }

    function getMaxBorrowAmount(address user) external view returns (uint256) {
        return (collateralBalances[user] * collateralFactorBasisPoints) / 10000;
    }

    function getTotalLiquidity() external view returns (uint256) {
        return address(this).balance;
    }
}

```

## 🧾 Contract Declaration

```solidity
 
contract SimpleLending {

```

We name our contract `SimpleLending`, because, well… it’s a **simple lending system**. This is the foundation of our DeFi bank.

---

## 🧠 The Brain of the Contract – Mappings

In any banking system — whether it’s your neighborhood branch or a DeFi smart contract — there needs to be a way to **remember who owns what**.

That’s exactly what we’re doing here.

This section is the **brain** of our lending pool. These mappings are how we track deposits, debts, collateral, and interest — like a digital ledger that keeps everything organized on-chain.

Let’s look at the four key mappings in this contract:

```solidity
 
mapping(address => uint256) public depositBalances;
mapping(address => uint256) public borrowBalances;
mapping(address => uint256) public collateralBalances;
mapping(address => uint256) public lastInterestAccrualTimestamp;

```

---

### `depositBalances` – Your Digital Vault

This mapping keeps track of how much ETH a user has **deposited into the lending pool**.

Think of it as a personal vault inside the contract. When a user calls the `deposit()` function, their ETH is stored in the contract, and this mapping gets updated to reflect how much they’ve added.

You can withdraw from this vault anytime — unless you've borrowed against it.

> 📦 It's like your checking account balance — always accessible unless tied to a loan.
> 

---

### `borrowBalances` – Your Loan Ledger

This mapping shows how much ETH a user has **borrowed** from the pool.

When you take out a loan using the `borrow()` function, the ETH leaves the contract and comes to your wallet. At the same time, this mapping gets updated to track exactly how much debt you owe.

We also use this value (along with interest) to determine whether you’re allowed to withdraw your collateral or borrow more.

> 🧾 It’s your loan tracker — the number that says: “Here’s what you owe.”
> 

---

### `collateralBalances` – The Safety Net

In DeFi, borrowing isn’t trust-based — it’s **math-based**.

To take a loan, you need to **lock up collateral**. This mapping keeps track of how much ETH each user has provided as collateral.

When you call `depositCollateral()`, the ETH stays in the contract, but your `collateralBalances` entry increases.

And when you try to borrow or withdraw collateral, the contract uses this mapping to check whether your position is still safe.

> 🔐 Think of this like your security deposit — it's there to protect the system if you don't pay your dues.
> 

---

### `lastInterestAccrualTimestamp` – The Timekeeper

Interest in our system doesn’t accumulate automatically every second — that would be way too gas-intensive.

Instead, we record **the last time** we calculated interest for each user. Then, whenever the user interacts with the system (borrows, repays, etc.), we check how much time has passed and calculate how much interest has built up *since then*.

This lets us **simulate continuous interest** without paying for it on every block.

> 🧮 It’s like a timestamp receipt: “Last time we checked, this user owed X. Let’s now update based on the time that’s passed.”
> 

---

## 💰 Lending Logic – Setting the Financial Rules

Every lending platform — whether it's a DeFi protocol or your local bank — runs on **a few key financial rules**.

In our smart contract, those rules are baked into two simple variables:

```solidity
 
uint256 public interestRateBasisPoints = 500;         // 5% annual interest rate
uint256 public collateralFactorBasisPoints = 7500;    // 75% loan-to-value (LTV)

```

Let’s unpack what each of these means and why they’re so important.

---

### 🧾 `interestRateBasisPoints` – How the Protocol Earns

This is the **interest rate** that borrowers will pay annually on their loan.

But instead of writing `5` for 5%, we write `500` — because we’re using **basis points**.

> 🔢 Basis points are used in finance to express percentages precisely, especially when dealing with fractional rates.
> 
> 
> 1 basis point = 0.01%, so 500 basis points = **5%**
> 

That means:

If a user borrows 1 ETH, they’ll owe 1.05 ETH after a year (assuming they don’t repay early).

By using basis points, we can **avoid floating-point math** (which Solidity doesn’t support), and keep things super precise.

---

### 🛡️ `collateralFactorBasisPoints` – How the Protocol Stays Safe

This variable determines **how much someone can borrow** based on the value of their collateral.

In our case:

- `7500` basis points = 75%
- This means users can only borrow up to **75% of the ETH they lock in as collateral**

Let’s say you deposit 2 ETH as collateral:

- 75% of 2 ETH = **1.5 ETH**
- That’s the maximum amount you’re allowed to borrow

Why not 100%?

Because **price volatility** exists. ETH might drop in value tomorrow. So we always want borrowers to leave a safety cushion — that’s what the **collateral factor** enforces.

> 🧠 In real-world terms: it’s like a bank saying “we’ll loan you money against your car — but only up to 75% of its current value.”
> 

This helps the protocol avoid losses and ensures borrowers always have enough collateral to back their loans.

---

## 📣 Events – Broadcasting Activity

Here are the events we’re using in our DeFi bank:

```solidity
 
event Deposit(address indexed user, uint256 amount);
event Withdraw(address indexed user, uint256 amount);
event Borrow(address indexed user, uint256 amount);
event Repay(address indexed user, uint256 amount);
event CollateralDeposited(address indexed user, uint256 amount);
event CollateralWithdrawn(address indexed user, uint256 amount);
```

Let’s break them down:

### 💸 `Deposit`

Emitted when a user adds ETH to the lending pool.

> It’s like saying: “User X just deposited Y ETH — update their balance and show it on the dashboard!”
> 

---

### 🏧 `Withdraw`

Emitted when a user takes ETH out of their deposit balance.

> Used to reflect withdrawals in the UI, and to log that the contract’s ETH balance changed.
> 

---

### 🧾 `Borrow`

Fires when a user successfully takes out a loan.

> Your app can show: “Congrats! You just borrowed 0.75 ETH from the pool.”
> 

---

### 💳 `Repay`

Triggers when a user sends ETH back to repay their loan.

> Important for updating the user’s loan balance and calculating remaining debt.
> 

---

### 🛡️ `CollateralDeposited`

Emitted when a user locks ETH as collateral.

> Frontend apps can use this to show: “You’re now eligible to borrow up to X ETH.”
> 

---

### 🔓 `CollateralWithdrawn`

Happens when a user safely withdraws their collateral (after repaying or keeping a healthy loan-to-value ratio).

> A useful trigger for visualizing how much collateral remains.
> 

---

## 

## 🔧 Functions – Powering the DeFi Engine

Everything we’ve built so far — the variables, the mappings, the parameters, the events — they’re all just the **foundations**.

The real magic happens when users **interact** with the contract.

These functions are how users *do things* inside your DeFi bank: depositing ETH, borrowing funds, repaying loans, and managing their collateral. This is where the contract becomes interactive.

Let’s kick things off with the very first function in that flow — the **deposit**.

---

# 🏦 `deposit()` – Add ETH to the Pool

```solidity
 
function deposit() external payable {
    require(msg.value > 0, "Must deposit a positive amount");

    depositBalances[msg.sender] += msg.value;

    emit Deposit(msg.sender, msg.value);
}

```

This function is how users **add ETH into the lending pool** — and it's as simple as it gets.

Here’s how it works, step by step:

---

### 1. `msg.value` – Receiving ETH

Whenever someone sends ETH to a smart contract, the amount is stored in a special variable called `msg.value`.

So if I call `deposit()` and attach 1 ETH, `msg.value` will be `1 ether`.

We start by checking:

```solidity
 
require(msg.value > 0, "Must deposit a positive amount");

```

This line prevents people from accidentally calling the function without sending any ETH. No free rides!

---

### 2. Updating the Balance

Next, we update the mapping that tracks each user’s deposits:

```solidity
 
depositBalances[msg.sender] += msg.value;

```

Here, `msg.sender` is the person who called the function — the depositor.

We're saying: “Take whatever they had before, and add the new amount to it.”

> 🧠 In simpler terms: “Update this user’s bank balance.”
> 

---

### 3. Emitting the Event

Finally, we let the world know what just happened:

```solidity
 
emit Deposit(msg.sender, msg.value);

```

This triggers the `Deposit` event, which frontend apps or indexers can listen to. It’s like the contract is shouting:

> “Hey! Alice just deposited 1 ETH!”
> 

This is how you make sure the UI updates and the transaction gets logged in tools like Etherscan.

---

### 

# 💸 `withdraw(uint256 amount)` – Take Your Money Back

```solidity
 
function withdraw(uint256 amount) external {
    require(amount > 0, "Must withdraw a positive amount");
    require(depositBalances[msg.sender] >= amount, "Insufficient balance");

    depositBalances[msg.sender] -= amount;
    payable(msg.sender).transfer(amount);

    emit Withdraw(msg.sender, amount);
}

```

Now that users can **deposit ETH**, we need to give them a way to **get their ETH back** — whenever they want.

This is exactly what the `withdraw()` function is for.

Let’s walk through it like a customer walking into a bank and saying,

“Hey, I’d like to take out some of my funds.”

---

### 1. ✅ Validating the Withdrawal Request

First, the contract does two quick sanity checks:

```solidity
 
require(amount > 0, "Must withdraw a positive amount");

```

This makes sure the user isn’t accidentally trying to withdraw 0 ETH. No weird edge cases, no confusion — just clean inputs.

```solidity
 
require(depositBalances[msg.sender] >= amount, "Insufficient balance");

```

Next, we check whether the user even **has enough ETH** in their deposit balance to make this withdrawal.

If not, the transaction gets rejected.

No overdrafts in DeFi 😉

---

### 2. 🧮 Updating the Balance

If both checks pass, the user’s deposit balance gets updated:

```solidity
 
depositBalances[msg.sender] -= amount;

```

This subtracts the withdrawn amount from their account.

Just like a bank ledger reflecting that a withdrawal has occurred.

---

### 3. 🪙 Sending ETH Back

After updating the balance, the contract sends ETH back to the user:

```solidity
 
payable(msg.sender).transfer(amount);

```

This line actually **transfers real ETH** back into the user’s wallet.

> 💡 Solidity requires .transfer() to be called on a payable address — that’s why we cast msg.sender to payable(msg.sender).
> 

---

### 4. 📣 Broadcasting the Event

Finally, we let everyone know what just happened:

```solidity
 
emit Withdraw(msg.sender, amount);

```

This fires the `Withdraw` event — a signal that your frontend can listen for to show a nice confirmation like:

> “✅ Withdrawal of 0.5 ETH successful!”
> 

---

### 

# 🧮 `calculateInterestAccrued(address user)` – The Heartbeat of Borrowing

```solidity
 
function calculateInterestAccrued(address user) public view returns (uint256) {
    if (borrowBalances[user] == 0) {
        return 0;
    }

    uint256 timeElapsed = block.timestamp - lastInterestAccrualTimestamp[user];
    uint256 interest = (borrowBalances[user] * interestRateBasisPoints * timeElapsed) / (10000 * 365 days);

    return borrowBalances[user] + interest;
}

```

This function is the **engine room** of our lending system — quietly powering the most important part of borrowing: **interest calculation**.

And guess what?

You’re going to see this function show up in *almost every major feature* that involves loans:

- ✅ When you **borrow**
- 💳 When you **repay**
- 🔓 When you **withdraw collateral**
- 📊 Even when checking **how much you owe**

Let’s understand exactly what it’s doing — and why it’s such a big deal.

---

### 📊 Why Do We Need This?

In the real world, loan interest builds up **over time**.

But smart contracts can’t update variables every second — that would be **way too expensive in gas**.

So instead of constantly updating each user’s loan balance...

We use this function to **calculate interest on demand** — only when it’s needed.

> Think of it as a just-in-time interest calculator.
> 

---

### ✅ How It Works

### 1. Check If the User Has Borrowed Anything

```solidity
 
if (borrowBalances[user] == 0) {
    return 0;
}

```

No loan? No interest. Simple.

---

### 2. Calculate How Much Time Has Passed

```solidity
 
uint256 timeElapsed = block.timestamp - lastInterestAccrualTimestamp[user];

```

We figure out how much time has passed since interest was last calculated for this user.

> block.timestamp gives us the current time.
> 
> 
> `lastInterestAccrualTimestamp` tells us when we last updated their debt.
> 

---

### 3. Apply the Interest Formula

```solidity
 
uint256 interest = (borrowBalances[user] * interestRateBasisPoints * timeElapsed) / (10000 * 365 days);

```

Let’s break this formula down:

- **`borrowBalances[user]`** → the user’s current loan principal
- **`interestRateBasisPoints`** → our annual interest rate (e.g. 500 = 5%)
- **`timeElapsed`** → how long they’ve had the debt
- **`10000 * 365 days`** → used to convert basis points into a yearly fraction

This gives us **the interest accrued so far**, based on how long the loan has been active.

---

### 4. Return the Total Debt (Principal + Interest)

```solidity
 
return borrowBalances[user] + interest;

```

Boom. Just like that, you get the **up-to-date debt** — no manual math needed.

> This total is what we use in all our other functions: to verify borrowing eligibility, check repayment amounts, and validate collateral withdrawals.
> 

# 🛡️ `depositCollateral()` – Lock ETH to Borrow Later

```solidity
 
function depositCollateral() external payable {
    require(msg.value > 0, "Must deposit a positive amount as collateral");

    collateralBalances[msg.sender] += msg.value;

    emit CollateralDeposited(msg.sender, msg.value);
}

```

So far, you’ve seen how to **deposit ETH into the pool** as a lender — that's great if you want to earn passive income.

But what if you want to **borrow** from the pool instead?

Well, just like in real life — if you want to take a loan, you need to **put something down as collateral.**

This function is how you do that.

Let’s break it down 👇

---

### 🔒 Collateral vs. Deposit — What’s the Difference?

Before we get into the code, here’s the key distinction:

| Deposits | Collateral |
| --- | --- |
| Money you lend to the pool 💸 | Money you lock to borrow 💳 |
| Can be withdrawn freely (unless borrowed against) | Only returned if your loan is safe |
| You earn interest on it (if the system supports that) | It backs your borrow and keeps the protocol safe |

So yes — **both are ETH**, but they **play completely different roles**.

That’s why we have a separate function for depositing collateral.

---

### ✅ Step-by-Step Breakdown

### 1. Check for a Real Deposit

```solidity
 
require(msg.value > 0, "Must deposit a positive amount as collateral");

```

We make sure the user is sending **some** ETH — no empty transactions allowed.

---

### 2. Increase the User’s Collateral Balance

```solidity
 
collateralBalances[msg.sender] += msg.value;

```

Once ETH is received, we update the user’s **collateral record**. This tells the contract:

> “Alright, this user has now locked up X ETH — we can allow them to borrow safely up to a certain limit.”
> 

This is the key that unlocks **borrowing power** in the system.

---

### 3. Emit the Event

```solidity
 
emit CollateralDeposited(msg.sender, msg.value);

```

As always, we emit an event so that frontends and tools can react to the deposit — updating dashboards, sending notifications, and logging the action for history.

---

# 🔓 `withdrawCollateral(uint256 amount)` – Take Back Your Locked ETH (If You're Safe)

```solidity
 
function withdrawCollateral(uint256 amount) external {
    require(amount > 0, "Must withdraw a positive amount");
    require(collateralBalances[msg.sender] >= amount, "Insufficient collateral");

    uint256 borrowedAmount = calculateInterestAccrued(msg.sender);
    uint256 requiredCollateral = (borrowedAmount * 10000) / collateralFactorBasisPoints;

    require(
        collateralBalances[msg.sender] - amount >= requiredCollateral,
        "Withdrawal would break collateral ratio"
    );

    collateralBalances[msg.sender] -= amount;
    payable(msg.sender).transfer(amount);

    emit CollateralWithdrawn(msg.sender, amount);
}

```

So you locked up some ETH as collateral. That gave you the power to borrow from the pool.

Now you want to **take some of that ETH back**.

Seems fair, right?

Well… *only if you’re still safe to do so.*

This function lets users withdraw their collateral — **but only if doing so doesn’t put their loan at risk.**

Let’s walk through it step-by-step like you’re negotiating with a very cautious, but very fair smart contract banker.

---

### ✅ Step 1: Basic Checks

```solidity
 
require(amount > 0, "Must withdraw a positive amount");
require(collateralBalances[msg.sender] >= amount, "Insufficient collateral");

```

These two lines are straightforward:

- Make sure the user is trying to withdraw a real amount.
- Make sure they actually have enough collateral to even attempt this.

If either check fails — we stop right there.

---

### 🧠 Step 2: Risk Assessment

Now comes the important part — **making sure the user is still collateralized** after the withdrawal.

### First, we check how much debt they owe:

```solidity
 
uint256 borrowedAmount = calculateInterestAccrued(msg.sender);

```

This includes any interest they’ve built up over time.

---

### Then we calculate how much collateral they *need* to stay safe:

```solidity
 
uint256 requiredCollateral = (borrowedAmount * 10000) / collateralFactorBasisPoints;

```

This is based on the loan-to-value ratio (LTV) we set earlier — 75% by default.

Let’s say you owe 1 ETH and the LTV is 75%:

- You need to have at least **1 / 0.75 = 1.33 ETH** locked as collateral.

---

### ❗ Step 3: Safety Check

We now simulate the future — what would happen *if* you took out this amount of collateral?

```solidity
 
require(
    collateralBalances[msg.sender] - amount >= requiredCollateral,
    "Withdrawal would break collateral ratio"
);

```

If removing that ETH would drop you below the safe limit, **we revert** the transaction.

> ⚠️ This is like a warning from the contract saying:
> 
> 
> “Hold on. If you do this, your loan won’t be backed anymore. I can’t allow that.”
> 

---

### 🪙 Step 4: Let It Go (If You’re Safe)

If you pass all the checks, you’re good to go.

```solidity
 
collateralBalances[msg.sender] -= amount;
payable(msg.sender).transfer(amount);
emit CollateralWithdrawn(msg.sender, amount);

```

We update your record, send the ETH back to your wallet, and emit an event to let the frontend know.

---

### 

# 🧾 `borrow(uint256 amount)` – Unlocking DeFi's Superpower: Borrowing ETH

```solidity
 
function borrow(uint256 amount) external {
    require(amount > 0, "Must borrow a positive amount");
    require(address(this).balance >= amount, "Not enough liquidity in the pool");

    uint256 maxBorrowAmount = (collateralBalances[msg.sender] * collateralFactorBasisPoints) / 10000;
    uint256 currentDebt = calculateInterestAccrued(msg.sender);

    require(currentDebt + amount <= maxBorrowAmount, "Exceeds allowed borrow amount");

    borrowBalances[msg.sender] = currentDebt + amount;
    lastInterestAccrualTimestamp[msg.sender] = block.timestamp;

    payable(msg.sender).transfer(amount);
    emit Borrow(msg.sender, amount);
}

```

Alright — now we get to the part where the contract really starts acting like a bank.

This function lets users **borrow ETH from the lending pool**, based on how much collateral they’ve locked up.

But — just like in real life — you can’t walk into a bank and say “Give me free money.”

There are rules. Let’s walk through them.

---

### 🧠 Step-by-Step Breakdown

### 1. Ask for a Real Loan

```solidity
 
require(amount > 0, "Must borrow a positive amount");

```

We make sure the user isn’t trying to borrow zero ETH or a negative amount. Every borrow action must have a real number behind it.

---

### 2. Make Sure the Pool Has Liquidity

```solidity
 
require(address(this).balance >= amount, "Not enough liquidity in the pool");

```

The contract checks its own balance to see if it can actually afford to lend out that much ETH.

> You can’t lend what you don’t have.
> 

---

### 3. Check Borrower’s Eligibility

We calculate how much the user *can* borrow, based on how much collateral they’ve provided:

```solidity
 
uint256 maxBorrowAmount = (collateralBalances[msg.sender] * collateralFactorBasisPoints) / 10000;

```

So if you’ve locked 2 ETH as collateral, and the collateral factor is 75%:

- `maxBorrowAmount = 2 * 0.75 = 1.5 ETH`

But it doesn’t stop there — we also need to factor in **any previous debt + interest** they already owe:

```solidity
 
uint256 currentDebt = calculateInterestAccrued(msg.sender);

```

Then we check:

```solidity
 
require(currentDebt + amount <= maxBorrowAmount, "Exceeds allowed borrow amount");

```

If this check fails, the borrow is blocked. The user’s collateral can’t back the requested amount safely.

> ✅ Only safe, overcollateralized loans are allowed.
> 

---

### 4. Update the Debt & Time

If the user is eligible, we move forward and record the new loan:

```solidity
 
borrowBalances[msg.sender] = currentDebt + amount;
lastInterestAccrualTimestamp[msg.sender] = block.timestamp;

```

We’re saying:

- “This user now owes `X` ETH (including past debt).”
- “And this is the time we’ll use to calculate interest moving forward.”

---

### 5. Transfer the ETH

```solidity
 
payable(msg.sender).transfer(amount);

```

The contract sends real ETH into the user’s wallet.

They can now use that ETH however they want — trade it, swap it, stake it — it’s fully theirs.

---

### 6. Broadcast the Borrow Event

```solidity
 
emit Borrow(msg.sender, amount);

```

This tells the outside world (your frontend, block explorers, analytics dashboards) that this user just took out a loan.

---

### 

# 💳 `repay()` – Pay Back Your Loan, Regain Your Freedom

```solidity
 
function repay() external payable {
    require(msg.value > 0, "Must repay a positive amount");

    uint256 currentDebt = calculateInterestAccrued(msg.sender);
    require(currentDebt > 0, "No debt to repay");

    uint256 amountToRepay = msg.value;
    if (amountToRepay > currentDebt) {
        amountToRepay = currentDebt;
        payable(msg.sender).transfer(msg.value - currentDebt); // Refund the extra
    }

    borrowBalances[msg.sender] = currentDebt - amountToRepay;
    lastInterestAccrualTimestamp[msg.sender] = block.timestamp;

    emit Repay(msg.sender, amountToRepay);
}

```

You’ve borrowed ETH. Time has passed. Interest has ticked up.

Now it’s time to **pay it back** — and that’s exactly what this function is for.

Let’s unpack what happens when a user sends ETH to `repay()`.

---

### ✅ Step-by-Step Breakdown

### 1. Check That You’re Actually Sending ETH

```solidity
 
require(msg.value > 0, "Must repay a positive amount");

```

Just like the `deposit()` function, we don’t want users triggering this by mistake. You need to actually send some ETH along with the function call.

---

### 2. Check How Much You Owe

```solidity
 
uint256 currentDebt = calculateInterestAccrued(msg.sender);
require(currentDebt > 0, "No debt to repay");

```

Here, we ask:

> “How much does this user owe — including any interest that’s built up?”
> 

If they don’t owe anything, we stop right there.

---

### 3. Accept What You Sent

```solidity
 
uint256 amountToRepay = msg.value;

```

Now we record how much the user is trying to repay.

---

### 4. Handle Overpayment (Refund the Extra)

```solidity
 
if (amountToRepay > currentDebt) {
    amountToRepay = currentDebt;
    payable(msg.sender).transfer(msg.value - currentDebt);
}

```

Sometimes users send *more* than they owe (especially if they’re not sure how much interest has built up). We handle that gracefully:

- Cap the repayment at the debt amount
- Automatically refund the extra ETH

> 🧠 No need to calculate your debt manually — just send enough or more, and the contract handles the rest.
> 

---

### 5. Update the Loan Record

```solidity
 
borrowBalances[msg.sender] = currentDebt - amountToRepay;
lastInterestAccrualTimestamp[msg.sender] = block.timestamp;

```

This reduces the user’s loan balance and resets the clock on interest accrual.

If they paid everything back, their borrow balance becomes zero — **debt-free!**

---

### 6. Fire the `Repay` Event

```solidity
 
emit Repay(msg.sender, amountToRepay);

```

The frontend, explorers, or analytics tools can now show:

> “✅ Loan repayment of 1 ETH complete!”
> 

---

---

### 

# 📊 Utility Functions – Supporting Tools for UIs & Integrations

Not every function in a smart contract changes the state or handles funds.

Sometimes, you just need to **check** something.

That’s where **utility functions** come in — simple read-only tools that help frontends, other smart contracts, or analytics dashboards fetch useful data from the contract.

In our DeFi bank, we’ve got two such utility functions that do exactly that:

---

### 🔍 `getMaxBorrowAmount(address user)`

```solidity
 
function getMaxBorrowAmount(address user) external view returns (uint256) {
    return (collateralBalances[user] * collateralFactorBasisPoints) / 10000;
}

```

This function calculates how much ETH a specific user is **allowed to borrow**, based on the collateral they’ve deposited.

It doesn’t care how much they’ve already borrowed — it just tells you the **upper limit** of what they *could* borrow, given their collateral.

> 🧠 Formula:
> 
> 
> `maxBorrow = collateral × collateralFactor`
> 

So if someone has locked 4 ETH and our collateral factor is 75%, this function will return:

- `4 × 0.75 = 3 ETH` → That’s their **max borrowable limit**

This is super useful for frontends:

- It lets the UI display: **“You can borrow up to 3 ETH”**
- Or show a progress bar of how much of their borrowing power they’ve used

---

### 💧 `getTotalLiquidity()`

```solidity
 
function getTotalLiquidity() external view returns (uint256) {
    return address(this).balance;
}

```

This one’s even simpler — it just tells you **how much ETH the lending pool currently holds**.

Since every deposit and repayment adds ETH to the contract, and every withdrawal or borrow removes ETH, this function gives a live look at the available funds.

> It’s like checking the vault balance of the bank.
> 

Frontends can use this to:

- Show the total pool size
- Indicate whether there’s enough liquidity for a borrow request
- Warn users if the pool is running low on funds

---

## 🔚 Wrap-Up – What You Built

You’ve just created a **fully functional lending pool** that can:

- Accept deposits 💰
- Let users lock collateral 🛡️
- Borrow ETH 📤
- Accrue interest over time 📈
- Let users repay and reclaim collateral ✅

This contract introduces you to **the core mechanics of DeFi lending** — the same principles used by platforms like **Aave**, **Compound**, and **MakerDAO**.

It’s simple, yet powerful.