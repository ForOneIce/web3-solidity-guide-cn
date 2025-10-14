# Masterkey Contract

Alright, let’s switch things up a bit.

Up until now, you’ve probably been writing smart contracts one at a time — everything packed into a single file, doing its own thing.

But as your contracts get more complex, you’ll start to notice the repetition.

> “Didn’t I already write this ownership logic somewhere else?”
> 
> 
> “Why am I copying the same `onlyOwner` check again?”
> 

This is where Solidity gives you something powerful: **inheritance**.

---

### What Is Inheritance?

Let’s say your parent owns a house. One day, they pass it down to you.

Now when you inherit that house, you're not just getting the four walls — you're getting *everything* that comes with it:

- The furniture
- The rules (like “no shoes inside”)
- Maybe even the family dog

You didn’t set any of that up yourself — but you still have access to all of it.

That’s exactly what inheritance is like in Solidity.

One contract (the "parent") defines a bunch of logic — functions, variables, modifiers, etc.

Another contract (the "child") inherits it all — and can use it as-is, or modify parts of it to suit its needs.

---

### Why Use It?

Inheritance helps you:

- Avoid writing the same logic in multiple places
- Split your code into smaller, focused pieces
- Reuse important features like access control or utility functions
- Make your contracts easier to update and maintain

It’s the backbone of most large-scale Solidity projects

---

### What If You Want to Change Something?

Let’s go back to that house example.

You inherit your parent’s house. Along with it, you get everything that comes with it — the furniture, the design, and even the rules they followed.

But what if you don’t want it all exactly the same?

Maybe you want to renovate the kitchen. Or change the “no pets” rule. It’s still the same house — but you’re making it your own.

**That’s exactly how customization works in Solidity when you inherit a contract.**

You get all the functions from the parent contract — but sometimes, you’ll want to **change how one of those functions behaves** in the new contract.

Solidity gives you a safe way to do that using two keywords:

- **`virtual`** goes in the **parent contract**. It marks a function as *changeable*. It’s like the parent saying,
    
    > “Here’s the rule — but feel free to change it if you need to.”
    > 
- **`override`** goes in the **child contract**. It tells Solidity,
    
    > “I know this function was inherited, but I’m replacing it with my own version.”
    > 

You **must** use both — Solidity won’t let you accidentally override something unless the parent explicitly allows it.

This keeps your contracts clean, predictable, and easier to maintain. You know exactly what’s inherited and what’s been changed — no surprises.

So in short:

Inheritance gives you everything from the parent, but with `virtual` and `override`, you get the flexibility to **customize** just the parts you need.

This way, Solidity keeps things safe and clear — no silent overrides, no accidental changes. Everything’s intentional.

---

### What We’re Building Today

You’re going to write two contracts that show off inheritance in action:

1. **Ownable.sol** – A simple contract that defines who the owner is and gives you a reusable `onlyOwner` check to protect sensitive functions.
2. **VaultMaster.sol** – A vault that accepts ETH deposits from anyone, but only lets the **owner** withdraw. Instead of writing the ownership logic again, it will simply **inherit** it from `Ownable`.

This is your first step into writing clean, modular, and production-style Solidity code.

Let’s dive in.

---

## The Plan

We’re going to write **two contracts**:

1. **Ownable.sol** – This contract keeps track of who the owner is, and protects sensitive functions using an `onlyOwner` modifier.
2. **VaultMaster.sol** – A vault that lets users deposit ETH, but **only the owner** can withdraw funds.

And here’s the best part: `VaultMaster` will **inherit** everything from `Ownable`, without rewriting a single line of that logic.

Let’s start with the foundation.

---

## Contract 1: Ownable.sol — Our Reusable Access Control

```solidity
 
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract Ownable {
    address private owner;

    event OwnershipTransferred(address indexed previousOwner, address indexed newOwner);

    constructor() {
        owner = msg.sender;
        emit OwnershipTransferred(address(0), msg.sender);
    }

    modifier onlyOwner() {
        require(msg.sender == owner, "Only owner can perform this action");
        _;
    }

    function ownerAddress() public view returns (address) {
        return owner;
    }

    function transferOwnership(address _newOwner) public onlyOwner {
        require(_newOwner != address(0), "Invalid address");
        address previous = owner;
        owner = _newOwner;
        emit OwnershipTransferred(previous, _newOwner);
    }
}

```

Let’s walk through it piece by piece.

---

### `address private owner;`

We store the address of the contract owner — the person who deployed the contract.

It’s marked `private`, so it’s only accessible inside this contract. Other contracts can’t access it directly.

### `event OwnershipTransferred(...)`

```solidity
 
event OwnershipTransferred(address indexed previousOwner, address indexed newOwner);

```

Events in Solidity are like logs — they’re stored on the blockchain, and your frontend can **listen** for them. They're not used for changing the state of the contract — they just let the outside world know that something happened.

Here, we log who the ownership was transferred **from** and **to**.

The `indexed` keyword helps filter logs easily — so you can search for all events involving a specific address.

---

### `constructor()`

```solidity
 
constructor() {
    owner = msg.sender;
    emit OwnershipTransferred(address(0), msg.sender);
}

```

This runs once when the contract is deployed. It sets the deployer (`msg.sender`) as the initial owner and emits the `OwnershipTransferred` event to log that change.

---

### 

### `modifier onlyOwner()`

```solidity
 
modifier onlyOwner() {
    require(msg.sender == owner, "Only owner can perform this action");
    _;
}

```

This is a custom modifier. When applied to a function, it **ensures only the owner can call it**.

If the `require` fails, the function call stops there.

---

### `ownerAddress()`

```solidity
 
function ownerAddress() public view returns (address) {
    return owner;
}

```

Since `owner` is private, we provide a public function so anyone can check who the current owner is.

---

### `transferOwnership()`

```solidity
 
function transferOwnership(address _newOwner) public onlyOwner {
    require(_newOwner != address(0), "Invalid address");
    address previous = owner;
    owner = _newOwner;
    emit OwnershipTransferred(previous, _newOwner);
}

```

This function allows the current owner to **transfer ownership** to someone else.

- It uses the `onlyOwner` modifier for protection.
- It checks that the new owner address is valid (not `0x0`).
- it stores the current owner addresss in the `previous`  variable
- Then it updates the owner and logs the change with the `OwnershipTransferred` event.

---

### So, what have we built?

A **reusable contract** that:

- Tracks the current owner
- Restricts access to sensitive functions
- Allows ownership to be transferred
- Emits events so ownership changes are logged publicly

But this contract doesn’t actually do anything *on its own*. It’s meant to be **inherited** by other contracts — like our next one.

---

## Contract 2: VaultMaster.sol — A Simple ETH Vault with Ownership

```solidity
 
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "./ownable.sol";

contract VaultMaster is Ownable {
    event DepositSuccessful(address indexed account, uint256 value);
    event WithdrawSuccessful(address indexed recipient, uint256 value);

    function getBalance() public view returns (uint256) {
        return address(this).balance;
    }

    function deposit() public payable {
        require(msg.value > 0, "Enter a valid amount");
        emit DepositSuccessful(msg.sender, msg.value);
    }

    function withdraw(address _to, uint256 _amount) public onlyOwner {
        require(_amount <= getBalance(), "Insufficient balance");

        (bool success, ) = payable(_to).call{value: _amount}("");
        require(success, "Transfer Failed");

        emit WithdrawSuccessful(_to, _amount);
    }
}

```

This contract is simple but powerful.

Let’s break it down:

---

### `import "./ownable.sol";`

This line brings in our `Ownable` contract from the other file.

Now everything defined in `Ownable` — including the `onlyOwner` modifier and `transferOwnership()` — is available in `VaultMaster`.

---

### `contract VaultMaster is Ownable`

This is the key line. It says:

> "VaultMaster inherits from Ownable."
> 

That means `VaultMaster` now **automatically has** all the functions, variables, and modifiers from `Ownable`.

---

### Events

Here are the two event declarations:

```solidity

event DepositSuccessful(address indexed account, uint256 value);
event WithdrawSuccessful(address indexed recipient, uint256 value);

```

**Breakdown:**

- `DepositSuccessful` is triggered when someone sends ETH to the contract.
    - `account`: the sender’s address (marked `indexed` so it can be easily filtered in logs).
    - `value`: the amount of ETH deposited.
- `WithdrawSuccessful` is triggered when the owner withdraws ETH from the contract.
    - `recipient`: the address receiving the funds (also `indexed`).
    - `value`: the amount withdrawn.

These events are emitted inside the `deposit()` and `withdraw()` functions to keep a public log of each transaction — useful for frontends, dashboards, and off-chain systems.

---

### `getBalance()`

```solidity
 
function getBalance() public view returns (uint256) {
    return address(this).balance;
}

```

Returns how much ETH the contract currently holds.

---

### `deposit()`

```solidity
 
function deposit() public payable   {
    require(msg.value > 0, "Enter a valid amount");
    emit DepositSuccessful(msg.sender, msg.value);
}

```

Allows anyone to send ETH into the contract.

We check that they’re actually sending something, and log the deposit.

This function lets **anyone** send ETH into the contract — it's open to the public.

We then use a `require` statement to make sure the person is actually sending some ETH — we don’t want zero-value deposits.

If the check passes, the function emits the `DepositSuccessful` event, logging:

- `msg.sender` (the sender’s address),
- and `msg.value` (the amount of ETH sent).

---

### `withdraw()`

```solidity
 
function withdraw(address _to, uint256 _amount) public onlyOwner {
    require(_amount <= getBalance(), "Insufficient balance");

    (bool success, ) = payable(_to).call{value: _amount}("");
    require(success, "Transfer Failed");

    emit WithdrawSuccessful(_to, _amount);
}

```

This function allows ETH to be withdrawn from the contract — but only by the owner.

The key here is the `onlyOwner` modifier. We didn’t define it inside this contract — we inherited it from the `Ownable` contract.

That’s the power of inheritance: we can reuse logic like access control without rewriting anything. The `onlyOwner` modifier checks if the caller is the current owner and blocks anyone else from executing the function.

Inside the function:

- We first check that the contract has enough ETH to withdraw.
- Then we send the specified amount to the given address using `.call`.
- If the transfer goes through, we emit the `WithdrawSuccessful` event to log the action.

---

## Bonus: Using OpenZeppelin's Ownable Instead

Alright, so we just built our own `Ownable` contract from scratch — and that’s a solid step in learning how access control works.

But in the real world, most developers don’t write everything from scratch. They rely on **trusted libraries** 

### What is OpenZeppelin?

**OpenZeppelin** is a team of top-tier smart contract developers who’ve created a library of secure, reusable, and community-audited contracts for Ethereum.

Their contracts handle things like:

- Access control (`Ownable`, `AccessControl`)
- Token standards (`ERC20`, `ERC721`, `ERC1155`)
- Security patterns (pausable, reentrancy guards)
- Proxies and upgradeable contracts
- Utilities and math libraries

These contracts are used across the Ethereum ecosystem — by protocols, DAOs, NFT platforms, DeFi apps, and pretty much anyone writing production-grade smart contracts.

**Why?** Because they’re:

- Battle-tested in the wild
- Continuously maintained
- Audited and trusted

Using OpenZeppelin saves time, reduces bugs, and helps you build safer contracts.

---

## How to Use OpenZeppelin’s `Ownable` in Remix

Since we’re working inside **Remix**, you can use OpenZeppelin right away — no installation needed.

---

### Step 1: Replace the import

In your `VaultMaster.sol`, instead of importing your custom `Ownable` contract from a local file, you can directly use the official OpenZeppelin version like this:

```solidity

import "@openzeppelin/contracts/access/Ownable.sol";

```

So, what’s with the `@` symbol?

The `@openzeppelin` part is a shorthand used in Remix (and in many Solidity projects) to refer to **packages** from the **npm-style package system**.

Even though we’re not explicitly installing anything in Remix, it still recognizes and resolves this path behind the scenes.

Think of it like this:

- `@openzeppelin/...` = “grab this from the OpenZeppelin library package”
- `contracts/access/Ownable.sol` = the actual folder and file path inside that package

So this line is telling Remix:

> “Hey, pull the Ownable.sol file from the contracts/access directory inside the OpenZeppelin package.”
> 

Remix automatically fetches OpenZeppelin contracts when you reference them this way, so you don’t need to set up anything manually.

It just works out of the box — which is super convenient when you're prototyping or learning.

---

### Step 2: Pass the owner in the constructor

OpenZeppelin’s version of `Ownable` expects you to **pass the initial owner** when the contract is deployed. So, inside your contract, add this constructor:

```solidity

constructor() Ownable(msg.sender) {}

```

This tells OpenZeppelin to set the deployer as the first owner — just like we did manually before.

---

### Final VaultMaster with OpenZeppelin

Here’s your updated `VaultMaster.sol` using OpenZeppelin’s `Ownable`:

```solidity

// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts/access/Ownable.sol";

contract VaultMaster is Ownable {

    event DepositSuccessful(address indexed account, uint256 value);
    event WithdrawSuccessful(address indexed recipient, uint256 value);

    // Set deployer as owner using OpenZeppelin's constructor
    constructor() Ownable(msg.sender) {}

    function getBalance() public view returns (uint256) {
        return address(this).balance;
    }

    function deposit() public payable {
        require(msg.value > 0, "Enter a valid amount");
        emit DepositSuccessful(msg.sender, msg.value);
    }

    function withdraw(address _to, uint256 _amount) public onlyOwner {
        require(_amount <= getBalance(), "Insufficient balance");

        (bool success, ) = payable(_to).call{value: _amount}("");
        require(success, "Transfer failed");

        emit WithdrawSuccessful(_to, _amount);
    }
}

```

---

## So why bother with OpenZeppelin?

Because in the long run:

- You write **less code**
- You get **fewer bugs**
- And your contracts follow **industry best practices**

It's like using reliable building blocks instead of starting from raw material every time. You can focus on the unique logic of your app — and leave the foundational work to the experts.

---

## Recap

Today, you learned:

- How to write a **base contract** that controls ownership
- How to **inherit** that contract in another
- How to use `modifier`s and `event`s to add control and visibility
- How to structure your Solidity projects more cleanly
- And how to use **OpenZeppelin’s Ownable** for real-world use

Inheritance makes your smart contracts more **modular**, **reusable**, and **maintainable**.

This is how smart contract systems are built in the real world — one small, composable piece at a time.