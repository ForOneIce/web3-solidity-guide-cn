# Simple IOU Contract

Let’s say you and your friends go out often — splitting bills, paying for travel, buying group gifts. Most of the time, one person pays upfront and the rest say “I’ll pay you back later.”

But you’ve probably been in this situation:

> “Wait… how much do I owe you again?”
> 
> 
> “Did I already pay you back?”
> 
> “What’s the total amount you covered for the group?”
> 

It gets messy fast.

That’s where this smart contract comes in. `SimpleIOU` is your **on-chain group ledger**. It helps friends:

- Track debts,
- Store ETH in their own in-app balance,
- And settle up easily, without doing math or spreadsheets.

<aside>
💻

**Here is the complete code :**

[https://github.com/snehasharma76/30daysSolidity_Web3Compass/blob/master/SimpleIOUApp.sol](https://github.com/snehasharma76/30daysSolidity_Web3Compass/blob/master/SimpleIOUApp.sol)

</aside>

Let’s go line-by-line and see how it works.

---

## State Variables: What are we keeping track of?

Before any logic can run, the contract needs to **remember a few things**.

### `owner`

```solidity
 
address public owner;

```

We start by defining who the **admin** of the group is.

The `owner` is the person who deployed the contract. This person will have special permissions, like adding new friends to the group. It’s like the first person who creates the Splitwise group — they manage the access.

---

### `registeredFriends` and `friendList`

```solidity
 
mapping(address => bool) public registeredFriends;
address[] public friendList;

```

We want this contract to be **private to your group**. Not just anyone on the internet should be able to join and start logging IOUs.

Here’s how we manage that:

- `registeredFriends` lets us **quickly check** if someone is allowed to use the contract.
- `friendList` gives us a **complete list** of all registered addresses, which is useful if you want to display all group members in a frontend.

When a friend is added, their address goes into both.

---

### `balances`

```solidity
 
mapping(address => uint256) public balances;

```

Now let’s handle the money.

Each person can deposit ETH into their personal balance **within the contract**. That ETH can later be used to:

- Pay off a debt,
- Transfer ETH to another member,
- Or withdraw it completely.

The actual ETH is stored *inside* the contract, and this mapping just keeps track of **who owns how much**.

---

### `debts`

```solidity
 
mapping(address => mapping(address => uint256)) public debts;

```

Here’s where the real IOU magic happens.

This is a **nested mapping**, meaning:

```solidity
 
debts[debtor][creditor] = amount;

```

For example:

```solidity
 
debts[0xAsha][0xRavi] = 1.5 ether;

```

Means Asha owes Ravi 1.5 ETH. And it’s right there, stored transparently on-chain. No miscommunication, no forgetting.

---

## The Constructor

```solidity
 
constructor() {
    owner = msg.sender;
    registeredFriends[msg.sender] = true;
    friendList.push(msg.sender);
}

```

When the contract is deployed:

- The `msg.sender` (whoever deployed it) becomes the owner.
- They’re automatically registered as a friend — since they obviously want to use the system.
- Their address gets added to the `friendList`.

So from the moment this contract is created, we already have one active member: the admin.

---

## Modifiers: Controlling Access

We don’t want just anyone to interact with this contract. These **modifiers** help protect functions from unauthorized access.

---

### `onlyOwner`

```solidity
 
modifier onlyOwner() {
    require(msg.sender == owner, "Only owner can perform this action");
    _;
}

```

This one’s simple: it ensures that **only the person who deployed the contract** (the owner) can perform certain actions — like adding a friend.

---

### `onlyRegistered`

```solidity
 
modifier onlyRegistered() {
    require(registeredFriends[msg.sender], "You are not registered");
    _;
}

```

Only registered friends — people the owner added — can:

- Deposit ETH,
- Record debts,
- Pay debts,
- Send ETH,
- Or withdraw it.

If someone tries to use the contract without being added first, the function will immediately reject their request.

---

## Functions: What Can Friends Do?

Now that we’ve set up the structure, let’s look at the actual functionality — what users can do and how it works behind the scenes.

---

### `addFriend`

```solidity
 
function addFriend(address _friend) public onlyOwner {
    require(_friend != address(0), "Invalid address");
    require(!registeredFriends[_friend], "Friend already registered");

    registeredFriends[_friend] = true;
    friendList.push(_friend);
}

```

Only the owner can call this function. It:

- Checks that the new address is not `0x0`
- Makes sure the person hasn’t already been added
- Then adds them to both `registeredFriends` and `friendList`

This keeps your IOU group small, private, and safe.

---

### `depositIntoWallet`

```solidity
 
function depositIntoWallet() public payable onlyRegistered {
    require(msg.value > 0, "Must send ETH");
    balances[msg.sender] += msg.value;
}

```

Here’s how a user adds ETH to their in-app balance:

- The function is `payable`, meaning it can receive ETH.
- `msg.value` holds the amount of ETH sent.
- That amount gets added to their internal balance.

Now they can use this ETH to pay off debts, send money, or withdraw it later.

---

### `recordDebt`

```solidity
 
function recordDebt(address _debtor, uint256 _amount) public onlyRegistered {
    require(_debtor != address(0), "Invalid address");
    require(registeredFriends[_debtor], "Address not registered");
    require(_amount > 0, "Amount must be greater than 0");

    debts[_debtor][msg.sender] += _amount;
}

```

If your friend owes you money, you call this to record it.

Let’s say Ravi paid for everyone’s lunch, and Asha owes him 0.05 ETH. Ravi would call:

```solidity
 
recordDebt(ashaAddress, 0.05 ether);

```

The contract stores that Asha now owes Ravi that amount.

No ETH is moved here — we’re just **recording the agreement**.

---

### `payFromWallet`

```solidity
 
function payFromWallet(address _creditor, uint256 _amount) public onlyRegistered {
    require(_creditor != address(0), "Invalid address");
    require(registeredFriends[_creditor], "Creditor not registered");
    require(_amount > 0, "Amount must be greater than 0");
    require(debts[msg.sender][_creditor] >= _amount, "Debt amount incorrect");
    require(balances[msg.sender] >= _amount, "Insufficient balance");

    balances[msg.sender] -= _amount;
    balances[_creditor] += _amount;
    debts[msg.sender][_creditor] -= _amount;
}

```

This is where you **pay someone back** using your ETH balance stored in the contract.

Here’s what it does:

- Checks you owe that person
- Ensures you have enough ETH
- Subtracts the amount from your balance
- Adds it to the creditor’s balance
- Reduces your recorded debt

Still, no ETH actually leaves the contract. It just moves internally between users’ balances.

---

### `transferEther` — Sending ETH Using `transfer()`

```solidity

function transferEther(address payable _to, uint256 _amount) public onlyRegistered {
    require(_to != address(0), "Invalid address");
    require(registeredFriends[_to], "Recipient not registered");
    require(balances[msg.sender] >= _amount, "Insufficient balance");

    balances[msg.sender] -= _amount;
    _to.transfer(_amount);
    balances[_to] += _amount;
}

```

### What's going on here?

Let’s say you’ve got 1 ETH saved up in your balance, and you want to send 0.25 ETH to your friend Rohit (who’s also in your friend group). This function allows you to send that ETH directly from the contract to their wallet.

Here's what it checks:

1. **Is the recipient’s address valid?**
2. **Are they registered in the group?**
3. **Do you have enough ETH in your balance?**

If all of that passes, your balance is reduced, and `_to.transfer(_amount)` sends ETH from the contract to the recipient's address.

### So… what is `transfer()`?

`transfer()` is a built-in Solidity method used to send ETH from a contract to an external address.

Syntax:

```solidity
recipientAddress.transfer(amount);

```

Here’s why `transfer()` was traditionally considered safe:

- It **automatically reverts** if the send fails
- It **forwards only 2300 gas** to the recipient — just enough to receive the ETH but not enough to execute any other code (which protects against reentrancy attacks)

### Why might that be a problem?

That 2300 gas limit is also its weakness. If the recipient is **a smart contract** (and not a wallet), it may need more gas to handle the incoming ETH — like updating a log, emitting an event, or storing state.

If the recipient contract needs more than 2300 gas — the transfer fails.

In modern Solidity, `transfer()` is fine for sending ETH to **wallets**, but risky or limiting when dealing with **contracts**.

---

### `transferEtherViaCall` — Sending ETH Using `call()`

```solidity

function transferEtherViaCall(address payable _to, uint256 _amount) public onlyRegistered {
    require(_to != address(0), "Invalid address");
    require(registeredFriends[_to], "Recipient not registered");
    require(balances[msg.sender] >= _amount, "Insufficient balance");

    balances[msg.sender] -= _amount;

    (bool success, ) = _to.call{value: _amount}("");
    balances[_to] += _amount;
    require(success, "Transfer failed");
}

```

This function does the same job — sending ETH to another friend — but it uses a more flexible method: `call`.

### So… what is `call()`?

`call()` is a **low-level** function in Solidity used for sending ETH and calling functions. It's written like this:

```solidity

(bool success, ) = recipient.call{value: amount}("");

```

It gives you a lot more control than `transfer()`:

- **No gas limit** — the recipient contract can execute whatever logic it wants
- You can **check the success** of the operation using the `success` variable

But with great power comes great responsibility — if you use `call()`, it’s up to you to:

- Check whether it succeeded
- Handle any failure cases properly
- Make sure your contract is protected from reentrancy attacks (more on that later in advanced contracts)

### Why do we use `call()` here?

Using `call()` makes the function **compatible with smart contract addresses** — not just externally owned accounts (wallets). So even if your friend’s wallet is actually a contract (like a Gnosis Safe or a payment splitter), the ETH will still go through.

---

### `withdraw`

```solidity
 
function withdraw(uint256 _amount) public onlyRegistered {
    require(balances[msg.sender] >= _amount, "Insufficient balance");

    balances[msg.sender] -= _amount;

    (bool success, ) = payable(msg.sender).call{value: _amount}("");
    require(success, "Withdrawal failed");
}

```

If you want to take your ETH out of the contract, this function does it.

- It subtracts the amount from your internal balance
- Sends it to your wallet using `call()`

Simple and safe.

---

### `checkBalance`

```solidity
 
function checkBalance() public view onlyRegistered returns (uint256) {
    return balances[msg.sender];
}

```

Just a quick way to see how much ETH you have in the system.

---

## Recap

You now have a fully working, clean, beginner-friendly debt tracker.

Users can:

- Join the group (if the owner adds them)
- Deposit ETH
- Record debts
- Settle them using their balance
- Send and withdraw ETH directly

You also learned:

- How to use mappings (single and nested)
- How ETH transfers work (`transfer` vs `call`)
- How to keep your contract access-controlled and secure

Let me know when you're ready to extend this with debt forgiveness, group splits, or event logging. You're well on your way.