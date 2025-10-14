# DecentralisedGovernance Contract

Welcome back to **30 Days of Solidity**.

Not long ago, you wrote your first **voting contract** —

a simple, elegant way for users to propose ideas and let others vote "yes" or "no."

That was your first step into the world of on-chain decision-making.

But today?

Today, we’re taking it to a whole new level.

Because real decentralized systems don’t just stop at collecting votes.

They need real **governance** —

with rules, protections, accountability, and execution — all built directly into the code.

And that’s exactly what we’re going to build next.

---

Before we dive into the contract itself, let's take a second to understand **why** governance matters —

and what exactly we mean when we say **DAO**.

---

## 🌐 What is a DAO, Really?

A **DAO** — short for **Decentralized Autonomous Organization** —

is a new kind of system where **the community, not a company**, runs the show.

No CEOs. No managers. No lawyers drafting contracts behind closed doors.

Instead, everything happens **on-chain**, transparently, based on the community's votes.

Imagine an online club where:

- Everyone who holds tokens gets a say in what happens.
- Major decisions — like how funds are spent, what features are built, or what upgrades happen — are voted on.
- Once a vote passes, the smart contract **automatically** executes the decision, without waiting for human approval.

It’s **pure democracy**, written in Solidity.

If you’ve ever heard of projects like **Uniswap**, **Aave**, or **Compound** —

they’re all governed by their communities, using DAO systems that run **just like the one you’re about to build**.

---

## ✨ What Makes This Different from Our Earlier Voting Contract?

When you built your simple voting system earlier,

you learned how to collect votes and decide outcomes.

Today’s project adds real-world complexity —

but in a way that’s still beginner-friendly.

In this upgraded DAO system:

- Users will **lock up a small deposit** when proposing an idea, to prevent spam.
- **Voting power** will depend on **how many tokens** someone owns — meaning bigger stakeholders have bigger voices.
- Proposals must **meet a quorum** — a minimum level of participation — to be valid.
- Even after a proposal wins, it can't be executed immediately.
    
    A **timelock** ensures there’s a public waiting period before anything happens, giving everyone time to react if needed.
    
- And when the time comes, proposals will **actually execute real actions** —
    
    like calling functions on other contracts automatically.
    

This isn’t just "let’s vote."

This is **let's vote, validate, delay, and execute — the full real-world flow**.

---

## 🚀 Why This Is Such a Huge Step

Today, you're not just learning Solidity syntax.

You’re learning how **blockchain societies govern themselves.**

You're learning how systems like:

- Treasury allocations
- Protocol upgrades
- Feature changes
- Ecosystem funding

are decided **not by companies**, but **by communities**, enforced by unstoppable smart contracts.

You're learning **how power gets distributed**, and **how trust is shifted from people to code**.

You're learning how to build a system where **no one needs permission** —

the rules are there, clear and automatic, for everyone to see.

---

## 🎯 What You're About to Build

In this project, you’ll create a smart contract that lets anyone:

- Propose an idea (after locking a token deposit)
- Let the community vote based on how many tokens they hold
- Ensure a minimum number of voters participate (quorum)
- Add a waiting period after the vote (timelock)
- Automatically execute the winning proposal’s actions if the vote passes

All of it — **fair**, **transparent**, and **trustless**.

This is the system real DeFi platforms use every single day —

and now, you're going to write it with your own hands.

---

# 🌱 Ready to Build?

First, we'll take a slow, careful walk through the **full contract code** —

to get a bird’s eye view of the entire system you’re about to master.

Then, just like always, we’ll break it down **section by section**,

**concept by concept**,

until you know exactly how every piece fits together.

This is your next leap —

from building apps,

to building **governments**.

Let's do this. 🚀🌾

---

# Full Contract: Decentralized Governance System

Before we start breaking it all down,

let’s take a full look at the contract you’re about to master.

Read through it once — not to memorize —

but to **get a feel for the big picture**:

how proposals are created, how votes are collected, how execution is delayed and then triggered.

It’s totally fine if it feels like a lot right now.

Once we break it down together, you’ll see it’s just simple building blocks stacked smartly.

Alright — here’s the full DAO contract you’re writing today:

---

```solidity
     
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "@openzeppelin/contracts/utils/math/SafeCast.sol";
import "@openzeppelin/contracts/security/ReentrancyGuard.sol";

/// @title Decentralized Governance System (ERC-20 Based)
/// @notice A DAO with weighted voting, quorum, proposal deposit, and timelock.
contract DecentralizedGovernance is ReentrancyGuard {
    using SafeCast for uint256;

    struct Proposal {
        uint256 id;
        string description;
        uint256 deadline;
        uint256 votesFor;
        uint256 votesAgainst;
        bool executed;
        address proposer;
        bytes[] executionData;
        address[] executionTargets;
        uint256 executionTime;
    }

    IERC20 public governanceToken;
    mapping(uint256 => Proposal) public proposals;
    mapping(uint256 => mapping(address => bool)) public hasVoted;

    uint256 public nextProposalId;
    uint256 public votingDuration;
    uint256 public timelockDuration;
    address public admin;
    uint256 public quorumPercentage = 5;
    uint256 public proposalDepositAmount = 10;

    event ProposalCreated(uint256 id, string description, address proposer, uint256 depositAmount);
    event Voted(uint256 proposalId, address voter, bool support, uint256 weight);
    event ProposalExecuted(uint256 id, bool passed);
    event QuorumNotMet(uint256 id, uint256 votesTotal, uint256 quorumNeeded);
    event ProposalDepositPaid(address proposer, uint256 amount);
    event ProposalDepositRefunded(address proposer, uint256 amount);
    event TimelockSet(uint256 duration);
    event ProposalTimelockStarted(uint256 proposalId, uint256 executionTime);

    modifier onlyAdmin() {
        require(msg.sender == admin, "Only admin can call this");
        _;
    }

    constructor(address _governanceToken, uint256 _votingDuration, uint256 _timelockDuration) {
        governanceToken = IERC20(_governanceToken);
        votingDuration = _votingDuration;
        timelockDuration = _timelockDuration;
        admin = msg.sender;
        emit TimelockSet(_timelockDuration);
    }

    function setQuorumPercentage(uint256 _quorumPercentage) external onlyAdmin {
        require(_quorumPercentage <= 100, "Quorum percentage must be between 0 and 100");
        quorumPercentage = _quorumPercentage;
    }

    function setProposalDepositAmount(uint256 _proposalDepositAmount) external onlyAdmin {
        proposalDepositAmount = _proposalDepositAmount;
    }

    function setTimelockDuration(uint256 _timelockDuration) external onlyAdmin {
        timelockDuration = _timelockDuration;
        emit TimelockSet(_timelockDuration);
    }

    function createProposal(
        string calldata _description,
        address[] calldata _targets,
        bytes[] calldata _calldatas
    ) external returns (uint256) {
        require(governanceToken.balanceOf(msg.sender) >= proposalDepositAmount, "Insufficient tokens for deposit");
        require(_targets.length == _calldatas.length, "Targets and calldatas length mismatch");

        governanceToken.transferFrom(msg.sender, address(this), proposalDepositAmount);
        emit ProposalDepositPaid(msg.sender, proposalDepositAmount);

        proposals[nextProposalId] = Proposal({
            id: nextProposalId,
            description: _description,
            deadline: block.timestamp + votingDuration,
            votesFor: 0,
            votesAgainst: 0,
            executed: false,
            proposer: msg.sender,
            executionData: _calldatas,
            executionTargets: _targets,
            executionTime: 0
        });

        emit ProposalCreated(nextProposalId, _description, msg.sender, proposalDepositAmount);

        nextProposalId++;
        return nextProposalId - 1;
    }

    function vote(uint256 proposalId, bool support) external {
        Proposal storage proposal = proposals[proposalId];
        require(block.timestamp < proposal.deadline, "Voting period over");
        require(governanceToken.balanceOf(msg.sender) > 0, "No governance tokens");
        require(!hasVoted[proposalId][msg.sender], "Already voted");

        uint256 weight = governanceToken.balanceOf(msg.sender);

        if (support) {
            proposal.votesFor += weight;
        } else {
            proposal.votesAgainst += weight;
        }

        hasVoted[proposalId][msg.sender] = true;

        emit Voted(proposalId, msg.sender, support, weight);
    }

    function finalizeProposal(uint256 proposalId) external {
        Proposal storage proposal = proposals[proposalId];
        require(block.timestamp >= proposal.deadline, "Voting period not yet over");
        require(!proposal.executed, "Proposal already executed");
        require(proposal.executionTime == 0, "Execution time already set");

        uint256 totalSupply = governanceToken.totalSupply();
        uint256 totalVotes = proposal.votesFor + proposal.votesAgainst;
        uint256 quorumNeeded = (totalSupply * quorumPercentage) / 100;

        if (totalVotes >= quorumNeeded && proposal.votesFor > proposal.votesAgainst) {
            proposal.executionTime = block.timestamp + timelockDuration;
            emit ProposalTimelockStarted(proposalId, proposal.executionTime);
        } else {
            proposal.executed = true;
            emit ProposalExecuted(proposalId, false);
            if (totalVotes < quorumNeeded) {
                emit QuorumNotMet(proposalId, totalVotes, quorumNeeded);
            }
        }
    }

    function executeProposal(uint256 proposalId) external nonReentrant {
        Proposal storage proposal = proposals[proposalId];
        require(!proposal.executed, "Proposal already executed");
        require(proposal.executionTime > 0 && block.timestamp >= proposal.executionTime, "Timelock not yet expired");

        proposal.executed = true; // set early to prevent reentrancy

        bool passed = proposal.votesFor > proposal.votesAgainst;

        if (passed) {
            for (uint256 i = 0; i < proposal.executionTargets.length; i++) {
                (bool success, bytes memory returnData) = proposal.executionTargets[i].call(proposal.executionData[i]);
                require(success, string(returnData));
            }
            emit ProposalExecuted(proposalId, true);
            governanceToken.transfer(proposal.proposer, proposalDepositAmount);
            emit ProposalDepositRefunded(proposal.proposer, proposalDepositAmount);
        } else {
            emit ProposalExecuted(proposalId, false);
        }
    }

    function getProposalResult(uint256 proposalId) external view returns (string memory) {
        Proposal storage proposal = proposals[proposalId];
        require(proposal.executed, "Proposal not yet executed");

        uint256 totalSupply = governanceToken.totalSupply();
        uint256 totalVotes = proposal.votesFor + proposal.votesAgainst;
        uint256 quorumNeeded = (totalSupply * quorumPercentage) / 100;

        if (totalVotes < quorumNeeded) {
            return "Proposal FAILED - Quorum not met";
        } else if (proposal.votesFor > proposal.votesAgainst) {
            return "Proposal PASSED";
        } else {
            return "Proposal REJECTED";
        }
    }

    function getProposalDetails(uint256 proposalId) external view returns (Proposal memory) {
        return proposals[proposalId];
    }
}

```

---

✅ Now that you’ve seen the whole thing at once,

next, we’ll start **breaking it down section-by-section** —

explaining every variable, every event, and every function with beginner-friendly flow.

---

# 📦 Imports — Bringing in the Tools We Need

Before we start building the actual governance logic,

we pull in a few important tools from OpenZeppelin —

the trusted library for secure, audited smart contract code.

You can think of these imports like grabbing your **hammer, wrench, and helmet** before starting to build something serious.

Here's what we're importing:

---

```solidity
     
import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "@openzeppelin/contracts/utils/math/SafeCast.sol";
import "@openzeppelin/contracts/security/ReentrancyGuard.sol";

```

---

## 🔍 What Each Import Does

---

### 🪙 `IERC20.sol`

This brings in the **interface** for ERC-20 tokens —

meaning it defines the basic functions (`balanceOf`, `transfer`, `transferFrom`, etc.) that any ERC-20 token must have.

We use this so that our governance contract can **talk to** whatever ERC-20 token is used for voting power —

without caring about its full internal implementation.

✅ Whether your token is simple, complex, or fancy —

as long as it follows ERC-20, our contract can interact with it.

---

### 🔢 `SafeCast.sol`

This utility helps with **safe type conversions**.

In Solidity, sometimes you need to convert between `uint256` and smaller types like `uint32`, `uint64`, etc.

Without safety checks, bad conversions can lead to **overflow** or **silent bugs**.

✅ `SafeCast` makes sure that any number you shrink down **won't lose important data** —

otherwise, the transaction will automatically revert.

In our case, it's mainly here as a best practice —

because we’re dealing with time, votes, and other numerical operations where safety matters.

---

### 🛡️ `ReentrancyGuard.sol`

This adds protection against a nasty category of attacks called **reentrancy attacks**.

Quick refresher:

A reentrancy attack happens when a smart contract calls an external address before updating its own internal state —

and that external address maliciously **calls back into** the contract and **abuses** unfinished logic.

✅ By using `ReentrancyGuard`, we can **lock** functions like `executeProposal()` so that they **can't be entered again** before the first call finishes.

This is absolutely crucial when moving tokens or executing external calls.

---

# 🏛️ Contract Declaration — Laying the Foundation

Now that we've imported all the essential tools,

it's time to **lay the foundation** for our governance system.

Here’s how we officially start building the contract:

---

```solidity
     
contract DecentralizedGovernance is ReentrancyGuard {
    using SafeCast for uint256;

```

---

## 🔍 What’s Happening Here?

---

### 🏗️ `contract DecentralizedGovernance is ReentrancyGuard`

This line **declares** our smart contract and gives it a name:

👉 `DecentralizedGovernance`.

It also **inherits** from `ReentrancyGuard`,

which means our contract automatically gets reentrancy protection for critical functions —

especially when we’re executing proposals that might call other contracts.

✅ This protects us from attackers trying sneaky tricks like double-spending or recursive calls during execution.

---

### 🧹 `using SafeCast for uint256`

This line says:

> "Hey Solidity — anytime I have a uint256,
> 
> 
> give me access to the **SafeCast** functions automatically."
> 

It’s like giving `uint256` variables **extra superpowers** —

allowing them to safely downcast themselves to smaller types (`uint32`, `uint64`, etc.) whenever needed,

without having to manually call `SafeCast.safeCastFunction(value)` each time.

✅ Cleaner code.

✅ Safer math operations.

---

# 🧩 Setting Up the Core Data: Proposals and Voting Records

Now that the contract is declared,

the next step is to define **what kind of information** our DAO will manage.

And at the heart of any governance system is one key thing: **Proposals**.

We need a way to store:

- What the proposal is about
- Who created it
- How people voted on it
- Whether it passed or failed
- And what actions should happen if it succeeds

Let’s set up the structure for all of that now.

---

## 📜 Full Code for Struct + Mappings

```solidity
     
struct Proposal {
    uint256 id;
    string description;
    uint256 deadline;
    uint256 votesFor;
    uint256 votesAgainst;
    bool executed;
    address proposer;
    bytes[] executionData;
    address[] executionTargets;
    uint256 executionTime;
}

mapping(uint256 => Proposal) public proposals;
mapping(uint256 => mapping(address => bool)) public hasVoted;

```

---

# 🔍 Deep Dive: Line-by-Line Breakdown

---

## 🏛️ `struct Proposal`

This **blueprint** defines what a **single proposal** looks like inside the contract.

Each proposal has:

- **`id`**: A unique ID number for tracking.
- **`description`**: A human-readable explanation of what the proposal is about.
- **`deadline`**: A timestamp marking when voting closes.
- **`votesFor`**: How many votes were cast in favor.
- **`votesAgainst`**: How many votes were cast against.
- **`executed`**: A boolean (true/false) to track whether the proposal has already been executed.
- **`proposer`**: The address of the person who created the proposal.
- **`executionData`**: The actual data payloads to be called on other contracts if the proposal passes.
- **`executionTargets`**: The list of contract addresses that these payloads should be sent to.
- **`executionTime`**: A future timestamp after the timelock when the proposal can officially be executed.

✅ This struct holds **everything** we need to fully manage a DAO proposal from start to finish —

from creation → voting → execution.

---

```solidity
mapping(uint256 => Proposal) public proposals;
```

This sets up a **mapping**, or a **lookup table**,

where each `Proposal` struct is stored and accessed by its unique `id`.

Think of it like:

> "Proposal ID #2 → this entire Proposal struct"
> 

✅ This allows the contract (and users) to quickly fetch details about any proposal just by its ID number.

✅ Marked as **public** so external users and frontends can easily fetch proposal data without writing extra getter functions.

---

```solidity
mapping(uint256 => mapping(address => bool)) public hasVoted;
```

This is a **double mapping** — a little trickier, but very important.

- The first key is the `proposalId`
- The second key is the `voter's address`
- The value is a simple `bool` (true/false)

This lets us **track whether a specific user has already voted** on a specific proposal.

✅ It prevents double-voting.

✅ It keeps the voting process fair and clean.

---

# 🏗️ Setting Up the Governance Rules

Now that we’ve defined **how proposals** will be stored and tracked,

the next step is to set up the **rules** that will control **how governance actually works**.

Every DAO needs to answer important questions like:

- Who is allowed to vote?
- How long does voting stay open?
- Who can create proposals?
- How many people need to participate for a proposal to be valid?

This next set of variables defines all of that.

---

## 📜 Full Code for Governance Settings

```solidity
     
IERC20 public governanceToken;
uint256 public nextProposalId;
uint256 public votingDuration;
uint256 public timelockDuration;
address public admin;
uint256 public quorumPercentage = 5;
uint256 public proposalDepositAmount = 10;

```

---

# 🔍 Deep Dive: Line-by-Line Breakdown

---

```solidity
IERC20 public governanceToken;
```

This is **one of the most important pieces** of the whole system.

The **governanceToken** is the **ERC-20 token** that represents **voting power**.

- Whoever holds this token gets the **right to vote** on proposals.
- The **more tokens** you have, the **stronger** your voting weight.
- If you don't hold any governance tokens, you can't vote.

✅ This token becomes the "citizenship badge" of your DAO.

✅ It's what gives users the right to propose ideas, vote on changes, and have a real voice.

---

> 🔥 Important:
> 
> 
> This token could be anything — it could be a custom DAO token, a wrapped staking token, or even something like UNI (Uniswap) or COMP (Compound) in real DAOs.
> 

Our contract **talks** to it through the standard ERC-20 interface (`IERC20`),

which means it doesn't care about the token’s internal logic —

as long as it behaves like a standard ERC-20.

---

```solidity
uint256 public nextProposalId;
```

This keeps track of the **next ID** that will be assigned when someone creates a new proposal.

It starts at `0` by default and **increments by 1** every time a new proposal is created.

✅ This ensures every proposal gets a **unique ID** without collisions.

---

```solidity
uint256 public votingDuration;
```

This defines **how long** each proposal stays open for voting.

- It’s measured in **seconds** (because that's how timestamps work in Solidity).
- Once the proposal’s deadline (`block.timestamp + votingDuration`) is reached, no more votes are allowed.

✅ Keeps the voting period **finite and predictable**.

---

```solidity
uint256 public timelockDuration;
```

This is the **waiting period** **after** a proposal wins before it can actually be executed.

The timelock gives users time to:

- Review the outcome
- Spot any malicious proposals
- Exit the system if needed (in case of major changes)

✅ It adds a **critical safety buffer** before real on-chain actions happen.

---

```solidity
address public admin;
```

This stores the **admin address** — the wallet that deployed the contract.

The admin has a few **special permissions**, like:

- Changing the quorum percentage
- Adjusting the proposal deposit amount
- Tweaking the timelock duration

✅ Admin control is limited and mostly focused on **setting parameters**, not interfering with voting itself.

✅ In real-world DAOs, admin control is often reduced or even removed over time (progressively decentralizing governance).

---

```solidity
uint256 public quorumPercentage = 5;
```

This sets the **minimum percentage of total votes** that must participate for a proposal to be valid.

For example:

- If `quorumPercentage = 5`
- And the total supply of tokens is 1,000,000
- Then at least 5% (50,000 tokens worth of votes) must participate

✅ Without a quorum, a tiny group of voters could pass decisions unfairly.

✅ Quorum ensures **enough of the community is involved** before big changes happen.

---

```solidity
uint256 public proposalDepositAmount = 10;
```

This defines **how many governance tokens** a user must lock when creating a proposal.

It acts as a **spam filter** —

preventing people from flooding the system with random proposals without consequences.

✅ If your proposal wins and executes, you get your deposit back.

✅ If your proposal fails, the deposit is lost or locked away — discouraging low-quality or troll proposals.

---

# 📢 Events — Broadcasting Important Moments

In a real DAO, it’s not enough to just store information on-chain.

You also want to **broadcast important milestones** to the outside world —

so that frontends, apps, explorers, and users can **listen**, **react**, and **display** the latest updates.

That’s exactly what **events** do.

You can think of them as **loud announcements** the contract makes every time something critical happens.

---

## 📜 Full List of Events

```solidity
     
event ProposalCreated(uint256 id, string description, address proposer, uint256 depositAmount);
event Voted(uint256 proposalId, address voter, bool support, uint256 weight);
event ProposalExecuted(uint256 id, bool passed);
event QuorumNotMet(uint256 id, uint256 votesTotal, uint256 quorumNeeded);
event ProposalDepositPaid(address proposer, uint256 amount);
event ProposalDepositRefunded(address proposer, uint256 amount);
event TimelockSet(uint256 duration);
event ProposalTimelockStarted(uint256 proposalId, uint256 executionTime);

```

---

# 🔍 Deep Dive: What Each Event Means

---

## 🧱 `ProposalCreated`

```solidity
     
event ProposalCreated(uint256 id, string description, address proposer, uint256 depositAmount);

```

Whenever a new proposal is created,

this event fires — announcing:

- The **ID** of the proposal
- A **short description** of what it’s about
- The **address** of the proposer
- How much **deposit** they locked in to create it

✅ Frontends use this to **list new proposals** live as they’re created.

---

## 🗳️ `Voted`

```solidity
     
event Voted(uint256 proposalId, address voter, bool support, uint256 weight);

```

Whenever someone casts a vote,

this event broadcasts:

- Which **proposal** they voted on
- Which **address** voted
- Whether they voted **for** or **against**
- How much **voting power** (token weight) they used

✅ Frontends can show **live voting updates** based on this event!

---

## 🏁 `ProposalExecuted`

```solidity
     
event ProposalExecuted(uint256 id, bool passed);

```

When a proposal is finally executed —

whether it passed or failed —

this event fires, letting everyone know:

- Which **proposal** was finalized
- Whether it **passed** (true) or **failed** (false)

✅ Helps track which proposals were actually **completed**.

---

## 📉 `QuorumNotMet`

```solidity
     
event QuorumNotMet(uint256 id, uint256 votesTotal, uint256 quorumNeeded);

```

If a proposal **fails** because not enough people participated (quorum not met),

this event explains exactly:

- Which **proposal** failed
- How many **votes** were collected
- How many **votes were needed** to meet quorum

✅ Very useful for analyzing voter turnout and DAO health!

---

## 💰 `ProposalDepositPaid`

```solidity
     
event ProposalDepositPaid(address proposer, uint256 amount);

```

When someone submits a proposal and pays the deposit,

this event records:

- The **proposer’s address**
- The **amount** of tokens locked

✅ Frontends can show deposit tracking and proof of commitment.

---

## 🎁 `ProposalDepositRefunded`

```solidity
     
event ProposalDepositRefunded(address proposer, uint256 amount);

```

If a proposal **passes and is executed**,

the proposer **gets their deposit back** —

and this event logs that refund happening.

✅ Helps prove that winning proposers are rewarded.

---

## ⏰ `TimelockSet`

```solidity
     
event TimelockSet(uint256 duration);

```

When the admin **changes the timelock duration** (the waiting period between passing and execution),

this event announces the **new timelock time**.

✅ Keeps governance transparent — no hidden parameter changes.

---

## ⏳ `ProposalTimelockStarted`

```solidity
     
event ProposalTimelockStarted(uint256 proposalId, uint256 executionTime);

```

When a proposal **wins the vote** but enters the **timelock delay** before execution,

this event announces:

- Which **proposal** is entering the timelock
- When exactly it will be **eligible for execution**

✅ Super important for showing when proposals are in the waiting zone.

---

# 🛡️ Admin-Only Access (One Last Time!)

```solidity
     
modifier onlyAdmin() {
    require(msg.sender == admin, "Only admin can call this");
    _;
}

```

This is our classic **onlyAdmin** modifier.

It simply **checks** that the person calling a certain function is the **admin** —

the one who deployed the contract.

If you're not the admin, the call **reverts** with an error.

If you are the admin, the function keeps going normally (`_` means "continue with the function").

✅ We use this to **protect sensitive settings** —

like changing the quorum percentage or updating the timelock duration —

so that random users can't mess with the governance rules.

---

# 🏗️Constructor — Setting Up the DAO's Core Settings

Before our governance system can start running,

we need to **set some starting parameters** —

things like which token we’ll use for voting, how long votes stay open, and who the first admin is.

The constructor handles all that **right when the contract is deployed**.

Here’s what it looks like:

---

## 📜 Full Code

```solidity
     
constructor(address _governanceToken, uint256 _votingDuration, uint256 _timelockDuration) {
    governanceToken = IERC20(_governanceToken);
    votingDuration = _votingDuration;
    timelockDuration = _timelockDuration;
    admin = msg.sender;
    emit TimelockSet(_timelockDuration);
}

```

---

# 🔍 Line-by-Line Breakdown

---

```solidity
governanceToken = IERC20(_governanceToken);
```

When the contract is deployed,

we expect the deployer to **tell us the address** of the ERC-20 token that will be used for governance voting.

We save that token’s address inside the contract so that:

- We know where to check people's balances.
- We know which token gives users voting power.

✅ This locks in **which token controls the DAO**.

---

```solidity
votingDuration = _votingDuration;
```

This sets the **number of seconds** that each proposal will stay open for voting.

✅ After a proposal is created, it will automatically **expire** when this much time has passed.

✅ Makes sure voting periods aren’t endless.

---

```solidity
timelockDuration = _timelockDuration;
```

This sets the **mandatory waiting time**

between a proposal *winning* and it actually being **executed**.

✅ This protects users by giving them time to exit or react to major changes before they’re finalized.

---

```solidity
admin = msg.sender;
```

Whoever deploys the contract becomes the **admin**.

✅ The admin can tweak governance settings later (like quorum %, deposit size, timelock duration).

---

```solidity
emit TimelockSet(_timelockDuration);
```

As soon as the contract is deployed,

we **emit an event** announcing the timelock setting to the world.

✅ This keeps everything **transparent** —

anyone watching the blockchain will know immediately what the starting timelock was.

---

# 🛠️ Updating the Quorum Percentage

```solidity
     
function setQuorumPercentage(uint256 _quorumPercentage) external onlyAdmin {
    require(_quorumPercentage <= 100, "Quorum percentage must be between 0 and 100");
    quorumPercentage = _quorumPercentage;
}

```

---

# 🔍 What’s Happening Here?

This function lets the **admin** change the **quorum percentage** —

which is the **minimum percentage of total tokens that must vote** for a proposal to be valid.

✅ Only the admin can call this (thanks to the `onlyAdmin` modifier).

✅ The new quorum must be **between 0% and 100%** —

otherwise, it reverts with an error.

Once set, the new `quorumPercentage` will apply to **all future proposals**.

---

# 💰 Updating the Proposal Deposit Amount

```solidity
     
function setProposalDepositAmount(uint256 _proposalDepositAmount) external onlyAdmin {
    proposalDepositAmount = _proposalDepositAmount;
}

```

---

# 🔍 What’s Happening Here?

This function allows the **admin** to change the **amount of tokens** a user must deposit to create a proposal.

✅ Only the admin can call this (because of the `onlyAdmin` modifier).

✅ There’s no range check here — the admin should set it responsibly depending on the DAO’s needs.

If the deposit is too small, spam proposals could flood the system.

If it’s too high, it might discourage participation.

---

# ⏳ Updating the Timelock Duration

```solidity
     
function setTimelockDuration(uint256 _timelockDuration) external onlyAdmin {
    timelockDuration = _timelockDuration;
    emit TimelockSet(_timelockDuration);
}

```

---

# 🔍 What’s Happening Here?

This function lets the **admin** update the **timelock duration** —

the waiting period between a proposal passing and it being executed.

✅ Only the admin can call it (`onlyAdmin` modifier again).

✅ After setting the new timelock, it **emits an event** (`TimelockSet`) to publicly announce the change.

This keeps everything **transparent**, so users know if the governance process has been sped up or slowed down.

---

# 🌟 Why This Matters

The timelock gives the community **time to react** after a proposal passes.

If the timelock is too short, decisions might feel rushed.

If it’s too long, execution could be frustratingly delayed.

✅ Being able to adjust it keeps the DAO **flexible** as it grows and evolves.

---

# 🧱 Creating a Proposal — Let’s Make Some Decisions!

In a decentralized governance system,

**everything starts with a proposal**.

Whether it's suggesting a new feature, upgrading a contract, or changing a setting —

it all begins when **someone raises an idea** and asks the community to vote.

This function, `createProposal()`, is where users **officially submit their proposals** to the DAO —

locking a deposit, defining what actions should happen if the proposal wins, and setting a voting deadline.

Let’s take a look at how it works.

---

## 📜 Full Code

```solidity
     
function createProposal(
    string calldata _description,
    address[] calldata _targets,
    bytes[] calldata _calldatas
) external returns (uint256) {
    require(governanceToken.balanceOf(msg.sender) >= proposalDepositAmount, "Insufficient tokens for deposit");
    require(_targets.length == _calldatas.length, "Targets and calldatas length mismatch");

    governanceToken.transferFrom(msg.sender, address(this), proposalDepositAmount);
    emit ProposalDepositPaid(msg.sender, proposalDepositAmount);

    proposals[nextProposalId] = Proposal({
        id: nextProposalId,
        description: _description,
        deadline: block.timestamp + votingDuration,
        votesFor: 0,
        votesAgainst: 0,
        executed: false,
        proposer: msg.sender,
        executionData: _calldatas,
        executionTargets: _targets,
        executionTime: 0
    });

    emit ProposalCreated(nextProposalId, _description, msg.sender, proposalDepositAmount);

    nextProposalId++;
    return nextProposalId - 1;
}

```

---

# 🔍 Step-by-Step Breakdown

---

### 📜 What Inputs Does It Take?

- `string calldata _description` — a short text describing what the proposal is about.
- `address[] calldata _targets` — the addresses of the contracts this proposal will interact with (if it passes).
- `bytes[] calldata _calldatas` — the actual function call data that will be sent to each target (like a function call packaged into bytes).

✅ These inputs define both **what the proposal means** and **what it will do** if approved.

---

### 🛡️ Step 1: Check if the User Has Enough Tokens

```solidity
     
require(governanceToken.balanceOf(msg.sender) >= proposalDepositAmount, "Insufficient tokens for deposit");

```

Before someone can create a proposal,

they must have **at least** enough governance tokens to cover the **proposal deposit**.

✅ This prevents spam — people need real "skin in the game" to propose something.

---

### 📏 Step 2: Make Sure Targets and Calldata Match

```solidity
     
require(_targets.length == _calldatas.length, "Targets and calldatas length mismatch");

```

Each target contract must have **one corresponding function call** (`calldata`).

We check to make sure the lists are the **same length**.

✅ This prevents execution errors later.

---

### 💸 Step 3: Collect the Proposal Deposit

```solidity
     
governanceToken.transferFrom(msg.sender, address(this), proposalDepositAmount);
emit ProposalDepositPaid(msg.sender, proposalDepositAmount);

```

We **transfer** the proposer’s deposit tokens into the DAO contract.

Then we **emit an event** announcing the payment.

✅ Now the system knows the proposer is serious.

---

### 🏗️ Step 4: Save the New Proposal

```solidity
     
proposals[nextProposalId] = Proposal({...});

```

We create a new `Proposal` struct and fill it with:

- Its unique ID (`nextProposalId`)
- The description the user provided
- A deadline (now + voting duration)
- Zero votes initially
- The proposer’s address
- The list of target addresses and call data
- No execution time yet (set later if the proposal wins)

✅ This officially stores the proposal in our DAO's memory.

---

### 📢 Step 5: Announce the New Proposal

```solidity
     
emit ProposalCreated(nextProposalId, _description, msg.sender, proposalDepositAmount);

```

We fire an event so that frontends and users can **see** that a new proposal has been created.

---

### 🆔 Step 6: Update the Proposal Counter

```solidity
     
nextProposalId++;

```

We increment the counter — so the next proposal will get a new unique ID.

Finally, the function **returns the ID** of the newly created proposal.

✅ Now the proposal is live and ready for the community to start voting!

---

# 🌟 Why This Function Matters

This function is **where governance begins**.

It makes sure:

- Only serious users can propose changes.
- Proposals are properly linked to real actions (not just ideas).
- Everything is recorded on-chain, visible, and transparent.

✅ Without this function, there would be no decisions to vote on!

It’s the first step of **community-driven control**.

---

# 🗳️ Casting Your Vote — Have Your Say in the DAO

Now that users can create proposals,

the next step is **giving the community a voice**.

This function, `vote()`, is where users officially **cast their votes** —

either in support of a proposal or against it —

with their voting power based on how many governance tokens they hold.

Voting is how ideas turn into decisions.

Let’s dive into how it works.

---

## 📜 Full Code

```solidity
     
function vote(uint256 proposalId, bool support) external {
    Proposal storage proposal = proposals[proposalId];
    require(block.timestamp < proposal.deadline, "Voting period over");
    require(governanceToken.balanceOf(msg.sender) > 0, "No governance tokens");
    require(!hasVoted[proposalId][msg.sender], "Already voted");

    uint256 weight = governanceToken.balanceOf(msg.sender);

    if (support) {
        proposal.votesFor += weight;
    } else {
        proposal.votesAgainst += weight;
    }

    hasVoted[proposalId][msg.sender] = true;

    emit Voted(proposalId, msg.sender, support, weight);
}

```

---

# 🔍 Step-by-Step Breakdown

---

### 📜 Step 1: Load the Proposal

```solidity
     
Proposal storage proposal = proposals[proposalId];

```

We first **load** the proposal that the user wants to vote on,

using the proposal’s ID.

✅ Now we have all the info (deadline, votes, proposer, etc.) at our fingertips.

---

### 🛑 Step 2: Make Sure Voting is Still Open

```solidity
     
require(block.timestamp < proposal.deadline, "Voting period over");

```

Voting is only allowed **before** the proposal’s deadline.

If the voting window has closed, the function **reverts**.

✅ This ensures votes are only counted during the official voting period.

---

### 🪙 Step 3: Make Sure the User Holds Governance Tokens

```solidity
     
require(governanceToken.balanceOf(msg.sender) > 0, "No governance tokens");

```

If you don’t hold any governance tokens,

you don't get to vote.

✅ This keeps voting fair — only real stakeholders participate.

---

### 🚫 Step 4: Prevent Double Voting

```solidity
     
require(!hasVoted[proposalId][msg.sender], "Already voted");

```

Each address can **only vote once** per proposal.

✅ No spamming, no changing your mind later.

Once you vote, it's locked in!

---

### ⚖️ Step 5: Calculate Voting Power

```solidity
     
uint256 weight = governanceToken.balanceOf(msg.sender);

```

The **more tokens** you have, the **more weight** your vote carries.

✅ Bigger token holders have proportionally bigger influence.

(That’s why governance tokens are valuable — they represent real power.)

---

### 🗳️ Step 6: Count the Vote

```solidity
     
if (support) {
    proposal.votesFor += weight;
} else {
    proposal.votesAgainst += weight;
}

```

Depending on whether the user supports or opposes the proposal:

- We add their weight to the `votesFor`
- Or we add it to the `votesAgainst`

✅ Every vote shifts the balance!

---

### 📝 Step 7: Mark the User as Voted

```solidity
     
hasVoted[proposalId][msg.sender] = true;

```

We record that this user **has now voted** on this proposal.

They can’t vote again.

✅ Keeps everything clean and tamper-proof.

---

### 📢 Step 8: Emit a Voting Event

```solidity
     
emit Voted(proposalId, msg.sender, support, weight);

```

Finally, we **broadcast an event** —

telling the outside world who voted, on which proposal, and with how much power.

✅ Frontends can update voting tallies live!

---

# 🌟 Why This Function Matters

This function is the **lifeblood of governance**.

Without it, proposals would just sit there gathering dust.

✅ It makes voting weighted, fair, secure, and auditable —

exactly what a real DAO needs to function.

✅ It makes sure every voter has **one voice** tied to **real stake**.

---

# 🏁 Finalizing a Proposal — Deciding the Fate

Alright —

voting is over.

The community has spoken.

Now the big question:

**Did the proposal pass?**

**Did it fail?**

**Was there enough participation to even make it valid?**

This function, `finalizeProposal()`, is where the DAO checks the final results —

and moves the proposal either toward execution (if it passed) or marks it as failed (if it didn’t).

Let’s walk through it.

---

## 📜 Full Code

```solidity
     
function finalizeProposal(uint256 proposalId) external {
    Proposal storage proposal = proposals[proposalId];
    require(block.timestamp >= proposal.deadline, "Voting period not yet over");
    require(!proposal.executed, "Proposal already executed");
    require(proposal.executionTime == 0, "Execution time already set");

    uint256 totalSupply = governanceToken.totalSupply();
    uint256 totalVotes = proposal.votesFor + proposal.votesAgainst;
    uint256 quorumNeeded = (totalSupply * quorumPercentage) / 100;

    if (totalVotes >= quorumNeeded && proposal.votesFor > proposal.votesAgainst) {
        proposal.executionTime = block.timestamp + timelockDuration;
        emit ProposalTimelockStarted(proposalId, proposal.executionTime);
    } else {
        proposal.executed = true;
        emit ProposalExecuted(proposalId, false);
        if (totalVotes < quorumNeeded) {
            emit QuorumNotMet(proposalId, totalVotes, quorumNeeded);
        }
        // Deposit is NOT refunded here for failed proposals.
    }
}

```

---

# 🔍 Step-by-Step Breakdown

---

### 📜 Step 1: Load the Proposal

```solidity
     
Proposal storage proposal = proposals[proposalId];

```

We pull up the proposal the user wants to finalize.

✅ Now we have access to its full details.

---

### ⏳ Step 2: Make Sure Voting is Over

```solidity
     
require(block.timestamp >= proposal.deadline, "Voting period not yet over");

```

We **must wait** until the official voting period has ended.

✅ This prevents someone from finalizing while votes are still coming in.

---

### 🚫 Step 3: Make Sure It’s Not Already Finalized

```solidity
     
require(!proposal.executed, "Proposal already executed");

```

If the proposal has already been executed or finalized,

we **block** the call to avoid double finalization.

✅ Prevents accidents and replays.

---

### 🕰️ Step 4: Make Sure It Hasn’t Already Entered Timelock

```solidity
     
require(proposal.executionTime == 0, "Execution time already set");

```

If the proposal has already entered the "waiting to execute" phase,

there’s no need to finalize again.

✅ This makes the logic clean and bulletproof.

---

### 📈 Step 5: Calculate Quorum and Total Votes

```solidity
     
uint256 totalSupply = governanceToken.totalSupply();
uint256 totalVotes = proposal.votesFor + proposal.votesAgainst;
uint256 quorumNeeded = (totalSupply * quorumPercentage) / 100;

```

We now check:

- **How many total tokens exist** (totalSupply)
- **How many votes were cast** (totalVotes)
- **How many votes were needed** (quorum)

✅ This is how we verify if enough people participated.

---

### ✅ Step 6A: If Proposal Passed and Quorum Met

```solidity
     
if (totalVotes >= quorumNeeded && proposal.votesFor > proposal.votesAgainst) {
    proposal.executionTime = block.timestamp + timelockDuration;
    emit ProposalTimelockStarted(proposalId, proposal.executionTime);
}

```

- If enough people voted **and** more people voted **for** than **against**...
- Then the proposal **moves into the timelock period**.

The **executionTime** is set to **now + timelockDuration**,

meaning it will be executable only after the waiting period.

✅ This gives the community time to prepare for the change or respond if needed.

✅ We emit an event so that everyone knows when the timelock started and when execution will be allowed.

---

### ❌ Step 6B: If Proposal Failed

```solidity
     
else {
    proposal.executed = true;
    emit ProposalExecuted(proposalId, false);
    if (totalVotes < quorumNeeded) {
        emit QuorumNotMet(proposalId, totalVotes, quorumNeeded);
    }
}

```

If the proposal **failed** (either not enough votes, or more votes against it):

- We immediately mark it as **executed** (no further action will happen).
- Emit an event saying the proposal failed.

If the problem was **not enough voters participating**,

we emit an extra event `QuorumNotMet` to explain exactly why it failed.

✅ This keeps the governance process **transparent and auditable**.

---

# 🌟 Why This Function Matters

This is the critical checkpoint where:

- Community decisions are honored.
- Spam and low-participation proposals are filtered out.
- Winning proposals are **locked** and prepared for safe execution.

✅ Without this step, voting would be meaningless.

✅ This is what turns votes into **real consequences**.

---

# 🏗️ Executing a Proposal — Making the Decision Real

So, the community has voted.

The proposal passed.

The timelock period is over.

Now it's time for the contract to **do what the community decided** —

whether that's transferring funds, calling functions on other contracts, or upgrading parts of the system.

This function, `executeProposal()`, is where the DAO **actually enforces** the will of the voters.

Let’s walk through it.

---

## 📜 Full Code

```solidity
     
function executeProposal(uint256 proposalId) external nonReentrant {
    Proposal storage proposal = proposals[proposalId];
    require(!proposal.executed, "Proposal already executed");
    require(proposal.executionTime > 0 && block.timestamp >= proposal.executionTime, "Timelock not yet expired");

    proposal.executed = true; // set executed early to prevent reentrancy

    bool passed = proposal.votesFor > proposal.votesAgainst;

    if (passed) {
        for (uint256 i = 0; i < proposal.executionTargets.length; i++) {
            (bool success, bytes memory returnData) = proposal.executionTargets[i].call(proposal.executionData[i]);
            require(success, string(returnData));
        }
        emit ProposalExecuted(proposalId, true);
        governanceToken.transfer(proposal.proposer, proposalDepositAmount);
        emit ProposalDepositRefunded(proposal.proposer, proposalDepositAmount);
    } else {
        emit ProposalExecuted(proposalId, false);
        // Deposit is NOT refunded here for failed proposals; it was not refunded in finalizeProposal either.
    }
}

```

---

# 🔍 Step-by-Step Breakdown

---

### 📜 Step 1: Load the Proposal

```solidity
     
Proposal storage proposal = proposals[proposalId];

```

We start by pulling up the proposal data using its ID.

✅ Now we have all the details needed to check if it’s ready for execution.

---

### 🚫 Step 2: Make Sure It Hasn’t Already Been Executed

```solidity
     
require(!proposal.executed, "Proposal already executed");

```

If someone tries to execute a proposal **twice**,

this check **blocks** them immediately.

✅ Double execution = disaster avoided.

---

### ⏳ Step 3: Make Sure the Timelock is Over

```solidity
     
require(proposal.executionTime > 0 && block.timestamp >= proposal.executionTime, "Timelock not yet expired");

```

Two things must be true:

- There **must** be a timelock execution time set (meaning the proposal passed voting).
- The **current time** must be **after** the execution time.

✅ This ensures users can’t rush a proposal through before the community has had time to react.

---

### 🔒 Step 4: Mark Proposal as Executed *Before* Doing Anything Else

```solidity
     
proposal.executed = true; // set executed early to prevent reentrancy

```

This is a **very important security move**.

We **mark the proposal as executed before** calling any external contracts —

so that if something strange happens, the proposal can’t be re-executed or attacked via reentrancy.

✅ This follows Solidity best practices for secure contract design.

---

### ⚖️ Step 5: Check If the Proposal Actually Passed

```solidity
     
bool passed = proposal.votesFor > proposal.votesAgainst;

```

Even though we're here,

we **double-check** whether the community really approved the proposal.

✅ Only passed proposals actually execute actions.

---

### ✅ Step 6A: Execute the Proposal (If Passed)

```solidity
     
if (passed) {
    for (uint256 i = 0; i < proposal.executionTargets.length; i++) {
        (bool success, bytes memory returnData) = proposal.executionTargets[i].call(proposal.executionData[i]);
        require(success, string(returnData));
    }
    emit ProposalExecuted(proposalId, true);
    governanceToken.transfer(proposal.proposer, proposalDepositAmount);
    emit ProposalDepositRefunded(proposal.proposer, proposalDepositAmount);
}

```

If the proposal passed:

- We **loop through** all the target contracts.
- We **call** the corresponding function on each target, using the stored calldata.
- If any call **fails**, the whole execution **reverts** (to stay consistent and safe).

✅ This is where real changes happen — upgrades, transfers, new settings, anything the proposal wanted to do!

After successful execution:

- We **emit an event** to mark the proposal as passed and executed.
- We **refund** the proposer's token deposit (because they contributed a successful idea).

✅ Good proposals are rewarded!

---

### ❌ Step 6B: If the Proposal Failed

```solidity
     
else {
    emit ProposalExecuted(proposalId, false);
}

```

If the proposal failed (rare at this point but still possible),

we simply **emit an event** saying it failed.

✅ No refund of the deposit in this case — it stays in the contract as a penalty for failed ideas.

---

# 🌟 Why This Function Matters

This is the **final step** that turns **votes into real action**.

✅ It honors the decisions made by the community.

✅ It automates the execution — no trusted middleman needed.

✅ It handles reward (refund) for good proposals and punishment for bad ones automatically.

✅ It keeps the entire process **secure**, **transparent**, and **trustless**.

---

# 🧠 Checking Proposal Results — Did It Pass or Fail?

After a proposal has been finalized and executed,

users might want to **check the outcome** without digging through events manually.

This function, `getProposalResult()`, gives a **simple summary**:

whether the proposal **passed**, **failed**, or **didn’t meet quorum**.

Let’s see how it works.

---

## 📜 Full Code

```solidity
     
function getProposalResult(uint256 proposalId) external view returns (string memory) {
    Proposal storage proposal = proposals[proposalId];
    require(proposal.executed, "Proposal not yet executed");

    uint256 totalSupply = governanceToken.totalSupply();
    uint256 totalVotes = proposal.votesFor + proposal.votesAgainst;
    uint256 quorumNeeded = (totalSupply * quorumPercentage) / 100;

    if (totalVotes < quorumNeeded) {
        return "Proposal FAILED - Quorum not met";
    } else if (proposal.votesFor > proposal.votesAgainst) {
        return "Proposal PASSED";
    } else {
        return "Proposal REJECTED";
    }
}

```

---

# 🔍 Step-by-Step Breakdown

---

### 📜 Step 1: Load the Proposal

```solidity
     
Proposal storage proposal = proposals[proposalId];

```

We pull up the proposal details by its ID.

✅ Now we have access to its voting results and deadline info.

---

### ⏳ Step 2: Make Sure It’s Already Executed

```solidity
     
require(proposal.executed, "Proposal not yet executed");

```

We only allow checking the result **after** the proposal is finalized and marked as executed.

✅ This prevents people from asking for results **while voting is still happening**.

---

### 🧮 Step 3: Calculate Total Votes and Quorum Needed

```solidity
     
uint256 totalSupply = governanceToken.totalSupply();
uint256 totalVotes = proposal.votesFor + proposal.votesAgainst;
uint256 quorumNeeded = (totalSupply * quorumPercentage) / 100;

```

- **totalSupply**: How many governance tokens exist overall.
- **totalVotes**: How many votes were actually cast on this proposal.
- **quorumNeeded**: How many votes were needed for the proposal to even be considered valid.

✅ These numbers help determine if the proposal was legitimate.

---

### 🛑 Step 4A: If Not Enough Participation

```solidity
     
if (totalVotes < quorumNeeded) {
    return "Proposal FAILED - Quorum not met";
}

```

If not enough votes were cast (quorum was missed),

the proposal automatically **fails**, even if there were more “for” votes.

✅ Protects against a small group controlling the system unfairly.

---

### ✅ Step 4B: If Passed

```solidity
     
else if (proposal.votesFor > proposal.votesAgainst) {
    return "Proposal PASSED";
}

```

If more people voted **for** than **against**,

and quorum was met, the proposal **passed** successfully.

✅ Ready to execute the actions!

---

### ❌ Step 4C: If Rejected

```solidity
     
else {
    return "Proposal REJECTED";
}

```

If more people voted **against** than **for**,

the proposal was **rejected**.

✅ No changes are made to the system.

---

# 🌟 Why This Function Matters

This function gives:

- 📜 A **simple, readable answer** to "what happened with this proposal?"
- 🚫 Guarantees that only **finished proposals** can be checked.
- ✅ Transparency without needing users to dig into blockchain events manually.

✅ Super helpful for frontends, explorers, and users looking to understand governance outcomes.

---

# 📋 Viewing Full Proposal Details

```solidity
     
function getProposalDetails(uint256 proposalId) external view returns (Proposal memory) {
    return proposals[proposalId];
}

```

---

# 🔍 What’s Happening Here?

This function lets **anyone** look up the **full information** about a specific proposal.

- You pass in the `proposalId`.
- The contract returns the **entire Proposal struct** —
    
    including its description, votes, proposer, targets, execution data, and more.
    

✅ No complicated logic.

✅ No restrictions.

✅ Just a simple, public way to view everything about a proposal.

---

# 🧪 How to Run the Decentralized Governance Contract (Step-by-Step)

---

## ⚙️ Step 1: Deploy a Governance Token

First, we need a **token** that will represent **voting power**.

You can use a simple ERC-20 token for this.

Here’s a **minimal token** you can deploy quickly:

```solidity
     
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts/token/ERC20/ERC20.sol";

contract GovernanceToken is ERC20 {
    constructor() ERC20("GovernanceToken", "GT") {
        _mint(msg.sender, 1000000 * 10 ** decimals()); // Mint 1 million tokens to yourself
    }
}

```

✅ Deploy this `GovernanceToken` contract first.

✅ After deployment, you’ll have a bunch of tokens in your wallet.

---

## 🏗️ Step 2: Deploy the Governance Contract

Now deploy the `DecentralizedGovernance` contract.

**Constructor arguments needed:**

- `governanceToken` address — the address of the `GovernanceToken` you just deployed.
- `votingDuration` — how long a proposal stays open for voting (example: `600` seconds for 10 minutes).
- `timelockDuration` — how long to wait after a proposal passes before it can be executed (example: `300` seconds for 5 minutes).

✅ Deploy it with these values and your governance system is live!

---

## 🔓 Step 3: Approve Governance Contract to Spend Your Tokens

Before you can create proposals, you need to **approve** the governance contract to **spend your tokens** for deposits.

Go to your `GovernanceToken` contract in Remix and call:

```

approve(governanceContractAddress, amount)

```

Where:

- `governanceContractAddress` = address of your DAO contract
- `amount` = enough tokens to cover the `proposalDepositAmount` and maybe voting fees later (example: approve 100 tokens).

✅ Without this step, your proposals will fail because the contract can't pull your tokens!

---

## 🎯 Step 4: Create a Fun Target Contract to Execute Later

Now to make the demonstration **more exciting**,

let’s create a **FunTransfer contract** —

this will let us **move Ether** if a proposal passes!

Here’s a super simple one:

```solidity
     
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract FunTransfer {
    address public owner;
    uint256 public received;

    constructor() {
        owner = msg.sender;
    }

    function receiveEther() external payable {
        received += msg.value;
    }

    function withdrawEther() external {
        payable(owner).transfer(address(this).balance);
    }

    function getBalance() external view returns (uint256) {
        return address(this).balance;
    }
}

```

✅ Deploy this `FunTransfer` contract too.

✅ You can even send it some Ether manually (through Remix UI) so it holds a balance.

---

## 📝 Step 5: Create a Proposal

Now we create a **proposal** that will trigger an action on the `FunTransfer` contract.

You need to:

- **Target** the address of the `FunTransfer` contract.
- **Calldata** to call a function like `withdrawEther()`.

How to get the calldata easily:

- create a new script.js file in remix
- Paste the following code:

```solidity
const { ethers } = require('ethers');

const abi = [
  "function withdrawEther()"
];

const iface = new ethers.utils.Interface(abi);
const data = iface.encodeFunctionData("withdrawEther");

console.log(data);

```

- Run the script
- Copy that bytes output.

Now create a proposal by calling:

```
createProposal(
    "Proposal to withdraw Ether from FunTransfer",
    ["0xd8b8A44ce1fB552f4A4dE0cb074EeBf5A24a2..."],
    ["0x736237.."]
)

```

✅ The governance contract will store your proposal.

✅ It will also deduct your deposit!

---

## 🗳️ Step 6: Vote on the Proposal

Now, to **vote**,

simply call:

```

vote(proposalId, true)  // Vote FOR

```

✅ You’ll need to vote with an address that owns governance tokens.

✅ Remember, the more tokens you have, the bigger your voting weight!

---

## ⏳ Step 7: Wait for Voting to End and Finalize

Wait for the **votingDuration** to expire.

Then call:

```

finalizeProposal(proposalId)

```

✅ This checks if it passed, meets quorum, and schedules it for execution.

✅ It will set an `executionTime` after the `timelockDuration`.

---

## 🛡️ Step 8: Wait for Timelock and Execute!

After the timelock expires, call:

```

executeProposal(proposalId)

```

✅ If the proposal passed,

the governance contract will call the `withdrawEther()` function on `FunTransfer`.

✅ Ether will be transferred to the owner wallet!

✅ The proposal creator will get their deposit refunded too.

---

# 🌟 Quick Summary Flow

> Deploy Governance TokenDeploy Governance ContractApprove spendingDeploy FunTransfer ContractCreate Proposal → Vote → Finalize → Execute
> 

✅ You’ve now simulated **full decentralized decision-making** — with **on-chain execution**!

---

# 🎉 Congratulations!

You didn't just vote —

you **governed**,

you **proposed**,

you **voted**,

and you **executed** a real on-chain action.

You now understand the full flow of **building and running a basic DAO**! 🚀🌿