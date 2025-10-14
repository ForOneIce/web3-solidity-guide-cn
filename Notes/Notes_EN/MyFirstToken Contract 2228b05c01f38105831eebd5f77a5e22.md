# MyFirstToken Contract

Alright, so far, we’ve built calculators, shared wallets, piggy banks, and even an ETH vault with ownership logic — and along the way, we learned how inheritance and access control can keep our contracts clean and secure.

But what if we shifted gears a bit?

What if, instead of just holding or moving around ETH, we wanted to create **our own token**?

You know — like the ones you see flying around in the crypto space all the time.

$MEME, $WAGMI, $COIN — they’re everywhere.

Wouldn’t it be fun to make your own?

Well today, you’re going to do exactly that. We’re writing a token contract — but not just any token — an **ERC-20 token**.

---

## Why Tokens Needed a Standard in the First Place

Back in the early days of Ethereum, everyone was excited about the idea of creating tokens.

And so they did. Lots of them.

But there was one big problem: **everyone built them differently**.

- One token had a transfer function that used `send()`
- Another one called it `moveTokens()`
- Some tokens didn’t let you check balances
- Others didn’t emit any events when transfers happened

Now imagine being a wallet or an exchange trying to support all of them.

Every time a new token came along, you'd have to write custom logic just to get it to work.

It was like trying to build a universal remote — but every TV manufacturer decided to use different buttons, different wires, and sometimes… no power switch at all.

So the Ethereum community came together and said:

> “Alright, enough of this. Let’s all agree on a common way to build tokens.”
> 

That’s when **ERC-20** was proposed — a simple set of rules that said:

- This is what a token should look like
- These are the functions every token must have
- If you follow this, your token will work with wallets, DApps, and exchanges right out of the box

And that moment — the birth of ERC-20 — is what made Ethereum’s token ecosystem explode.

---

### What Does ERC-20 Mean?

Alright, so what even is “ERC-20”?

Let’s break it down.

**ERC** stands for **Ethereum Request for Comments** — it’s basically a public proposal that says:

> “Hey Ethereum devs, here’s a set of rules and behaviors we think everyone should follow when building a certain kind of smart contract.”
> 

It’s how the Ethereum community agrees on standards — like building codes for blockchain.

And **ERC-20**?

That’s the **20th proposal** in the ERC series — and it just so happens to be the one that defined how **fungible tokens** should behave on Ethereum.

---

### What Kind of Rules Does ERC-20 Define?

ERC-20 lays down a consistent interface — a shared language — that all tokens should speak. It covers things like:

- **Naming and Display**
    
    Your token must have a `name`, `symbol`, and `decimals` so wallets can show them properly.
    
- **Balances and Supply**
    
    There must be a `totalSupply()` and a way to check how many tokens an address owns using `balanceOf(address)`.
    
- **Transfers**
    
    There should be a `transfer()` function that lets people send tokens from their wallet to someone else.
    
- **Approvals and Delegated Spending**
    
    Your token needs an `approve()` function so users can let someone else (like a smart contract) spend tokens on their behalf.
    
    And a `transferFrom()` function to actually carry out those approved transfers.
    
- **Event Emissions**
    
    Whenever tokens move or permissions are granted, your contract should emit `Transfer` and `Approval` events.
    
    These help wallets, DApps, and block explorers track what’s happening on-chain.
    

---

### Why This Matters

By following these rules, your token becomes **plug-and-play** — it can:

- Show up in wallets like MetaMask
- Be traded on decentralized exchanges
- Work with lending protocols, DAOs, or any other app that supports ERC-20s

If your token doesn’t follow the ERC-20 rules, it’s like building a power outlet that doesn’t fit any plug — it might still work, but no one can use it.

So in short:

> ERC-20 is the blueprint that made Ethereum’s token economy possible.
> 
> 
> It’s why tokens are interoperable, reusable, and instantly compatible with the ecosystem.
> 

And now, you’re going to build one yourself.

---

## What We’re Building Today

We’re going to build a **minimal ERC-20 token** from scratch.

It won’t have minting, burning, or advanced features — and it skips some safety checks that you'd normally want in production.

But it’s perfect for understanding how tokens **actually work under the hood**.

Our token will be called **SimpleToken**, with the symbol **SIM**.

Here’s the contract we’ll walk through:

---

## `SimpleERC20.sol`

```solidity
 
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract SimpleERC20 {
    string public name = "SimpleToken";
    string public symbol = "SIM";
    uint8 public decimals = 18;
    uint256 public totalSupply;

    mapping(address => uint256) public balanceOf;
    mapping(address => mapping(address => uint256)) public allowance;

    event Transfer(address indexed from, address indexed to, uint256 value);
    event Approval(address indexed owner, address indexed spender, uint256 value);

    constructor(uint256 _initialSupply) {
        totalSupply = _initialSupply * (10 ** uint256(decimals));
        balanceOf[msg.sender] = totalSupply;
        emit Transfer(address(0), msg.sender, totalSupply);
    }

    function transfer(address _to, uint256 _value) public returns (bool) {
        require(balanceOf[msg.sender] >= _value, "Not enough balance");
        _transfer(msg.sender, _to, _value);
        return true;
    }

    function approve(address _spender, uint256 _value) public returns (bool) {
        allowance[msg.sender][_spender] = _value;
        emit Approval(msg.sender, _spender, _value);
        return true;
    }

    function transferFrom(address _from, address _to, uint256 _value) public returns (bool) {
        require(balanceOf[_from] >= _value, "Not enough balance");
        require(allowance[_from][msg.sender] >= _value, "Allowance too low");

        allowance[_from][msg.sender] -= _value;
        _transfer(_from, _to, _value);
        return true;
    }

    function _transfer(address _from, address _to, uint256 _value) internal {
        require(_to != address(0), "Invalid address");
        balanceOf[_from] -= _value;
        balanceOf[_to] += _value;
        emit Transfer(_from, _to, _value);
    }
}

```

Let’s walk through this step by step.

---

## The Breakdown

---

### Token Metadata

```solidity
 
string public name = "SimpleToken";
string public symbol = "SIM";
uint8 public decimals = 18;

```

These three lines define how your token will appear in wallets and exchanges:

- `name` is the full name of the token.
- `symbol` is the short ticker (like "ETH" or "DAI").
- `decimals` defines how divisible it is.
    
    Most tokens use 18 decimals — just like ETH.
    

---

### Token Supply

```solidity
 
uint256 public totalSupply;

```

This tracks the total number of tokens that exist. We’ll set this when the contract is deployed — more on that in the constructor.

---

### Balances and Allowances

```solidity
 
mapping(address => uint256) public balanceOf;
mapping(address => mapping(address => uint256)) public allowance;

```

- `balanceOf` tells you how many tokens each address holds.
- `allowance` is a nested mapping that tracks who’s allowed to spend tokens on behalf of whom — and how much.

This is a core feature of ERC-20: letting someone else (like a DApp or smart contract) move your tokens, but **only if you’ve approved it first**.

---

### Events

```solidity
 
event Transfer(address indexed from, address indexed to, uint256 value);
event Approval(address indexed owner, address indexed spender, uint256 value);

```

As you already know, Events in Solidity are like the contract’s way of saying, “Hey, something just happened!” — and they’re a crucial part of how smart contracts interact with the outside world.

They don’t modify any state or perform logic inside the contract. Instead, they **emit logs** that external tools like wallets, DApps, and block explorers can listen to.

Here’s what each one does:

- `Transfer`: This event fires whenever tokens move from one address to another. Wallets and explorers rely on this to display transaction histories.
- `Approval`: This gets triggered when someone gives another address permission to spend tokens on their behalf.

You’ll notice the `indexed` keyword on the address parameters — this makes those values **searchable** in the event logs. So if you want to find all transfers *from* a specific address, or all approvals *to* a certain spender, that’s how you do it.

---

### Constructor – Minting Initial Supply

```solidity
 
constructor(uint256 _initialSupply) {
    totalSupply = _initialSupply * (10 ** uint256(decimals));
    balanceOf[msg.sender] = totalSupply;
    emit Transfer(address(0), msg.sender, totalSupply);
}

```

This constructor runs once — and only once — when the contract is deployed.

Here’s what’s happening, line by line:

---

```json
totalSupply = _initialSupply * (10 ** uint256(decimals));
```

This line sets the total number of tokens that will exist.

- `_initialSupply` is the number you pass in when deploying the contract.
- But remember — ERC-20 tokens use **decimals** for precision (just like ETH uses wei).
- So if your token uses 18 decimals (which is the standard), and you want to create 100 tokens, you actually need to represent it as:
    
    `100 * 10^18 = 100000000000000000000`
    

That’s why we multiply the initial supply by `10 ** decimals`.

This scaled number is stored in `totalSupply`.

---

```json
balanceOf[msg.sender] = totalSupply;
```

The entire supply is given to the person who deployed the contract — `msg.sender`.

That means the deployer starts out holding 100% of the tokens.

From there, they can transfer, sell, or distribute tokens however they want.

---

```json
emit Transfer(address(0), msg.sender, totalSupply);
```

This emits a `Transfer` event to signal that the tokens were "minted."

- We set the `from` address as `address(0)`, which is a special way of saying:
    
    > "These tokens didn’t come from another user — they were created out of thin air."
    > 

Wallets, explorers, and frontend tools understand this pattern and display it as a minting event.

---

### 

## transfer()

```
function transfer(address _to, uint256 _value) public returns (bool) {
    require(balanceOf[msg.sender] >= _value, "Not enough balance");
    _transfer(msg.sender, _to, _value);
    return true;
}
```

This function allows a user to send their tokens to another address. It's the most direct and commonly used way to move tokens — whether you're sending a friend some SIM tokens or paying for something with your ERC-20 token.

Here's the step-by-step:

- First, it ensures the sender (`msg.sender`) has enough tokens using `require(balanceOf[msg.sender] >= _value)`.
- Then, instead of handling the transfer logic here, it calls an internal helper function `_transfer()` that performs the actual token movement.

So why split it?

That brings us to a very important design pattern in Solidity: **separation of logic**.

We don’t want to duplicate the transfer logic across multiple places like `transfer()` and `transferFrom()`. Instead, we extract the core balance-changing logic into a single function — `_transfer()` — and reuse it wherever necessary.

> Think of transfer() as a frontend button, and _transfer() as the backend engine. The user only sees the button, but the real action happens behind the scenes.
> 

Another benefit of this approach is safety and consistency. If we ever want to change the way token balances are updated (e.g. adding a fee or logging extra data), we only need to modify `_transfer()` in one place — and both `transfer()` and `transferFrom()` will benefit from the update.

---

## _transfer()

```
function _transfer(address _from, address _to, uint256 _value) internal {
    require(_to != address(0), "Invalid address");
    balanceOf[_from] -= _value;
    balanceOf[_to] += _value;
    emit Transfer(_from, _to, _value);
}
```

This is the actual engine that moves tokens around.

It's marked `internal`, which means it can only be called from **within this contract or its derived contracts** — not by external users or other contracts. That’s a deliberate choice: we don’t want people calling `_transfer()` directly and bypassing important checks like allowances.

Here’s what it does:

- It makes sure the recipient address isn’t the zero address — because that would effectively burn the tokens.
- It subtracts the amount from the sender’s balance.
- It adds the same amount to the recipient.
- Then it emits a `Transfer` event so that wallets and frontends can reflect the change.

By moving this logic into its own internal function, we get:

- Clean, reusable code
- Fewer bugs from duplicated logic
- A single source of truth for how balances are updated

Both `transfer()` and `transferFrom()` rely on `_transfer()` to keep things consistent and DRY (Don’t Repeat Yourself).

---

## transferFrom()

```
function transferFrom(address _from, address _to, uint256 _value) public returns (bool) {
    require(balanceOf[_from] >= _value, "Not enough balance");
    require(allowance[_from][msg.sender] >= _value, "Allowance too low");

    allowance[_from][msg.sender] -= _value;
    _transfer(_from, _to, _value);
    return true;
}
```

This function allows someone who has been **approved** to actually move tokens on someone else’s behalf.

Here’s the typical flow:

1. Alice calls `approve(Bob, 100)`
2. Bob calls `transferFrom(Alice, Carol, 50)`

In this example:

- Bob is `msg.sender`
- Alice is `_from`
- Carol is `_to`

The function does three things:

- It checks that Alice actually has the tokens (`balanceOf[_from] >= _value`)
- It checks that Bob has been approved to spend at least that amount (`allowance[_from][msg.sender] >= _value`)
- It decreases Bob’s allowance by the amount
- Then it calls `_transfer()` to perform the actual token movement

This function is what makes things like DEX swaps and DAO voting possible — any time a smart contract moves your tokens, it’s probably using `transferFrom()` under the hood.

## approve()

```
function approve(address _spender, uint256 _value) public returns (bool) {
    allowance[msg.sender][_spender] = _value;
    emit Approval(msg.sender, _spender, _value);
    return true;
}
```

This function allows you to give another address (usually a smart contract) permission to spend tokens on your behalf.

Here’s a real-world analogy:

> You give your friend a signed permission slip to take $50 from your wallet. They can’t take more than that, and they need the slip to do it.
> 

Same here:

- `_spender` is the address you’re authorizing
- `_value` is the maximum amount they’re allowed to spend

The allowance is stored in a nested mapping: `allowance[owner][spender]`.
And just like with transfers, we emit an event — `Approval` — to let the outside world know this happened.

This is the foundation of all **delegated token movements** — like trading on a DEX, subscribing to a service, or participating in yield farming.

Note: This function only sets permission. The actual transfer is done via `transferFrom()`.

---

## 

## What’s Missing (And Why You Should Care)

Alright, we’ve built a working ERC-20 token — and that’s awesome.

But before you rush off to launch it on mainnet and start your own coin empire… there are a few things this minimal contract **doesn’t do** — and they’re important.

Here’s what’s missing:

- **No protection against `approve()` front-running**
    
    This is a well-known issue in ERC-20 where someone could exploit a race condition to spend more than they’re supposed to. The fix? Use `increaseAllowance()` and `decreaseAllowance()` instead — or other safer patterns.
    
- **No minting or burning**
    
    The total supply is fixed at deployment. You can’t increase it (mint) or reduce it (burn) without adding extra functions.
    
- **No access control or ownership**
    
    Anyone can interact with this contract — there’s no way to restrict who can mint, pause, or upgrade the contract.
    
- **No pausing or circuit breakers**
    
    If something goes wrong (like an exploit or bug), you can’t pause transfers or shut things down temporarily.
    
- **No upgradability or modularity**
    
    Once it’s deployed, it’s locked in. You can’t upgrade the logic unless you redeploy — which can be messy if your token is already in circulation.
    

So while this contract is **perfect for learning**, it’s not ready for the big leagues. If you’re building something real — a token that will hold real value — you need more robust tooling.

---

## Bonus: Use OpenZeppelin to Save Time (and Sanity)

In production, most developers don’t write everything from scratch.

They use battle-tested libraries — and the go-to choice in the Ethereum world is **OpenZeppelin**.

### Why OpenZeppelin?

Because it’s:

- Secure (used by major DeFi protocols)
- Audited and community-trusted
- Modular and customizable
- Easy to integrate with other contracts

Here’s how easy it is to spin up your own token using OpenZeppelin:

```solidity

// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts/token/ERC20/ERC20.sol";

contract MyToken is ERC20 {
    constructor(uint256 initialSupply) ERC20("MyToken", "MTK") {
        _mint(msg.sender, initialSupply * 10 ** decimals());
    }
}

```

That’s it. You’ve just created a production-grade ERC-20 token — in a few lines.

And if you want to go further, OpenZeppelin gives you **plug-and-play extensions** like:

- `ERC20Burnable`
- `ERC20Pausable`
- `AccessControl` or `Ownable`
- `ERC20Permit` for gasless approvals

You can pick what you need, compose it into your contract, and you’re good to go — with all the security and best practices baked in.

So when you’re done learning and ready to build for real — don’t reinvent the wheel. Use OpenZeppelin.

---

## Recap

Today, you learned:

- Why ERC-20 exists and how it standardized token behavior across Ethereum
- What `name`, `symbol`, `decimals`, and `totalSupply` do
- How balances and allowances are tracked
- How to approve and transfer tokens
- And how OpenZeppelin can help you do all this — securely and efficiently

With just a few lines of code, you now understand how 99% of tokens on Ethereum actually work.

Let’s keep going.