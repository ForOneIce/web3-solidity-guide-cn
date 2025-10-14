# The Ether Piggy Bank

---

Let’s go back to where it all began.

You’ve already built an **auction system** where users could place bids.

You’ve created a **treasure chest** that only an admin could unlock.

You’ve learned how to track balances and restrict access using modifiers.

But let’s be real — so far, we’ve been playing with toy money. Just numbers stored on the blockchain.

What if we wanted to build something a little more *practical*? Something a little more *real*?

Let’s say you and a few close friends want to create a shared savings pool — a digital piggy bank. Not open to the public, just your own little club.

Everyone should be able to:

- Join the group (if approved)
- Deposit savings
- Check their balance
- And maybe even withdraw when they need it

And the best part? Eventually, this piggy bank will be able to accept **real ETH**. That’s right — actual Ether, stored securely on-chain.

<aside>
💻

Here is the complete code 👇🏼

[https://github.com/snehasharma76/30daysSolidity_Web3Compass/blob/master/EtherPiggyBank.sol](https://github.com/snehasharma76/30daysSolidity_Web3Compass/blob/master/EtherPiggyBank.sol)

</aside>

Let’s start building.

---

## What all data do we need for our piggy bank?

Before we start writing functions, let’s think about this like we’re designing a system from scratch. We gather around a table and ask:

> “What should this piggy bank remember?”
> 

---

### 1. Who’s in charge?

Every group needs a leader — someone responsible for adding new members, and keeping things in order.

So we introduce the **bank manager**:

```solidity
 
address public bankManager;

```

This is the person who deployed the contract. They’re the one with admin privileges — the only one who can approve new members into the club.

---

### 2. Who are the members?

We need a way to track who’s allowed to use the piggy bank.

```solidity
 
address[] members;
mapping(address => bool) public registeredMembers;

```

Here’s why we use both:

- `members`: an array that stores the full list of people who’ve joined
- `registeredMembers`: a mapping that lets us quickly check if someone’s approved

Think of `members` like your group chat list. And `registeredMembers` like the security gate that says yes or no when someone tries to interact with the contract.

---

### 3. How much has each person saved?

Of course, we need to know how much each member has deposited over time.

```solidity
 
mapping(address => uint256) balance;

```

This is the heart of the piggy bank. It stores the balance for every member.

---

## Setting things up: constructor

Now that we know what we need, let’s write the part that sets it all in motion.

```solidity
 
constructor() {
    bankManager = msg.sender;
    members.push(msg.sender);
}

```

Here’s what’s happening:

- When the contract is deployed, the deployer becomes the `bankManager`
- We also add them to the `members` list — so the manager is also the first saver

We’ve officially opened our piggy bank for business.

---

## Now let’s build the rules (Modifiers)

Before we start writing the actual features, we need to talk about **who can do what**.

This is where **modifiers** come in — little pieces of reusable logic that protect your functions.

---

 `onlyBankManager`

```solidity
 
modifier onlyBankManager() {
    require(msg.sender == bankManager, "Only bank manager can perform this action");
    _;
}

```

This modifier makes sure that only the manager can call certain functions — like adding members.

If someone else tries? The contract says: “Nope. Access denied.”

---

### `onlyRegisteredMember`

```solidity
 
modifier onlyRegisteredMember() {
    require(registeredMembers[msg.sender], "Member not registered");
    _;
}

```

This one ensures that only people who’ve been officially added to the club can deposit or withdraw savings.

---

Alright — now that we have our roles and memory in place, let’s move on to the actual features.

---

## Adding new members

Let’s say the manager wants to add one of their friends, Alex.

Here’s the function:

```solidity
 
function addMembers(address _member) public onlyBankManager {
    require(_member != address(0), "Invalid address");
    require(_member != msg.sender, "Bank Manager is already a member");
    require(!registeredMembers[_member], "Member already registered");

    registeredMembers[_member] = true;
    members.push(_member);
}

```

We check:

- That the address is valid
- That the manager isn’t trying to add themselves again
- That the person isn’t already in the club

Once everything looks good, we approve the new member.

---

### Viewing the members

Let’s say someone wants to see who all is in the group. Maybe to double-check that they’ve been added.

```solidity
 
function getMembers() public view returns (address[] memory) {
    return members;
}

```

This is a public `view` function — it simply returns the full list of members.

---

## Depositing (simulated savings)

Now we get to the good stuff.

Let’s say you want to simulate adding 100 units to your piggy bank.

Here’s the function:

```solidity
 
function deposit(uint256 _amount) public onlyRegisteredMember {
    require(_amount > 0, "Invalid amount");
    balance[msg.sender] += _amount;
}

```

This function:

- Makes sure the amount is greater than zero
- Adds that amount to your balance

At this point, we’re still not dealing with **real ETH**. These are just numbers — a placeholder for savings.

Think of this like a prototype. You’re testing the system before putting actual money in.

---

## Withdrawing savings (on paper)

You’ve saved up 100 units. Now you want to take out 30.

```solidity
 
function withdraw(uint256 _amount) public onlyRegisteredMember {
    require(_amount > 0, "Invalid amount");
    require(balance[msg.sender] >= _amount, "Insufficient balance");
    balance[msg.sender] -= _amount;
}

```

We check:

- That the amount is valid
- That you have enough saved
- Then we subtract from your balance

No Ether is being transferred. This is just an internal update — we’re still in simulation mode.

---

## But what if we want to deposit *real ETH*?

Until now, we’ve only been playing with numbers.

But now your friends say:

> “This is great. But what if we actually start saving real Ether? Like… actually send money into the piggy bank?”
> 

That’s where we introduce two new concepts:

- `payable`
- `msg.value`

---

## Depositing real Ether into the piggy bank

Let’s write a new version of the deposit function — one that accepts real ETH.

```solidity
 
function depositAmountEther() public payable onlyRegisteredMember {
    require(msg.value > 0, "Invalid amount");
    balance[msg.sender] += msg.value;
}

```

Here’s what’s happening:

- `payable` means this function is **allowed to receive Ether**. Without it, any ETH sent would be rejected.
- `msg.value` holds the **amount of ETH** (in wei) that the user sent in

So when a member calls this function and includes Ether in the transaction:

- That ETH gets stored inside the contract
- And we add it to their balance

This is real savings now — actual ETH, tracked per user, just like a bank.

---

## What have we built?

- A savings club with clear roles (bank manager and members)
- A registration system for new members
- A balance tracker for deposits and withdrawals
- And finally — the ability to accept real Ether into the piggy bank

From simple logic to actual ETH management, you’ve now crossed into the real world of smart contracts.

---

## Where can we go from here?

You could now:

- Add a withdrawal function that sends Ether back to users
- Add limits, cooldowns, or approval systems

This piggy bank may have started with just you and your friends…

But now? It’s a legit on-chain system — and you built it.

Let’s keep going.