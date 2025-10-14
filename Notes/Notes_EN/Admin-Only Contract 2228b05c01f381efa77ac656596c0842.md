# Admin-Only Contract

In our previous contracts, everyone was treated equally. If someone followed the rules (like placing a higher bid), they could interact with the contract freely. But in the real world, **some actions should be limited to a specific person or role**.

Think about a game where only the admin can distribute rewards. Or a treasury where only one person should approve withdrawals. We need a way to say:

> "Hey, only this person should be able to do this."
> 

That’s what this contract teaches you — how to create functions that **only the contract owner** can call, using something called a **modifier**. And along the way, we’ll simulate a treasure chest system where one person controls who can take treasure, how much, and when.

<aside>
💻

**Here’s the complete code** 👇🏼

[https://github.com/snehasharma76/30daysSolidity_Web3Compass/blob/master/AdminOnly.sol](https://github.com/snehasharma76/30daysSolidity_Web3Compass/blob/master/AdminOnly.sol)

</aside>

Let’s walk through the contract line by line — explaining what’s happening and why it matters.

---

## Setting Up the Owner

```solidity
 
address public owner;

constructor() {
    owner = msg.sender;
}

```

We start by declaring an `owner` variable. This will store the address of the person who deployed the contract.

In the constructor, we set `owner = msg.sender`. Remember from earlier lessons: `msg.sender` is a global variable that tells us **who is calling the function**. And since the constructor runs only once (at deployment), we’re saying:

> "Whoever deployed this contract becomes the owner."
> 

This owner will be the one with special powers in the rest of the contract.

---

## Reusable Access Control with a Modifier

```solidity
 
modifier onlyOwner() {
    require(msg.sender == owner, "Access denied: Only the owner can perform this action");
    _;
}

```

This is our **guardrail**.

Modifiers in Solidity let you create little reusable permission checks that you can attach to functions. This one checks if the caller is the owner. If they’re not, the function won’t run.

The `_` is where the rest of the function will be inserted if the check passes.

Now instead of repeating `require(msg.sender == owner)` everywhere, we can just write `onlyOwner` — it’s cleaner, safer, and easier to manage.

---

## Adding Treasure to the Chest

```solidity
 
uint256 public treasureAmount;

function addTreasure(uint256 amount) public onlyOwner {
    treasureAmount += amount;
}

```

Here we’re introducing the actual treasure — stored as a `uint256`.

Only the owner can call `addTreasure()`. When they do, we increase the treasure count by the amount they specify.

The `onlyOwner` modifier makes sure no random person can sneak in and add (or pretend to add) treasure.

---

## Approving Others to Withdraw

```solidity
 
mapping(address => uint256) public withdrawalAllowance;

function approveWithdrawal(address recipient, uint256 amount) public onlyOwner {
    require(amount <= treasureAmount, "Not enough treasure available");
    withdrawalAllowance[recipient] = amount;
}

```

Now let’s say the owner wants to allow someone to take out some treasure. We use a **mapping** to track how much each address is allowed to withdraw.

Before approving, we check that the treasure chest actually has enough to cover that amount.

So here’s what’s happening:

- The owner sets an allowance for a specific address.
- That address can later try to withdraw — but only if they’ve been approved.

This is kind of like a bank manager giving you a withdrawal slip with a limit written on it.

---

## Actually Withdrawing Treasure

```solidity
 
mapping(address => bool) public hasWithdrawn;

function withdrawTreasure(uint256 amount) public {

```

Here’s where the action happens — the function that lets people take treasure out of the chest.

Let’s break down what happens inside it.

---

### Case 1: The Owner Is Withdrawing

```solidity
 
if (msg.sender == owner) {
    require(amount <= treasureAmount, "Not enough treasury available for this action.");
    treasureAmount -= amount;
    return;
}

```

If the owner is calling this function, we give them full flexibility — they can withdraw any amount, as long as it doesn’t exceed what’s in the chest.

Notice how we don’t use their allowance or track their withdrawals — because they’re the one in charge.

---

### Case 2: A Regular User Is Withdrawing

```solidity
 
uint256 allowance = withdrawalAllowance[msg.sender];
require(allowance > 0, "You don't have any treasure allowance");
require(!hasWithdrawn[msg.sender], "You have already withdrawn your treasure");
require(allowance <= treasureAmount, "Not enough treasure in the chest");

```

If the caller isn’t the owner, we start running checks.

1. Have they been approved for anything?
2. Have they already withdrawn before?
3. Is there still enough treasure in the chest?

If any of these fail, the function stops immediately.

So now we’ve built a system where **users can only withdraw once, and only if the owner has approved them**.

---

### Finishing the Withdrawal

```solidity
 
hasWithdrawn[msg.sender] = true;
treasureAmount -= allowance;
withdrawalAllowance[msg.sender] = 0;
}

```

Once all checks pass, we:

- Mark them as "withdrawn"
- Subtract their approved amount from the chest
- Reset their allowance to zero so they can't try again

This ensures users get only what they were given — and only once.

---

## Resetting a User’s Withdrawal Status

```solidity
 
function resetWithdrawalStatus(address user) public onlyOwner {
    hasWithdrawn[user] = false;
}

```

Let’s say you want to let someone withdraw again — maybe they completed another quest or passed another level.

This function gives the owner the power to reset a user’s withdrawal status, allowing them to go through the process again.

Again, it's protected by `onlyOwner`.

---

## Transferring Ownership

```solidity
 
function transferOwnership(address newOwner) public onlyOwner {
    require(newOwner != address(0), "Invalid address");
    owner = newOwner;
}

```

Just like in real-world systems, sometimes the original creator steps down and hands over control.

This function lets the current owner assign a new owner. And we include a quick check to make sure the new address isn’t empty or invalid.

Once this runs, the new owner will have full control — including the ability to add treasure, approve users, and transfer ownership again.

---

## Viewing the Treasure (Owner Only)

```solidity
 
function getTreasureDetails() public view onlyOwner returns (uint256) {
    return treasureAmount;
}

```

Lastly, this function lets the owner check how much treasure is in the chest. It’s marked `view` since it doesn’t change any data, and `onlyOwner` so only the admin can use it.

---

## Final Thoughts

This contract may look small, but it teaches you a **huge concept in Solidity**: how to control who has permission to do what.

Here’s what we covered:

- How to use `msg.sender` to track and verify the caller
- How modifiers help avoid repetition and enforce clean access control
- How to use mappings and flags to track user-specific permissions
- How to simulate a real-world admin panel — only the owner can add, approve, reset, or transfer control

This is a common pattern you’ll see in token contracts, NFT minting, governance systems, and just about every production-level smart contract.

---

### What’s Next?

Here are a few ideas you can try to level up this contract:

- Add a **cooldown timer**: users can only withdraw once every X minutes
- Emit **events** when treasure is added, withdrawn, or ownership changes
- Add a **view function** that tells users if they’re approved and whether they’ve already withdrawn
- Add a **maximum withdrawal limit per user**

Let me know when you're ready for the next one — we’re just getting warmed up.