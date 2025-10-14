# AuctionHouse Contract

Alright. You’ve worked with data types like `string`, `uint`, `mapping`, and `array`. You’ve seen how to store and retrieve data, and even how to enforce some logic with `require()`.

But let’s be honest — working with just isolated pieces is fine for practice...

Now it’s time to bring it all together and **build a complete, real-world contract**.

In this tutorial, we’re going to build an **auction system**. Think of it like an online bidding platform where users can place bids for an item, and the highest bidder at the end wins.

<aside>
💻

Here is the complete code 👇🏼

[https://github.com/snehasharma76/30daysSolidity_Web3Compass/blob/master/AuctionHouse.sol](https://github.com/snehasharma76/30daysSolidity_Web3Compass/blob/master/AuctionHouse.sol)

</aside>

But before jumping into the code, let’s ask ourselves:

---

## What data do we actually need to run an auction?

Let’s think it through.

### 1. Who created the auction?

We need to track the person who’s running the show — the one who deployed the contract. Let’s call them the **owner**.

```solidity
 
address public owner;

```

We’ll set this when the contract is deployed, and make it `public` so that anyone can verify who owns the auction.

---

### 2. What are we auctioning?

There’s got to be an item, right? A painting, a phone, a rare digital collectible — we want to let the auction creator describe what’s up for grabs.

```solidity
 
string public item;

```

This can be anything — we’ll let the owner provide the item name when deploying the contract.

---

### 3. When does the auction end?

We don’t want the auction to go on forever. So we’ll define how long it should run — and track when it ends using a timestamp.

```solidity
 
uint public auctionEndTime;

```

We’ll calculate this using the current time + how long the auction should run. More on that when we get to the constructor.

---

### 4. Who’s currently winning?

That’s kind of important. We need to store:

- The highest bid so far
- And the address of the person who placed that bid

```solidity
 
address private highestBidder;
uint private highestBid;

```

We mark these as `private` so no one can directly access them and cheat the system. But we’ll still let people see them — just **after** the auction ends.

---

### 5. Has the auction ended yet?

We need a flag to mark if the auction is done — to make sure it doesn’t get ended twice, or someone tries to view the winner too early.

```solidity
 
bool public ended;

```

This starts as `false`, and gets flipped to `true` once someone officially ends the auction.

---

### 6. Who placed bids, and how much?

We want to keep track of each user’s bid. That way, we can make sure people don’t bid the same amount again, and we know who participated.

```solidity
 
mapping(address => uint) public bids;

```

And maybe we also want a full list of everyone who placed at least one bid — not just their amounts:

```solidity
 
address[] public bidders;

```

This is helpful if you want to show a leaderboard or just keep things transparent.

---

## Okay, so how do we set this all up?

Let’s start with the constructor — the part that runs only once when the contract is deployed.

```solidity
 
constructor(string memory _item, uint _biddingTime) {
    owner = msg.sender;
    item = _item;
    auctionEndTime = block.timestamp + _biddingTime;
}

```

Here’s what’s happening:

- `msg.sender` is a global variable — it gives us the address of **whoever is deploying the contract**. We save that as the owner.
- `_item` is the name of the thing being auctioned — we store it in the `item` variable.
- `_biddingTime` is how long the auction should last (in seconds). We add it to the current time (`block.timestamp`) to figure out when the auction should end.

For example, if someone deploys this with `"Digital Art"` and `300`, the auction ends **5 minutes later**.

---

## Now let’s talk about placing a bid.

This is the main thing users will do. Here’s what the function looks like:

```solidity
 
function bid(uint amount) external {
    require(block.timestamp < auctionEndTime, "Auction has already ended.");
    require(amount > 0, "Bid amount must be greater than zero.");
    require(amount > bids[msg.sender], "New bid must be higher than your current bid.");

    if (bids[msg.sender] == 0) {
        bidders.push(msg.sender);
    }

    bids[msg.sender] = amount;

    if (amount > highestBid) {
        highestBid = amount;
        highestBidder = msg.sender;
    }
}

```

Let’s go through it slowly.

---

### First: Can this bid even happen?

We use `require()` to set rules — if the rule fails, the function stops immediately.

1. Has the auction already ended?

```solidity
 
require(block.timestamp < auctionEndTime, "Auction has already ended.");

```

If the current time is past the auction deadline, no more bids are accepted.

1. Is the bid a valid amount?

```solidity
 
require(amount > 0, "Bid amount must be greater than zero.");

```

We don’t want people placing zero or negative bids.

1. Is this bid higher than the person’s last bid?

```solidity
 
require(amount > bids[msg.sender], "New bid must be higher than your current bid.");

```

This ensures people are only increasing their bids — not spamming or lowering them.

---

### Next: Is this a new bidder?

```solidity
 
if (bids[msg.sender] == 0) {
    bidders.push(msg.sender);
}

```

If this is the user’s **first ever bid**, we add them to the `bidders` array. That way, we have a full list of who participated.

---

### Then: Save the bid

```solidity
 
bids[msg.sender] = amount;

```

We update their bid amount in the `bids` mapping.

---

### And finally: Are they the new leader?

```solidity
 
if (amount > highestBid) {
    highestBid = amount;
    highestBidder = msg.sender;
}

```

If this is the biggest bid we’ve seen so far, we update both the bid and the bidder.

---

## Ending the Auction

At some point, the auction has to stop. That’s where this function comes in:

```solidity
 
function endAuction() external {
    require(block.timestamp >= auctionEndTime, "Auction hasn't ended yet.");
    require(!ended, "Auction end already called.");
    ended = true;
}

```

Let’s walk through it.

1. First, we check if the auction time has passed:

```solidity
 
require(block.timestamp >= auctionEndTime, "Auction hasn't ended yet.");

```

1. Then we make sure no one’s already ended it:

```solidity
 
require(!ended, "Auction end already called.");

```

1. If all is good, we flip the `ended` flag:

```solidity
 
ended = true;

```

This is a **common Solidity pattern** — using a `bool` to make sure something only happens once.

---

## Viewing Info After the Auction

We don’t want to expose the winner until the auction is over. That’s why we made `highestBidder` and `highestBid` private. But we can reveal them like this:

```solidity
 
function getWinner() external view returns (address, uint) {
    require(ended, "Auction has not ended yet.");
    return (highestBidder, highestBid);
}

```

We check that the auction has ended. If it has, we return the winner’s address and their bid.

---

## Want to see all the bidders?

Sure, we’ve got that too:

```solidity
 
function getAllBidders() external view returns (address[] memory) {
    return bidders;
}

```

This just returns the `bidders` array, so anyone can see who took part.

---

## Quick Recap

This contract introduced a bunch of important concepts:

- **Global variables**:
    
    `msg.sender` tells us who’s calling a function
    
    `block.timestamp` gives us the current time
    
- **require()** helps us enforce rules and control what’s allowed
- **Constructors** run once and set up the initial state of the contract
- **Visibility** keywords like `public`, `private`, and `external` control what can be accessed and by whom
- **Mappings** and **arrays** help us track user data and participation

---

## What can you build on top of this?

This auction works — but you can always add more.

Here are a few things you can try:

- Add a **minimum increment rule** (like each new bid must be at least 5% higher)
- Allow non-winners to **withdraw their bids**
- Add a **starting price**