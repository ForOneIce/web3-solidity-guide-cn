# SafeDeposit contract

Alright — so far, we’ve written basic and advanced ERC-20 tokens, launched a full token sale, and explored how **inheritance** can help us reuse and organize code. Think about how our TokenSale contract extended the ERC-20 token to add new logic without rewriting everything.

Now we’re going to level up again — this time by building a **modular vault system** that lets users store sensitive secrets on-chain in different types of deposit boxes.

---

## 🧠 The Idea

Picture this:

You’ve got users, and each of them wants to store a secret in a personal vault. But not all vaults are the same:

- Some are **basic**, meant for everyday use.
- Some are **premium**, offering bonus features like metadata.
- Some are **time-locked**, where the user can’t open the vault until a certain time has passed.

Instead of writing three completely separate contracts, we’re going to **build them smartly** — using shared rules, common logic, and Solidity’s powerful object-oriented features.

That brings us to one key concept we need to understand before diving in:

---

## 📘 What’s an Interface in Solidity?

If inheritance is about **reusing common logic**, interfaces are about **enforcing structure**.

An **interface** in Solidity is basically a contract with *only function definitions* — no logic, no storage, no state variables.

Think of it as a blueprint or a promise:

> “Any contract that implements this interface must include these functions — and they must behave as expected.”
> 

It’s like setting a standard. And that has major benefits:

- ✅ It makes contract interactions predictable — even across different types
- ✅ It lets us write tools (like managers or dashboards) that talk to *any* contract that follows the interface — without knowing the full implementation
- ✅ It keeps our system modular and flexible

We’re going to use an interface to define what every vault box must support:

- Who owns the box?
- Can they store and retrieve a secret?
- What type of box is it?
- When was it created?

Each vault type will then implement this interface in its own way.

---

## 🧱 What’s an Abstract Contract?

Alongside interfaces, Solidity gives us another powerful tool: **abstract contracts**.

An abstract contract is a contract that can have both **implemented functions** and **unimplemented (virtual) functions** — basically a mix of interface and real logic.

It’s useful when you want to:

- Provide shared logic (like common variables or internal helper functions)
- Leave certain pieces to be filled in by child contracts

Here’s the key idea:

> We use abstract contracts as foundations — they are not meant to be deployed directly.
> 

In our vault system, we’ll build an abstract base contract that handles shared logic like:

- Tracking the owner of a vault
- Storing and retrieving secrets
- Enforcing access control

Then our `Basic`, `Premium`, and `TimeLocked` vaults will **extend** this base contract and add their own flavor — all while conforming to the interface.

---

## 🛠️ What We’re Going to Build

We’ll take this in clear, digestible steps:

### 1. `IDepositBox.sol` — The Interface

We’ll define a simple rulebook of required functions. Every vault will be required to follow this.

### 2. `BaseDepositBox.sol` — The Abstract Contract

This will be our shared foundation. It’ll implement most of the logic defined in the interface, like secret storage, ownership, and deposit time.

It’ll also use `modifiers` to restrict access — for example, making sure only the owner can access or modify the vault.

### 3. `BasicDepositBox.sol` — The Standard Vault

This vault just does what the interface says. Nothing fancy. It’s the default vault.

### 4. `PremiumDepositBox.sol` — A Vault With Extras

This one builds on the base vault but adds support for custom metadata. Think of it as an upgrade with more features.

### 5. `TimeLockedDepositBox.sol` — A Vault You Can’t Open Yet

This one is time-sensitive. The secret can’t be revealed until a lock period expires.

We’ll add special modifiers to enforce that behavior.

### 6. `VaultManager.sol` — The Controller

Finally, we’ll tie it all together with a `VaultManager` contract.

This contract:

- Lets users create any type of vault
- Keeps track of which boxes belong to which users
- Allows vaults to be renamed
- Handles ownership transfers
- Acts as a single point of interaction for the whole system

---

## Why This is Important

This system shows how you can write scalable, modular, and well-organized Solidity code — just like large-scale real-world protocols do.

It also teaches how **interfaces**, **abstract contracts**, and **inheritance** work together — forming the core of most production-grade smart contracts.

---

Let’s dive in by writing our interface: `IDepositBox.sol`. Ready? Let’s start building vaults.

## `IDepositBox` — Our Vault’s Interface

```solidity
 
interface IDepositBox {
    function getOwner() external view returns (address);
    function transferOwnership(address newOwner) external;
    function storeSecret(string calldata secret) external;
    function getSecret() external view returns (string memory);
    function getBoxType() external pure returns (string memory);
    function getDepositTime() external view returns (uint256);
}

```

### What’s Going On Here?

This is our **interface** — the contract blueprint.

We’re basically saying: "Hey, any vault (or deposit box) that wants to be part of our system **must** implement these functions."

Let’s go over what each function represents:

- `getOwner()` — returns the current owner of the box.
- `transferOwnership()` — allows transferring ownership to someone else.
- `storeSecret()` — a function to save a string (our “secret”) inside the vault.
- `getSecret()` — retrieves the stored secret.
- `getBoxType()` — lets us know what kind of box it is (Basic, Premium, etc.).
- `getDepositTime()` — returns when the box was created.

> These are like the rules of the game — every deposit box type will follow them, even if they implement them differently.
> 

## `BaseDepositBox.sol` – Our Abstract Foundation

This contract is the **core** of our deposit box system. All specific types of boxes — like Basic, Premium, and TimeLocked — are built on top of this contract.

Let’s start by understanding the top-level structure.

```json
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "./IDepositBox.sol";

abstract contract BaseDepositBox is IDepositBox {
    address private owner;
    string private secret;
    uint256 private depositTime;

    event OwnershipTransferred(address indexed previousOwner, address indexed newOwner);
    event SecretStored(address indexed owner);

    modifier onlyOwner() {
        require(msg.sender == owner, "Not the box owner");
        _;
    }

    constructor() {
        owner = msg.sender;
        depositTime = block.timestamp;
    }

    function getOwner() public view override returns (address) {
        return owner;
    }

    function transferOwnership(address newOwner) external virtual  override onlyOwner {
        require(newOwner != address(0), "New owner cannot be zero address");
        emit OwnershipTransferred(owner, newOwner);
        owner = newOwner;
    }

    function storeSecret(string calldata _secret) external virtual override onlyOwner {
        secret = _secret;
        emit SecretStored(msg.sender);
    }

    function getSecret() public view virtual override onlyOwner returns (string memory) {
        return secret;
    }

    function getDepositTime() external view virtual  override returns (uint256) {
        return depositTime;
    }
}

```

### Import Statement

```solidity
 
import "./IDepositBox.sol";

```

We're importing an interface called `IDepositBox`. Think of an interface as a **contract that only has function declarations**, no actual implementation. By importing and inheriting from it, this contract is saying:

> “I promise to implement all the functions declared in IDepositBox — even if not all of them are written here yet.”
> 

This helps us enforce structure across all deposit boxes.

---

### Abstract Contract

```solidity
 
abstract contract BaseDepositBox is IDepositBox

```

The keyword `abstract` means this contract **cannot be deployed directly**. It’s designed to act like a **template** or **foundation** for other contracts to build on.

Why is it abstract?

Because it doesn’t fully implement every function required by the interface. For example, it **doesn’t define** the `getBoxType()` function — each child contract will define that differently (like "Basic", "Premium", etc.).

So this base contract handles the **common logic**, and each specific box fills in the rest.

---

## 🧱 State Variables

```solidity
 
address private owner;
string private secret;
uint256 private depositTime;

```

Let’s break these down:

- `owner`: Stores the address of the person who owns this deposit box. Only this person is allowed to store or retrieve secrets.
- `secret`: A private string that the user can store securely in this box.
- `depositTime`: Records the exact time (Unix timestamp) when the box was deployed. This can be useful for time-based logic (e.g., locks).

All of these are marked `private`, meaning they can only be accessed internally. If someone wants to read them, they must go through a **public getter function** (which we provide).

---

## 📣 Events

```solidity
 
event OwnershipTransferred(address indexed previousOwner, address indexed newOwner);
event SecretStored(address indexed owner);

```

Events help log important on-chain activity. These are useful for frontends and tools like Etherscan or TheGraph.

- `OwnershipTransferred`: Emitted when someone transfers ownership of the deposit box.
- `SecretStored`: Emitted when a new secret is stored.

The keyword `indexed` makes it easier to filter logs by those fields when querying on-chain data.

---

## 🔐 Modifier: `onlyOwner`

```solidity
 
modifier onlyOwner() {
    require(msg.sender == owner, "Not the box owner");
    _;
}

```

This modifier restricts access to certain functions. If a function is marked with `onlyOwner`, then only the current owner can run it.

Otherwise, the function call reverts with the message `"Not the box owner"`.

We use this modifier in every sensitive function — like storing secrets or transferring ownership.

---

## 🧰 Constructor

```solidity
 
constructor() {
    owner = msg.sender;
    depositTime = block.timestamp;
}

```

This function runs once — right when the box is deployed.

- `msg.sender`: The person who deployed the contract becomes the `owner`.
- `block.timestamp`: The current time (in seconds since Unix epoch) is recorded as the deposit time.

So the box sets up both ownership and time tracking automatically at creation.

---

## 👤 `getOwner()`

```solidity
 
function getOwner() public view override returns (address) {
    return owner;
}

```

Returns the current owner of the box. It’s a simple getter function — and required by the `IDepositBox` interface.

---

## 🔄 `transferOwnership()`

```solidity
 
function transferOwnership(address newOwner) external virtual override onlyOwner {
    require(newOwner != address(0), "New owner cannot be zero address");
    emit OwnershipTransferred(owner, newOwner);
    owner = newOwner;
}

```

Allows the current owner to hand over ownership to someone else.

- It checks that the new owner address isn’t the zero address (`0x000...0`) — which would lock the box forever.
- Emits an event to signal the ownership change.

The `onlyOwner` modifier ensures that **only the current owner** can transfer ownership.

---

## 🧳 `storeSecret()`

```solidity
 
function storeSecret(string calldata _secret) external virtual override onlyOwner {
    secret = _secret;
    emit SecretStored(msg.sender);
}

```

This is how the owner stores a string in the box.

- It only accepts a call from the current owner (`onlyOwner`).
- Stores the secret string in the private variable `secret`.
- Emits an event after storing.

We use `calldata` here because it's cheaper on gas when passing in string arguments.

---

## 🔍 `getSecret()`

```solidity
 
function getSecret() public view virtual override onlyOwner returns (string memory) {
    return secret;
}

```

This function lets the owner retrieve the stored secret.

- It’s a `view` function, so it doesn’t cost gas if called externally (e.g. from a frontend).
- Marked with `onlyOwner` to ensure privacy — no one else can see the secret.

---

## 🕒 `getDepositTime()`

```solidity
 
function getDepositTime() external view virtual override returns (uint256) {
    return depositTime;
}

```

Returns the time the box was deployed. Useful for time-based features — for example, if a secret should only be revealed after a certain number of days.

## 🧱 `BasicDepositBox.sol` — The Simplest Extension

Alright, now that we’ve built a solid foundation with our `BaseDepositBox` contract, it’s time to start creating specific types of deposit boxes.

We’ll start with the most straightforward one: the `BasicDepositBox`.

Let’s look at the full code first:

```solidity
  
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "./BaseDepositBox.sol";

contract BasicDepositBox is BaseDepositBox {
    function getBoxType() external pure override returns (string memory) {
        return "Basic";
    }
}

```

At first glance, this contract looks super small — and it is. But behind the scenes, it’s incredibly powerful because of everything it inherits from `BaseDepositBox`.

---

## 🔄 Inheriting from `BaseDepositBox`

```solidity
  
import "./BaseDepositBox.sol";

```

We start by importing the base contract. This gives `BasicDepositBox` access to **everything** defined in the base:

- Ownership management
- Secret storing and retrieving
- Deposit time tracking
- Access control modifiers
- Events

Then we define our new contract like this:

```solidity
  
contract BasicDepositBox is BaseDepositBox

```

This means: “Make a new contract that **inherits** all the logic from `BaseDepositBox`.” So we don’t need to rewrite all that logic — we simply reuse it.

This is the power of inheritance in Solidity: you write core features once in a parent contract, and extend them as needed in child contracts.

---

## 🏷️ `getBoxType()` — Identifying the Box

```solidity
  
function getBoxType() external pure override returns (string memory) {
    return "Basic";
}

```

This is the only function this contract defines, and it's here to fulfill a promise made to the `IDepositBox` interface.

Let’s break it down:

- `external`: This function is only meant to be called from outside the contract (e.g. from another contract or a frontend).
- `pure`: It doesn’t read or write any storage — it simply returns a hardcoded string.
- `override`: It’s overriding the abstract `getBoxType()` function declared in `IDepositBox` (and left unimplemented in `BaseDepositBox`).
- `returns (string memory)`: It gives us the box type — which in this case is `"Basic"`.

Why is this useful?

Let’s say you have many boxes deployed — Basic, Premium, and TimeLocked — and you want to know what type each box is. You can simply call `getBoxType()` and get a human-readable label.

This is especially helpful when you're building things like:

- A frontend dashboard to show your boxes
- Sorting logic in the `VaultManager` contract
- On-chain analytics

---

## 🧠 What This Contract Actually Does

Even though it's just a few lines long, `BasicDepositBox` gives us a fully functional, secure, owner-controlled deposit box.

It can:

- Store a secret string (only the owner can set or retrieve it)
- Transfer ownership to someone else
- Emit events when secrets are stored or owners are changed
- Report its own box type as `"Basic"`

And all of that power comes from the **base contract** — we just added one specific label to identify this type of box.

## 💎 `PremiumDepositBox.sol` — With Extra Metadata Power

This time, we’re building on top of our `BaseDepositBox` again — but now we’re giving the deposit box **something extra**: a piece of data called `metadata`.

Think of this like a personalized label, tag, or info field that the owner can set. It could describe what the secret is about, when it should be accessed, or any other note you want to attach.

Here’s the full contract:

```solidity
  
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "./BaseDepositBox.sol";

contract PremiumDepositBox is BaseDepositBox {
    string private metadata;

    event MetadataUpdated(address indexed owner);

    function getBoxType() external pure override returns (string memory) {
        return "Premium";
    }

    function setMetadata(string calldata _metadata) external onlyOwner {
        metadata = _metadata;
        emit MetadataUpdated(msg.sender);
    }

    function getMetadata() external view onlyOwner returns (string memory) {
        return metadata;
    }
}

```

Now let’s walk through every part of this step by step.

---

## ✅ Inheritance and Import

```solidity
  
import "./BaseDepositBox.sol";

contract PremiumDepositBox is BaseDepositBox {

```

As before, we import the base logic from `BaseDepositBox`. This gives us access to:

- Ownership checks
- Secret storage
- Event logging
- Deposit timestamp

And we **extend** it by declaring `PremiumDepositBox` as a child of `BaseDepositBox`.

This means we can **reuse all the core functionality** and focus only on what makes this version different — which is the metadata.

---

## 🧾 `metadata` — A New Private Field

```solidity
  
string private metadata;

```

We introduce a new state variable called `metadata`.

It’s marked `private`, which means only functions **within this contract** can read or modify it. (We’ll create external access functions for the owner.)

This could be used for anything — a label like “Backup Keys,” or a note like “Access after retirement,” etc.

---

## 📢 `MetadataUpdated` Event

```solidity
  
event MetadataUpdated(address indexed owner);

```

Just like we emit events when someone stores a secret or transfers ownership, we do the same here when someone updates metadata.

Why? Because:

- It helps off-chain systems (frontends, explorers) track changes
- It improves transparency
- It makes debugging easier

We also mark `owner` as `indexed` so it can be easily filtered or searched in the logs.

---

## 🏷️ `getBoxType()` — Identify the Box

```solidity
  
function getBoxType() external pure override returns (string memory) {
    return "Premium";
}

```

Same as before — this tells us what type of box this is.

It returns `"Premium"`, which helps other contracts or frontends identify and label it properly.

This is required by the `IDepositBox` interface and helps us distinguish between different box types (Basic, Premium, TimeLocked).

---

## ✍️ `setMetadata()` — Owner Can Add Info

```solidity
  
function setMetadata(string calldata _metadata) external onlyOwner {
    metadata = _metadata;
    emit MetadataUpdated(msg.sender);
}

```

Let’s unpack it:

- `external`: Only callable from outside the contract (not internally)
- `onlyOwner`: Uses our modifier from `BaseDepositBox` to restrict access — only the box owner can update metadata
- `string calldata _metadata`: The new metadata value is passed in as an argument
- `metadata = _metadata`: We update the stored value
- `emit MetadataUpdated(msg.sender)`: We log the change for transparency

This gives owners a way to attach notes, categories, or tags to their deposit box.

---

## 📬 `getMetadata()` — Owner Can Read It

```solidity
  
function getMetadata() external view onlyOwner returns (string memory) {
    return metadata;
}

```

This function lets the owner **retrieve** the metadata they previously stored.

It’s `view` because it just reads from storage without changing anything.

And again, it’s protected by `onlyOwner` — no one else can see what you’ve written.

## ⏳ `TimeLockedDepositBox.sol` — Secrets You Can’t Open Just Yet

This contract introduces a twist: **you can store a secret, but you can’t retrieve it until a specific time has passed**.

It’s like a digital safe that stays locked for a set period — no matter how badly you want to open it, you’ve got to wait.

We inherit from our trusty `BaseDepositBox` again, but this time, we add **timing logic** to control when the secret becomes accessible.

Let’s go through the contract in full, then explain it line by line.

---

### ✅ Full Contract

```solidity
  
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "./BaseDepositBox.sol";

contract TimeLockedDepositBox is BaseDepositBox {
    uint256 private unlockTime;

    constructor(uint256 lockDuration) {
        unlockTime = block.timestamp + lockDuration;
    }

    modifier timeUnlocked() {
        require(block.timestamp >= unlockTime, "Box is still time-locked");
        _;
    }

    function getBoxType() external pure override returns (string memory) {
        return "TimeLocked";
    }

    function getSecret() public view override onlyOwner timeUnlocked returns (string memory) {
        return super.getSecret();
    }

    function getUnlockTime() external view returns (uint256) {
        return unlockTime;
    }

    function getRemainingLockTime() external view returns (uint256) {
        if (block.timestamp >= unlockTime) return 0;
        return unlockTime - block.timestamp;
    }
}

```

---

## 🔄 Inheritance and Import

```solidity
  
import "./BaseDepositBox.sol";

contract TimeLockedDepositBox is BaseDepositBox {

```

Just like `BasicDepositBox` and `PremiumDepositBox`, we’re importing the shared base logic from `BaseDepositBox`.

We inherit all the core functionality:

- Owner tracking
- Secret storage
- Deposit time
- Event logging

And we’ll add the time-lock feature on top of that.

---

## 🧭 `unlockTime` — The Lock Timer

```solidity
  
uint256 private unlockTime;

```

This variable stores the **timestamp** when the secret becomes accessible.

It's `private`, meaning no one can access it directly — but we’ll expose it via a getter function.

---

## 🏗️ Constructor — Locking Things Up

```solidity
  
constructor(uint256 lockDuration) {
    unlockTime = block.timestamp + lockDuration;
}

```

Here’s what’s happening:

- When the contract is deployed, the constructor takes a number: `lockDuration`, in **seconds**.
- It adds that duration to the current timestamp (`block.timestamp`) and sets that as the `unlockTime`.

So if you pass in `3600` (1 hour), the secret will stay locked for 1 hour from deployment.

This makes the contract flexible — you can lock secrets for minutes, hours, days, or years.

---

## 🔐 `timeUnlocked` Modifier — Access Gatekeeper

```solidity
  
modifier timeUnlocked() {
    require(block.timestamp >= unlockTime, "Box is still time-locked");
    _;
}

```

This is a **custom modifier** — just like `onlyOwner` — but it checks whether the current time has passed the unlock time.

If it hasn’t, the function call is rejected with an error.

We’ll use this modifier on any function that should be protected by the time-lock.

---

## 📦 `getBoxType()`

```solidity
  
function getBoxType() external pure override returns (string memory) {
    return "TimeLocked";
}

```

Just like the other box types, this helps us identify what kind of deposit box this is.

It returns the string `"TimeLocked"` and satisfies the `IDepositBox` interface.

---

## 🔍 `getSecret()` — But Only After the Lock

```solidity
  
function getSecret() public view override onlyOwner timeUnlocked returns (string memory) {
    return super.getSecret();
}

```

This function overrides the regular `getSecret()` function from `BaseDepositBox`.

But now, it adds **two** access checks:

1. `onlyOwner`: Only the box owner can view the secret.
2. `timeUnlocked`: Only after the unlock time has passed.

Then it calls `super.getSecret()` to retrieve the actual secret from the base contract.

This layering keeps our code clean and avoids duplicating logic.

---

## 📅 `getUnlockTime()` — When Does It Unlock?

```solidity
  
function getUnlockTime() external view returns (uint256) {
    return unlockTime;
}

```

This is a simple getter function that tells you the **exact timestamp** when the box becomes unlockable.

Useful for frontends and display purposes.

---

## ⏱️ `getRemainingLockTime()` — Countdown Helper

```solidity
  
function getRemainingLockTime() external view returns (uint256) {
    if (block.timestamp >= unlockTime) return 0;
    return unlockTime - block.timestamp;
}

```

This is another frontend-friendly helper.

- If the current time is already past the unlock time, we return `0`.
- Otherwise, we subtract `now` from `unlockTime` and return the number of **seconds left** until it can be opened.

Great for countdowns, timers, or visualizing progress bars in your UI.

## 🧱 VaultManager — Your Deposit Box Dashboard

This contract acts like the **control center** for users to create, name, manage, and interact with their deposit boxes.

Think of it as your **vault app backend**:

- It lets users create different types of boxes (basic, premium, time-locked).
- It keeps track of which user owns which boxes.
- It enforces ownership rules.
- It provides helper functions for naming and retrieving box info.

---

## ✅ Full Contract Code (for reference)

```solidity
   
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "./IDepositBox.sol";
import "./BasicDepositBox.sol";
import "./PremiumDepositBox.sol";
import "./TimeLockedDepositBox.sol";

contract VaultManager {
    mapping(address => address[]) private userDepositBoxes;
    mapping(address => string) private boxNames;

    event BoxCreated(address indexed owner, address indexed boxAddress, string boxType);
    event BoxNamed(address indexed boxAddress, string name);

    function createBasicBox() external returns (address) {
        BasicDepositBox box = new BasicDepositBox();
        userDepositBoxes[msg.sender].push(address(box));
        emit BoxCreated(msg.sender, address(box), "Basic");
        return address(box);
    }

    function createPremiumBox() external returns (address) {
        PremiumDepositBox box = new PremiumDepositBox();
        userDepositBoxes[msg.sender].push(address(box));
        emit BoxCreated(msg.sender, address(box), "Premium");
        return address(box);
    }

    function createTimeLockedBox(uint256 lockDuration) external returns (address) {
        TimeLockedDepositBox box = new TimeLockedDepositBox(lockDuration);
        userDepositBoxes[msg.sender].push(address(box));
        emit BoxCreated(msg.sender, address(box), "TimeLocked");
        return address(box);
    }

    function nameBox(address boxAddress, string calldata name) external {
        IDepositBox box = IDepositBox(boxAddress);
        require(box.getOwner() == msg.sender, "Not the box owner");

        boxNames[boxAddress] = name;
        emit BoxNamed(boxAddress, name);
    }

    function storeSecret(address boxAddress, string calldata secret) external {
        IDepositBox box = IDepositBox(boxAddress);
        require(box.getOwner() == msg.sender, "Not the box owner");

        box.storeSecret(secret);
    }

    function transferBoxOwnership(address boxAddress, address newOwner) external {
        IDepositBox box = IDepositBox(boxAddress);
        require(box.getOwner() == msg.sender, "Not the box owner");

        box.transferOwnership(newOwner);

        address[] storage boxes = userDepositBoxes[msg.sender];
        for (uint i = 0; i < boxes.length; i++) {
            if (boxes[i] == boxAddress) {
                boxes[i] = boxes[boxes.length - 1];
                boxes.pop();
                break;
            }
        }

        userDepositBoxes[newOwner].push(boxAddress);
    }

    function getUserBoxes(address user) external view returns (address[] memory) {
        return userDepositBoxes[user];
    }

    function getBoxName(address boxAddress) external view returns (string memory) {
        return boxNames[boxAddress];
    }

    function getBoxInfo(address boxAddress) external view returns (
        string memory boxType,
        address owner,
        uint256 depositTime,
        string memory name
    ) {
        IDepositBox box = IDepositBox(boxAddress);
        return (
            box.getBoxType(),
            box.getOwner(),
            box.getDepositTime(),
            boxNames[boxAddress]
        );
    }
}

```

## 🧩 The Imports

```solidity
   
import "./IDepositBox.sol";
import "./BasicDepositBox.sol";
import "./PremiumDepositBox.sol";
import "./TimeLockedDepositBox.sol";

```

- `IDepositBox`: The interface we use to interact with all types of boxes in a unified way.
- The other three are concrete implementations (actual deployable contracts).

**Why use the interface?**

Because it lets us treat every box the same way, no matter its specific type. All deposit boxes follow the same rules (`getOwner()`, `storeSecret()`, etc.), so we don’t care *which kind* they are when we call those shared functions.

---

## 🧠 State Variables

```solidity
   
mapping(address => address[]) private userDepositBoxes;
mapping(address => string) private boxNames;

```

- `userDepositBoxes`: Maps a user’s address to all the deposit boxes they own (as contract addresses).
- `boxNames`: Lets users assign custom names to each of their boxes. Stored per box address.

---

## 🔔 Events

```solidity
   
event BoxCreated(address indexed owner, address indexed boxAddress, string boxType);
event BoxNamed(address indexed boxAddress, string name);

```

- `BoxCreated`: Emits every time a user creates a new box.
- `BoxNamed`: Fires when a user gives a nickname to their box.

These help the frontend and explorers display actions that happened.

---

## 🏗️ createBasicBox

```solidity
   
function createBasicBox() external returns (address) {
    BasicDepositBox box = new BasicDepositBox();
    userDepositBoxes[msg.sender].push(address(box));
    emit BoxCreated(msg.sender, address(box), "Basic");
    return address(box);
}

```

### Breakdown:

- `BasicDepositBox box = new BasicDepositBox();`: This line **deploys a new BasicDepositBox contract** and stores its address in the variable `box`.
- `userDepositBoxes[msg.sender].push(...)`: Adds the new box to the list of boxes owned by the sender.
- Emits an event so UIs can track this creation.
- Returns the new box’s address for easy access.

🧠 *This is how users "mint" a new digital vault box for themselves.*

---

## 

## 🏗️ `createPremiumBox()` — For Users Who Want to Store Extra Metadata

```solidity
    
function createPremiumBox() external returns (address) {
    PremiumDepositBox box = new PremiumDepositBox();
    userDepositBoxes[msg.sender].push(address(box));
    emit BoxCreated(msg.sender, address(box), "Premium");
    return address(box);
}

```

### What does this function do?

This function lets a user create a new **PremiumDepositBox** — a special kind of deposit box that supports extra data through the `setMetadata()` function.

Let’s go through each line:

---

### 1. Deploy a New PremiumDepositBox

```solidity
    
PremiumDepositBox box = new PremiumDepositBox();

```

This line **creates a brand new contract** of type `PremiumDepositBox`. Behind the scenes, this:

- Inherits from `BaseDepositBox`, so it gets ownership, secret storage, deposit time, etc.
- Adds a new state variable called `metadata` and functions to update/retrieve it.

When this line runs, the contract is **deployed on-chain**, and the user who called `createPremiumBox()` becomes the owner of the box (thanks to the `BaseDepositBox` constructor).

---

### 2. Save the Box in the User's Vault List

```solidity
    
userDepositBoxes[msg.sender].push(address(box));

```

Here, we:

- Convert the contract to its address using `address(box)`
- Add that address to the `userDepositBoxes` mapping for the user who created it

This ensures that when the user wants to retrieve all their boxes later, this one will show up.

---

### 3. Emit an Event for Frontend and Tracking

```solidity
    
emit BoxCreated(msg.sender, address(box), "Premium");

```

This fires a `BoxCreated` event that logs:

- The user who created the box
- The address of the new contract
- The string `"Premium"` so UIs know what kind of box it is

This is helpful for wallets, dashboards, and explorers to display activity without needing to index the chain manually.

---

### 4. Return the Box Address

```solidity
    
return address(box);

```

Lastly, we return the newly created box’s address so the frontend (or calling code) can interact with it immediately.

---

## 

## ⏳ `createTimeLockedBox()` — For Secrets You Don’t Want Opened Just Yet

```solidity
    
function createTimeLockedBox(uint256 lockDuration) external returns (address) {
    TimeLockedDepositBox box = new TimeLockedDepositBox(lockDuration);
    userDepositBoxes[msg.sender].push(address(box));
    emit BoxCreated(msg.sender, address(box), "TimeLocked");
    return address(box);
}

```

This function allows a user to create a **Time-Locked Deposit Box** — a smart contract that behaves just like a basic box, **but with one crucial twist**: the secret stored inside can’t be accessed until a certain amount of time has passed.

---

### Let’s go step-by-step:

---

### 1. Accept a Lock Duration

```solidity
    
function createTimeLockedBox(uint256 lockDuration)

```

This function takes in a number: `lockDuration` — which represents how many **seconds** the box should be locked for.

- For example, if `lockDuration = 3600`, the box will be locked for **1 hour**.
- During this period, the owner can **store a secret**, but they **won’t be able to view it** until the lock expires.

This is great for things like:

- Time capsules
- Delayed reveal messages
- Future gifts or commitments

---

### 2. Deploy the TimeLockedDepositBox

```solidity
    
TimeLockedDepositBox box = new TimeLockedDepositBox(lockDuration);

```

This line creates a new instance of `TimeLockedDepositBox`, passing in the `lockDuration`.

Inside the `TimeLockedDepositBox` constructor, this lock duration is added to the current block timestamp like this:

```solidity
    
unlockTime = block.timestamp + lockDuration;

```

So the box tracks **when it's safe to unlock** based on blockchain time.

The deployer (`msg.sender`) automatically becomes the owner — thanks to inheritance from `BaseDepositBox`.

---

### 3. Save the Box Under the User’s Account

```solidity
    
userDepositBoxes[msg.sender].push(address(box));

```

This adds the new box's address to the caller's list of deposit boxes, stored in the `userDepositBoxes` mapping.

- This allows each user to maintain a personal "vault" of deposit boxes.
- The contract keeps track of who owns which boxes without storing full user data on-chain.

---

### 4. Emit the `BoxCreated` Event

```solidity
    
emit BoxCreated(msg.sender, address(box), "TimeLocked");

```

Events are a key part of how dApps track on-chain actions without constantly querying smart contract state.

This logs the new box creation with:

- Who created it
- Its contract address
- The box type: `"TimeLocked"`

Frontends can use this to show “New Time-Locked Box Created” in the UI or show it in a user activity feed.

---

### 5. Return the Address

```solidity
    
return address(box);

```

Finally, the function returns the address of the newly deployed contract, so the caller (or frontend) can start interacting with it immediately.

---

## 🏷️ nameBox

```solidity
   
function nameBox(address boxAddress, string calldata name) external {
    IDepositBox box = IDepositBox(boxAddress);
    require(box.getOwner() == msg.sender, "Not the box owner");

    boxNames[boxAddress] = name;
    emit BoxNamed(boxAddress, name);
}

```

### What’s happening here?

- First, we **cast the generic address into the interface**:
    
    ```solidity
       
    IDepositBox box = IDepositBox(boxAddress);
    
    ```
    
    This lets us call `getOwner()` on the box without knowing what type it is.
    
- Then we **check ownership**:
    
    ```solidity
       
    require(box.getOwner() == msg.sender, "Not the box owner");
    
    ```
    
    This ensures only the rightful owner can rename the box.
    
- Finally, we save the new name and emit an event.

---

## 📝 storeSecret

```solidity
   
function storeSecret(address boxAddress, string calldata secret) external {
    IDepositBox box = IDepositBox(boxAddress);
    require(box.getOwner() == msg.sender, "Not the box owner");

    box.storeSecret(secret);
}

```

- Same ownership pattern.
- Calls the `storeSecret()` function from the interface, which is implemented in each box type.
- No event is emitted here because the box itself will emit one (`SecretStored`).

---

## 

## 🔄 `transferBoxOwnership()` — Handing Over a Vault

```solidity
    
function transferBoxOwnership(address boxAddress, address newOwner) external {
    IDepositBox box = IDepositBox(boxAddress);
    require(box.getOwner() == msg.sender, "Not the box owner");

    box.transferOwnership(newOwner);

    // Remove box from old owner
    address[] storage boxes = userDepositBoxes[msg.sender];
    for (uint i = 0; i < boxes.length; i++) {
        if (boxes[i] == boxAddress) {
            boxes[i] = boxes[boxes.length - 1];
            boxes.pop();
            break;
        }
    }

    // Add box to new owner
    userDepositBoxes[newOwner].push(boxAddress);
}

```

This function allows the current owner of a deposit box to **transfer ownership to someone else** — just like handing someone the keys to your safety deposit box.

It also makes sure that the `VaultManager` updates its own records to reflect this change.

Let’s go step by step:

---

### 1. Interface Casting & Ownership Check

```solidity
    
IDepositBox box = IDepositBox(boxAddress);
require(box.getOwner() == msg.sender, "Not the box owner");

```

- First, we take the `boxAddress` (which is just a regular address on the blockchain) and **cast it** to the `IDepositBox` interface.
    
    That’s what this does:
    
    ```solidity
        
    IDepositBox box = IDepositBox(boxAddress);
    
    ```
    
    Now, we can call functions like `getOwner()` or `transferOwnership()` on it — because we’ve told Solidity, *"Hey, trust me, this contract implements the `IDepositBox` interface."*
    
- Next, we verify that the person calling this function is actually the **current owner** of the box:
    
    ```solidity
        
    require(box.getOwner() == msg.sender, "Not the box owner");
    
    ```
    

If they’re not the owner, the function reverts. No unauthorized handoffs.

---

### 2. Call the Box's `transferOwnership()` Method

```solidity
    
box.transferOwnership(newOwner);

```

This tells the box contract itself to update its internal `owner` state.

This is important:

The actual box owns the data and logic — **VaultManager doesn't control it** — so this step ensures the box updates its own permissions.

---

### 3. Update the VaultManager’s Mapping

After changing ownership **inside the box contract**, we now have to make sure the `VaultManager`'s `userDepositBoxes` mapping reflects that too.

If we don’t update this list, our system will show the wrong owner in the frontend or APIs.

---

### 3.1 Remove the Box from the Sender’s List

```solidity
    
address[] storage boxes = userDepositBoxes[msg.sender];
for (uint i = 0; i < boxes.length; i++) {
    if (boxes[i] == boxAddress) {
        boxes[i] = boxes[boxes.length - 1];
        boxes.pop();
        break;
    }
}

```

- We grab the sender’s list of boxes.
- We loop through to find the one that’s being transferred.
- Once we find it:
    - We **swap it with the last item** in the array.
    - Then call `.pop()` to remove the last item.
- This is a classic Solidity trick to remove an item from an array without leaving gaps (because Solidity arrays don’t support `.remove(index)`).

---

### 3.2 Add the Box to the New Owner’s List

```solidity
    
userDepositBoxes[newOwner].push(boxAddress);

```

We then push the box onto the new owner's personal array of boxes — effectively moving ownership within our registry.

---

## 

## 📬 `getUserBoxes()` — View All Boxes Owned by a User

```solidity
    
function getUserBoxes(address user) external view returns (address[] memory) {
    return userDepositBoxes[user];
}

```

### What it does:

This function simply returns the **list of deposit box addresses** that belong to a specific user.

### Why it matters:

- Every time someone creates a box (via `createBasicBox()`, `createPremiumBox()`, or `createTimeLockedBox()`), the address of that newly created contract is stored in the `userDepositBoxes` mapping.
- This function allows you to retrieve that list for **any address**.

### How it works:

- `userDepositBoxes[user]` gives us the dynamic array of addresses stored for that particular user.
- Since it’s marked `view`, it’s **read-only** — calling this function **doesn’t cost any gas**.
- It returns a full array that can be used on the frontend to list out all of a user’s vaults.

> Use Case: Perfect for creating dashboards or profile pages that list a user's deposit boxes.
> 

---

## 🔎 `getBoxName()` — Read the Custom Name of a Box

```solidity
    
function getBoxName(address boxAddress) external view returns (string memory) {
    return boxNames[boxAddress];
}

```

### What it does:

Returns the **custom name** that the owner has assigned to a particular deposit box.

### Where the name comes from:

- Earlier, the user can call `nameBox(address, string)` to give their box a custom label.
- That label is stored in the `boxNames` mapping.

### Logic:

- It takes in the `boxAddress` as input.
- It looks up that address in the `boxNames` mapping and returns whatever string was set.
- If no name was given, Solidity just returns the **default empty string `""`**.

> Use Case: Great for improving the UI — instead of showing raw addresses, you can show meaningful names like “Main Vault” or “Dad’s Locker”.
> 

---

## 🧾 `getBoxInfo()` — Full Info in One Call

```solidity
    
function getBoxInfo(address boxAddress) external view returns (
    string memory boxType,
    address owner,
    uint256 depositTime,
    string memory name
) {
    IDepositBox box = IDepositBox(boxAddress);
    return (
        box.getBoxType(),
        box.getOwner(),
        box.getDepositTime(),
        boxNames[boxAddress]
    );
}

```

This is your **all-in-one helper function** for getting every important piece of metadata about a box.

Let’s break it down:

---

### 1. Interface Cast

```solidity
    
IDepositBox box = IDepositBox(boxAddress);

```

- We take the raw box address and **cast it to the IDepositBox interface**.
- This lets us safely call functions like `getBoxType()`, `getOwner()`, and `getDepositTime()` without knowing exactly which type of box it is (Basic, Premium, TimeLocked, etc.).
- As long as the contract implements the interface, these calls will work.

---

### 2. Return All the Key Details

```solidity
    
return (
    box.getBoxType(),
    box.getOwner(),
    box.getDepositTime(),
    boxNames[boxAddress]
);

```

Here’s what each part gives us:

- `box.getBoxType()` — Calls the child contract’s implementation and returns a string like `"Basic"`, `"Premium"`, or `"TimeLocked"`.
- `box.getOwner()` — Returns the current owner of the box.
- `box.getDepositTime()` — Returns the block timestamp when the box was deployed.
- `boxNames[boxAddress]` — Pulls the custom name (if any) from the `VaultManager`'s internal `boxNames` mapping.

---

### Why this function is useful:

Instead of making **four separate calls**, you get everything in **one**:

- ✅ Box type
- ✅ Owner
- ✅ Creation time
- ✅ Custom name

> Use Case: Build a table or card UI that shows a full summary of each user’s boxes with a single call — very useful for efficient frontend rendering.
> 

## ✅ Wrap-Up — A Modular Vault System Done Right

And that’s a wrap on our **Vault System**.

We started with a simple idea — letting users store secrets securely on-chain — and ended up building a **modular**, **extensible**, and **well-structured** smart contract system using Solidity’s most powerful concepts:

- ✅ **Interfaces** to define a common standard for all vaults
- ✅ **Abstract contracts** to provide reusable logic and enforce consistency
- ✅ **Inheritance** to build multiple vault types (Basic, Premium, TimeLocked) on top of shared foundations
- ✅ A **manager contract** to control the creation, naming, and ownership of vaults

What you just built mirrors the architecture behind real-world smart contract systems. Platforms like Compound, Aave, and OpenZeppelin use this same layered approach — with interfaces for standardization, abstract bases for reuse, and clean modular contracts on top.

So now, not only do you understand **how inheritance works**, but you’ve also seen how to structure larger systems that scale as your product grows.