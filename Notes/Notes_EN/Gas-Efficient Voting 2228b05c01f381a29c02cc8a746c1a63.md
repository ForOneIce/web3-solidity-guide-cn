# Gas-Efficient Voting

Alright — we’ve come a long way. From learning how to build token contracts to secure vault systems, you now have a solid grasp on smart contract architecture and inheritance.

But there’s one thing we haven’t touched deeply yet: **gas optimization**.

When writing smart contracts, especially ones that many users will interact with (like voting systems), it’s crucial to be **gas-efficient** — otherwise, even simple actions can become expensive and inaccessible for users.

So in this lesson, we’re going to explore how to write **a voting contract that’s lean, efficient, and cost-effective**.

But before we jump into the optimized version, let’s look at what a typical, unoptimized version might look like — and why it can be improved.

## 🧱 Part 1: The General Voting Contract (Unoptimized)

Before diving into the **GasEfficientVoting** contract, let’s imagine how this voting system would look in a more “standard” Solidity contract — something a beginner might write without worrying too much about gas.

Here’s a simplified version of that:

```solidity
 
pragma solidity ^0.8.0;

contract BasicVoting {
    struct Proposal {
        string name;
        uint256 voteCount;
        uint256 startTime;
        uint256 endTime;
        bool executed;
    }

    Proposal[] public proposals;
    mapping(address => mapping(uint => bool)) public hasVoted;

    function createProposal(string memory name, uint duration) public {
        proposals.push(Proposal({
            name: name,
            voteCount: 0,
            startTime: block.timestamp,
            endTime: block.timestamp + duration,
            executed: false
        }));
    }

    function vote(uint proposalId) public {
        Proposal storage proposal = proposals[proposalId];
        require(block.timestamp >= proposal.startTime, "Too early");
        require(block.timestamp <= proposal.endTime, "Too late");
        require(!hasVoted[msg.sender][proposalId], "Already voted");

        hasVoted[msg.sender][proposalId] = true;
        proposal.voteCount++;
    }

    function executeProposal(uint proposalId) public {
        Proposal storage proposal = proposals[proposalId];
        require(block.timestamp > proposal.endTime, "Too early");
        require(!proposal.executed, "Already executed");

        proposal.executed = true;
        // Some execution logic here
    }
}

```

### 🚨 Problems in This Version:

- Uses `string` instead of `bytes32` for names (more expensive)
- Uses dynamic arrays for proposals (more gas to grow and access)
- Tracks individual votes using a **nested mapping** (`mapping(address => mapping(uint => bool))`) — more storage
- Every storage write (like vote tracking) consumes gas

---

Now let’s walk through the **GasEfficientVoting** version and see how each part was optimized:

# What We’re Looking At

We’ve already written a standard voting contract before. It used strings for proposal names, `uint256` everywhere, arrays to store data, and multiple mappings to track votes.

Now, we’re rewriting that same logic — **but this time with gas optimization in mind.**

So the contract is still:

- Letting users create proposals
- Letting users vote
- Tracking who voted for what
- Letting us execute a proposal
- Emitting events

But it’s doing all of that using **less gas per operation.**

Let’s go through it, one section at a time:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract GasEfficientVoting {
    
    // Use uint8 for small numbers instead of uint256
    uint8 public proposalCount;
    
    // Compact struct using minimal space types
    struct Proposal {
        bytes32 name;          // Use bytes32 instead of string to save gas
        uint32 voteCount;      // Supports up to ~4.3 billion votes
        uint32 startTime;      // Unix timestamp (supports dates until year 2106)
        uint32 endTime;        // Unix timestamp
        bool executed;         // Execution status
    }
    
    // Using a mapping instead of an array for proposals is more gas efficient for access
    mapping(uint8 => Proposal) public proposals;
    
    // Single-slot packed user data
    // Each address occupies one storage slot in this mapping
    // We pack multiple voting flags into a single uint256 for gas efficiency
    // Each bit in the uint256 represents a vote for a specific proposal
    mapping(address => uint256) private voterRegistry;
    
    // Count total voters for each proposal (optional)
    mapping(uint8 => uint32) public proposalVoterCount;
    
    // Events
    event ProposalCreated(uint8 indexed proposalId, bytes32 name);
    event Voted(address indexed voter, uint8 indexed proposalId);
    event ProposalExecuted(uint8 indexed proposalId);
    

    
    // === Core Functions ===
    
    /**
     * @dev Create a new proposal
     * @param name The proposal name (pass as bytes32 for gas efficiency)
     * @param duration Voting duration in seconds
     */
    function createProposal(bytes32 name, uint32 duration) external {
        require(duration > 0, "Duration must be > 0");
        
        // Increment counter - cheaper than .push() on an array
        uint8 proposalId = proposalCount;
        proposalCount++;
        
        // Use a memory struct and then assign to storage
        Proposal memory newProposal = Proposal({
            name: name,
            voteCount: 0,
            startTime: uint32(block.timestamp),
            endTime: uint32(block.timestamp) + duration,
            executed: false
        });
        
        proposals[proposalId] = newProposal;
        
        emit ProposalCreated(proposalId, name);
    }
    
    /**
     * @dev Vote on a proposal
     * @param proposalId The proposal ID
     */
    function vote(uint8 proposalId) external {
        // Require valid proposal
        require(proposalId < proposalCount, "Invalid proposal");
        
        // Check proposal voting period
        uint32 currentTime = uint32(block.timestamp);
        require(currentTime >= proposals[proposalId].startTime, "Voting not started");
        require(currentTime <= proposals[proposalId].endTime, "Voting ended");
        
        // Check if already voted using bit manipulation (gas efficient)
        uint256 voterData = voterRegistry[msg.sender];
        uint256 mask = 1 << proposalId;
        require((voterData & mask) == 0, "Already voted");
        
        // Record vote using bitwise OR
        voterRegistry[msg.sender] = voterData | mask;
        
        // Update proposal vote count
        proposals[proposalId].voteCount++;
        proposalVoterCount[proposalId]++;
        
        emit Voted(msg.sender, proposalId);
    }
    
    /**
     * @dev Execute a proposal after voting ends
     * @param proposalId The proposal ID
     */
    function executeProposal(uint8 proposalId) external {
        require(proposalId < proposalCount, "Invalid proposal");
        require(block.timestamp > proposals[proposalId].endTime, "Voting not ended");
        require(!proposals[proposalId].executed, "Already executed");
        
        proposals[proposalId].executed = true;
        
        emit ProposalExecuted(proposalId);
        
        // In a real contract, execution logic would happen here
    }
    
    // === View Functions ===
    
    /**
     * @dev Check if an address has voted for a proposal
     * @param voter The voter address
     * @param proposalId The proposal ID
     * @return True if the address has voted
     */
    function hasVoted(address voter, uint8 proposalId) external view returns (bool) {
        return (voterRegistry[voter] & (1 << proposalId)) != 0;
    }
    
    /**
     * @dev Get detailed proposal information
     * Uses calldata for parameters and memory for return values
     */
    function getProposal(uint8 proposalId) external view returns (
        bytes32 name,
        uint32 voteCount,
        uint32 startTime,
        uint32 endTime,
        bool executed,
        bool active
    ) {
        require(proposalId < proposalCount, "Invalid proposal");
        
        Proposal storage proposal = proposals[proposalId];
        
        return (
            proposal.name,
            proposal.voteCount,
            proposal.startTime,
            proposal.endTime,
            proposal.executed,
            (block.timestamp >= proposal.startTime && block.timestamp <= proposal.endTime)
        );
    }
    
    /**
     * @dev Convert string to bytes32 (helper for frontend integration)
     * Note: This is a pure function that doesn't use state, so it's gas-efficient
     */

}
```

## 

# 🚀 What We’re Looking At

In our earlier voting contract, we went the classic route — human-readable strings for proposal names, `uint256` as the default type for everything, and arrays + mappings to keep track of votes. It worked, but it wasn’t lean.

This time, we're building the **same voting system**, but with a new mindset: **how can we save every possible unit of gas without compromising functionality?**

So what stays the same?

- ✅ Users can still create proposals
- ✅ Users can vote
- ✅ We track voting history
- ✅ Proposals can be executed
- ✅ Events are emitted for transparency

But here’s the catch: we do all of this **faster and cheaper** by making smarter storage and logic decisions.

Let’s unpack the contract piece by piece 👇

---

## 🧱 Contract Definition

```solidity

// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract GasEfficientVoting {

```

- The name says it all — this contract is a trimmed-down, gas-friendly version of our voting system.

---

## 🔐 Storage Variables

```solidity

uint8 public proposalCount;

```

- **Why `uint8`?** We probably won’t have more than 255 proposals. Using `uint256` would give us way more range than needed, but waste 31 extra bytes of storage.
- This little change reduces gas usage for both reading and writing this variable.

---

## 🗃️ The Proposal Struct

```solidity
 
struct Proposal {
    bytes32 name;
    uint32 voteCount;
    uint32 startTime;
    uint32 endTime;
    bool executed;
}

```

This is where optimization really starts showing:

| Field | Type | Why it’s chosen |
| --- | --- | --- |
| `name` | `bytes32` | Fixed-size, cheaper than `string` |
| `voteCount` | `uint32` | Enough for 4.2 billion votes, saves gas |
| `startTime` | `uint32` | Same here — fits UNIX timestamps |
| `endTime` | `uint32` | We don’t need nanosecond precision |
| `executed` | `bool` | A single byte for true/false |

> Struct Packing Trick: Solidity stores struct data in 32-byte chunks. By picking types carefully (uint32, bool, bytes32), we reduce wasted space — and fewer storage slots = lower gas.
> 

---

## 🧭 Proposal Mapping

```solidity
mapping(uint8 => Proposal) public proposals;

```

- Instead of storing proposals in an array, we use a mapping.
- Mappings give us **direct access (O(1))** to each proposal, without iteration or bounds checks like arrays require.

> Also, we’re using uint8 as the key — smaller keys = smaller storage footprint.
> 

---

## 💾 Voter Registry Using Bitmaps

```solidity
 
mapping(address => uint256) private voterRegistry;

```

Here’s where the contract gets real slick.

Instead of using:

```solidity
 
mapping(address => mapping(uint8 => bool)) voted;

```

We compress all of a voter's history into **a single uint256**:

- Each **bit** represents whether they voted on that proposal.
- Bit 0 = voted on Proposal 0
- Bit 1 = voted on Proposal 1
- …and so on, up to 256 proposals.

This lets us:

- ✅ Store all votes in one storage slot per address
- ✅ Check if someone voted using bitwise `AND`
- ✅ Record votes using bitwise `OR`

> Way cheaper than multiple mappings and multiple slots per user.
> 

---

## 👥 Proposal Voter Count

```solidity
 
mapping(uint8 => uint32) public proposalVoterCount;

```

- Tracks how many voters voted for each proposal.
- Optional but useful for analytics or UI.
- Again, we use `uint32` — more than enough range, lower gas.

---

## 🧾 Events

```solidity
 
event ProposalCreated(uint8 indexed proposalId, bytes32 name);
event Voted(address indexed voter, uint8 indexed proposalId);
event ProposalExecuted(uint8 indexed proposalId);

```

- `indexed` lets you filter logs more efficiently (e.g., show all proposals someone voted on).
- We keep these events minimal — emitting huge logs eats gas.

---

## 🧱 Function 1: `createProposal`

```solidity
  
function createProposal(bytes32 name, uint32 duration) external {
    require(duration > 0, "Duration must be > 0");

    uint8 proposalId = proposalCount;
    proposalCount++;

    Proposal memory newProposal = Proposal({
        name: name,
        voteCount: 0,
        startTime: uint32(block.timestamp),
        endTime: uint32(block.timestamp) + duration,
        executed: false
    });

    proposals[proposalId] = newProposal;

    emit ProposalCreated(proposalId, name);
}

```

### 🔍 What’s happening here?

1. **Input validation**
    
    ```solidity
      
    require(duration > 0, "Duration must be > 0");
    
    ```
    
    We make sure the voting duration isn’t zero. Otherwise, people could create proposals that can never be voted on.
    
2. **Generate a unique ID for this proposal**
    
    ```solidity
      
    uint8 proposalId = proposalCount;
    proposalCount++;
    
    ```
    
    We're using a simple counter (`uint8`) instead of pushing to an array. Arrays need dynamic resizing and extra gas for bounds management — this is much leaner.
    
3. **Create the Proposal in memory**
    
    ```solidity
      
    Proposal memory newProposal = Proposal({...});
    
    ```
    
    Why memory? Because it’s cheaper to build data structures off-chain and then write them to storage only once.
    
4. **Assign to storage**
    
    ```solidity
      
    proposals[proposalId] = newProposal;
    
    ```
    
    Now that we’ve assembled our proposal, we put it in the `proposals` mapping using its ID.
    
5. **Emit event**
    
    ```solidity
      
    emit ProposalCreated(proposalId, name);
    
    ```
    
    For frontend UIs or logs to track when a proposal is created.
    

---

## 🗳️ Function 2: `vote`

```solidity
  
function vote(uint8 proposalId) external {
    require(proposalId < proposalCount, "Invalid proposal");

    uint32 currentTime = uint32(block.timestamp);
    require(currentTime >= proposals[proposalId].startTime, "Voting not started");
    require(currentTime <= proposals[proposalId].endTime, "Voting ended");

    uint256 voterData = voterRegistry[msg.sender];
    uint256 mask = 1 << proposalId;
    require((voterData & mask) == 0, "Already voted");

    voterRegistry[msg.sender] = voterData | mask;

    proposals[proposalId].voteCount++;
    proposalVoterCount[proposalId]++;

    emit Voted(msg.sender, proposalId);
}

```

### 🔍 What’s going on?

1. **Check that the proposal exists**
    
    ```solidity
      
    require(proposalId < proposalCount, "Invalid proposal");
    
    ```
    
    The ID must be within the valid range (0 to proposalCount - 1).
    
2. **Check if voting is allowed (timewise)**
    
    ```solidity
      
    require(currentTime >= proposals[proposalId].startTime, "Voting not started");
    require(currentTime <= proposals[proposalId].endTime, "Voting ended");
    
    ```
    
    We're making sure that voting has started and hasn’t ended yet.
    
3. **Bitmask to check if already voted**
    
    ```solidity
      
    uint256 mask = 1 << proposalId;
    require((voterRegistry[msg.sender] & mask) == 0, "Already voted");
    
    ```
    
    - `1 << proposalId` creates a binary mask like `000100` (if `proposalId` is 2).
    - Bitwise AND checks if that bit is already set in the user’s registry.
    - If it’s set, the user already voted.
4. **Record the vote using bitwise OR**
    
    ```solidity
      
    voterRegistry[msg.sender] = voterData | mask;
    
    ```
    
    - Bitwise OR sets the bit at position `proposalId` to `1`, marking the vote.
5. **Increment vote counts**
    
    ```solidity
      
    proposals[proposalId].voteCount++;
    proposalVoterCount[proposalId]++;
    
    ```
    
6. **Emit a vote event**
    
    ```solidity
      
    emit Voted(msg.sender, proposalId);
    
    ```
    

---

## ✅ Function 3: `executeProposal`

```solidity
  
function executeProposal(uint8 proposalId) external {
    require(proposalId < proposalCount, "Invalid proposal");
    require(block.timestamp > proposals[proposalId].endTime, "Voting not ended");
    require(!proposals[proposalId].executed, "Already executed");

    proposals[proposalId].executed = true;

    emit ProposalExecuted(proposalId);
}

```

### 🔍 What this function does:

- Anyone can execute a proposal **after** the voting period ends.
- It ensures:
    - The proposal exists
    - The voting window has ended
    - The proposal hasn’t already been executed

Then we mark the proposal as executed and emit an event.

💡 **In real-world apps**, you might add proposal logic here — maybe triggering payments or DAO config changes.

---

## 👁️ Function 4: `hasVoted`

```solidity
  
function hasVoted(address voter, uint8 proposalId) external view returns (bool) {
    return (voterRegistry[voter] & (1 << proposalId)) != 0;
}

```

### 🔍 What this does:

- Creates a bitmask just like in the `vote()` function.
- Checks if that bit is set in the voter’s registry.
- Returns `true` if the voter already voted on that proposal.

> ⚡ Gas-efficient read: Only one storage access, and one bitwise operation.
> 

---

## 📊 Function 5: `getProposal`

```solidity
  
function getProposal(uint8 proposalId) external view returns (
    bytes32 name,
    uint32 voteCount,
    uint32 startTime,
    uint32 endTime,
    bool executed,
    bool active
) {
    require(proposalId < proposalCount, "Invalid proposal");

    Proposal storage proposal = proposals[proposalId];

    return (
        proposal.name,
        proposal.voteCount,
        proposal.startTime,
        proposal.endTime,
        proposal.executed,
        (block.timestamp >= proposal.startTime && block.timestamp <= proposal.endTime)
    );
}

```

### 🔍 Breakdown:

- Checks that the proposal exists.
- Returns all its fields, **plus** an extra `active` flag that indicates if the voting is ongoing at the moment.
- This is useful for UI/UX — lets the frontend know whether to show a “Vote” button or not.

> ✅ Efficient: all variables are compact and the function is view, meaning it’s free to call off-chain (via JSON-RPC / frontend).
> 

## 🧵 Wrapping It All Up

We started with a plain, beginner-style voting contract — simple, readable, but not very gas-friendly. Then we rebuilt it from the ground up with one goal in mind: **efficiency**.

By switching to fixed-size types, using mappings instead of arrays, and packing vote history into bitmaps, we drastically reduced the contract's gas footprint **without losing any core functionality**.

This lesson wasn’t just about writing a better voting contract. It was about learning how to **think like a Solidity developer who ships real, scalable apps** — apps that are mindful of cost, performance, and user experience on-chain.

Because in the world of smart contracts, gas isn’t just a technical detail — it’s the difference between something that’s usable and something that’s too expensive to touch.

So here’s your next step:

✅ Revisit your past contracts

✅ Look for places where you can optimize

✅ Ask yourself — *do I really need a string here?* *Do I need a dynamic array?*

✅ Start thinking in bits, bytes, and storage slots

Because *writing Solidity is one thing. Writing gas-efficient Solidity?*

That’s when you level up.