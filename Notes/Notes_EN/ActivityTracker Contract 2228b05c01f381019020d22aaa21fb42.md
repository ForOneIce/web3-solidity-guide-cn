# ActivityTracker Contract

Alright! So far, we’ve built calculators, shared wallets, piggy banks, even a TipJar that could accept USD and magically convert it to ETH.

But what if we shift gears a bit?

Let’s say you and your friends are all on a fitness journey. You’re logging your workouts, tracking progress, celebrating milestones like “10 runs done!” or “100km covered!” — and you want it all to be transparent and provable on-chain.

Not just data for the sake of it — but data that can trigger live updates on a frontend, unlock rewards, or even mint NFTs when you reach certain goals.

**That’s where Solidity events come into play.**

Today, we’re building a fun little contract called...

---

```solidity
 
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract SimpleFitnessTracker {

```

Yup, the **SimpleFitnessTracker** — but don’t let the name fool you. While the contract is small, it teaches you one of the most powerful features in all of Solidity: **events.**

<aside>
💻

Here’s the complete code 👇🏼

[https://github.com/snehasharma76/30daysSolidity_Web3Compass/blob/master/ActivityTracker.sol](https://github.com/snehasharma76/30daysSolidity_Web3Compass/blob/master/ActivityTracker.sol)

</aside>

---

### 

### What Are Events?

Events in Solidity are like your smart contract's way of grabbing a megaphone and saying, “Hey, something just happened!”

They don’t change anything on-chain — instead, they **emit logs** that can be picked up by your frontend or any external system listening in.

Imagine Alice just crushed her 10th workout. The contract emits an event saying, “Alice hit 10 workouts!” Your frontend catches that signal and boom — it can throw confetti, unlock an achievement, or trigger a badge animation.

So yeah, events may just look like a boring `emit` line in your code… but behind the scenes, they’re the secret sauce that makes your dApp feel *alive*.

Alright, time to set up the data behind the magic.

---

### State Variables & Structs — Laying the Groundwork

---

### 

Before we dive into the actual logic, let’s set up the data structures our contract will rely on.

We’re building a fitness tracker, so we need to store information about users and their workouts. You can always expand this later to include things like goals, achievements, or social features — but for now, we’re keeping it simple and focused.

Let’s start with two key `structs`:

### The `UserProfile` Struct

```solidity
    
struct UserProfile {
    string name;
    uint256 weight; // in kg
    bool isRegistered;
}

```

This struct holds basic user info:

- Their name
- Their weight (in kilograms)
- A flag that tells us if they’ve registered or not

Every address that signs up will get one of these profiles stored on-chain.

### The `WorkoutActivity` Struct

```solidity
    
struct WorkoutActivity {
    string activityType;
    uint256 duration; // in seconds
    uint256 distance; // in meters
    uint256 timestamp;
}

```

This struct captures the details of each workout:

- The type of activity (running, cycling, swimming, etc.)
- How long it lasted (in seconds)
- How far the user went (in meters)
- When it happened (`block.timestamp`)

Each time a user logs a workout, we’ll create one of these and add it to their workout history.

---

### Mapping It All Together

Now that we have our data structures, let’s wire them up using mappings:

```solidity
    
mapping(address => UserProfile) public userProfiles;
mapping(address => WorkoutActivity[]) private workoutHistory;
mapping(address => uint256) public totalWorkouts;
mapping(address => uint256) public totalDistance;

```

Here’s what each one does:

- `userProfiles`: Stores a profile for each user (by their address)
- `workoutHistory`: Keeps an array of workout logs per user
- `totalWorkouts`: Tracks how many workouts each user has logged
- `totalDistance`: Tracks the total distance a user has covered

With this setup, we’re ready to build out the core features — user registration, logging workouts, tracking progress, and celebrating milestones.

Not bad for just a few lines of setup, right?

### Declaring Events — This Is the Cool Part

Let’s declare some events that will act like signals our frontend can listen to:

```solidity
   
event UserRegistered(address indexed userAddress, string name, uint256 timestamp);
event ProfileUpdated(address indexed userAddress, uint256 newWeight, uint256 timestamp);
event WorkoutLogged(address indexed userAddress, string activityType, uint256 duration, uint256 distance, uint256 timestamp);
event MilestoneAchieved(address indexed userAddress, string milestone, uint256 timestamp);

```

Let’s break this down step by step:

### What’s an Event, Technically?

An `event` in Solidity is like defining a custom log format. When something important happens in your contract, you can `emit` one of these events, and it’ll be recorded in the transaction logs.

Events don’t affect your contract’s state — they’re just a way for the contract to say, “Hey, something just happened,” and send out the relevant details.

These logs can then be picked up by your frontend to display messages, update the UI, or trigger actions in real time.

### Understanding the Parameters

Take this event for example:

```solidity
  
event WorkoutLogged(
  address indexed userAddress,
  string activityType,
  uint256 duration,
  uint256 distance,
  uint256 timestamp
);

```

Here’s what each parameter means:

- `userAddress`: Who did the workout.
- `activityType`: What kind of activity they did — running, cycling, etc.
- `duration`: How long the workout lasted, in seconds.
- `distance`: How far they went, in meters.
- `timestamp`: When it happened, captured using `block.timestamp`.

Each of our events follows a similar pattern — they log the who, what, and when — giving your frontend everything it needs to react instantly.

### Why Use `indexed`?

You’ll notice that each event includes `address indexed userAddress`. So what’s the deal with `indexed`?

When you mark a parameter as `indexed`, you’re making it searchable. This means you can filter logs in your frontend based on that specific value.

For instance, if you want to show only Alice’s events on her profile page, your frontend can query the logs and pull just the ones where `userAddress == Alice`.

Without `indexed`, you’d have to scan every single event log and manually check each one — which is inefficient and slow.

With `indexed`, it’s optimized and scalable — exactly what you want in a dApp.

One thing to note: you can only index up to **three** parameters in a single event. So use them wisely.

---

### Modifiers

```solidity
 
modifier onlyRegistered() {
    require(userProfiles[msg.sender].isRegistered, "User not registered");
    _;
}

```

This modifier is a simple check — it ensures the caller has already registered. We’ll reuse it in multiple functions to keep things clean.

---

### 

### `registerUser()` — Let the Gains Begin

```solidity

function registerUser(string memory _name, uint256 _weight) public {
    require(!userProfiles[msg.sender].isRegistered, "User already registered");

    userProfiles[msg.sender] = UserProfile({
        name: _name,
        weight: _weight,
        isRegistered: true
    });

    emit UserRegistered(msg.sender, _name, block.timestamp);
}

```

This is the moment when someone officially joins the fitness squad.

First, we make sure they haven’t registered already using a `require` check. Then, we save their name and weight in our `userProfiles` mapping and set the `isRegistered` flag to true.

Now comes the cool part — the event.

```solidity

emit UserRegistered(msg.sender, _name, block.timestamp);

```

This line tells the blockchain: “Hey, someone just registered — here’s who, and here’s when.”

Let’s tie it back to the event declaration:

```solidity

event UserRegistered(address indexed userAddress, string name, uint256 timestamp);

```

When we emit the event, we pass in all the required values — in the same order as declared: the sender’s address, their name, and the current timestamp. These values get packaged into the log and are instantly available for the frontend to pick up and respond to.

Your frontend might use this event to:

- Display a welcome popup
- Trigger a badge or animation
- Store the event in an off-chain database
- Track analytics like “total signups today”

### But Wait — Doesn’t This Cost Gas?

Good question. Yes, events do cost a little gas to emit (because logs are written to the blockchain), but **they’re much cheaper than storing data on-chain**.

In fact, emitting an event is one of the most gas-efficient ways to expose data from your smart contract. That’s why we often use them for frontend updates, analytics, and anything that doesn’t need to be stored permanently in state.

So go ahead — emit proudly. It’s cheap, fast, and makes your dApp feel dynamic.

---

### 

### `updateWeight()` — Progress, Not Perfection

```solidity
function updateWeight(uint256 _newWeight) public onlyRegistered {
    UserProfile storage profile = userProfiles[msg.sender];

    if (_newWeight < profile.weight && (profile.weight - _newWeight) * 100 / profile.weight >= 5) {
        emit MilestoneAchieved(msg.sender, "Weight Goal Reached", block.timestamp);
    }

    profile.weight = _newWeight;
    emit ProfileUpdated(msg.sender, _newWeight, block.timestamp);
}

```

This function allows a registered user to update their weight — but it’s not just a simple setter. There’s some smart logic packed in here to reward meaningful progress.

Let’s unpack it:

### Step-by-Step Breakdown

1. **Access the User’s Profile**
    
    
    ```solidity
    
    UserProfile storage profile = userProfiles[msg.sender];
    
    ```
    
    In this line, we’re creating a **reference** to the user’s profile stored on the blockchain.
    
    By using the `storage` keyword, we’re saying:
    
    “Hey Solidity, point directly to the data that already exists in the contract’s storage — don’t make a copy of it.”
    
    This is important because we want to **modify the actual profile** that lives on-chain. If we had used `memory` instead, we’d only be working with a temporary copy — and any changes we make would be discarded at the end of the function.
    
    So by using `storage`, we make sure that when we update something like the user’s weight, the update is permanent and reflected in the global `userProfiles` mapping.
    
2. **Check for a Weight Milestone**
    
    Before we update the weight, we check if the new weight is at least **5% less** than the current one.
    
    ```solidity
    
    if (_newWeight < profile.weight && (profile.weight - _newWeight) * 100 / profile.weight >= 5)
    
    ```
    
    This ensures we only trigger a milestone for significant progress — not minor fluctuations. If the condition is true, we emit this:
    
    ```solidity
    
    emit MilestoneAchieved(msg.sender, "Weight Goal Reached", block.timestamp);
    
    ```
    
    This event tells the frontend: “This user just hit a major goal!” — perfect for triggering badges, sound effects, or unlocking a reward.
    
3. **Update the User’s Weight**
    
    After the milestone check, we go ahead and save the new weight:
    
    ```solidity
    
    profile.weight = _newWeight;
    
    ```
    
4. **Emit the Profile Update**
    
    Finally, we let the outside world know the user’s weight has changed:
    
    ```solidity
    
    emit ProfileUpdated(msg.sender, _newWeight, block.timestamp);
    
    ```
    
    This helps the frontend refresh user stats or recalculate progress bars.
    

---

### 

### `logWorkout()` — Tracking Every Rep, Run, and Ride

```solidity
   
function logWorkout(
    string memory _activityType,
    uint256 _duration,
    uint256 _distance
) public onlyRegistered {
    // Create new workout activity
    WorkoutActivity memory newWorkout = WorkoutActivity({
        activityType: _activityType,
        duration: _duration,
        distance: _distance,
        timestamp: block.timestamp
    });

    // Add to user's workout history
    workoutHistory[msg.sender].push(newWorkout);

    // Update total stats
    totalWorkouts[msg.sender]++;
    totalDistance[msg.sender] += _distance;

    // Emit workout logged event
    emit WorkoutLogged(
        msg.sender,
        _activityType,
        _duration,
        _distance,
        block.timestamp
    );

    // Check for workout count milestones
    if (totalWorkouts[msg.sender] == 10) {
        emit MilestoneAchieved(msg.sender, "10 Workouts Completed", block.timestamp);
    } else if (totalWorkouts[msg.sender] == 50) {
        emit MilestoneAchieved(msg.sender, "50 Workouts Completed", block.timestamp);
    }

    // Check for distance milestones
    if (totalDistance[msg.sender] >= 100000 && totalDistance[msg.sender] - _distance < 100000) {
        emit MilestoneAchieved(msg.sender, "100K Total Distance", block.timestamp);
    }
}

```

This is the **core engine** of the fitness tracker — it handles everything that happens when a user logs a new workout.

Let’s walk through it step by step:

---

### 1. Accept Workout Details as Input

The function takes three parameters:

- `_activityType`: What kind of workout the user did (e.g., "Running", "Swimming").
- `_duration`: How long the workout lasted, in seconds.
- `_distance`: How far the user went, in meters.

The function is `public` and restricted by the `onlyRegistered` modifier, so **only registered users** can log workouts.

---

### 2. Record the Workout in a Struct

We create a new `WorkoutActivity` instance using the input data and the current block’s timestamp:

```solidity
   
WorkoutActivity memory newWorkout = WorkoutActivity({
    activityType: _activityType,
    duration: _duration,
    distance: _distance,
    timestamp: block.timestamp
});

```

This `newWorkout` object holds all the important info:

- What the user did (`activityType`)
- How long it lasted (`duration`)
- How far they went (`distance`)
- When it happened (`timestamp`)

But here’s something important: we declared the struct using the `memory` keyword. Why?

---

### Why `memory`?

In Solidity, you have two main data locations: `storage` and `memory`.

- `storage` is persistent — it lives on the blockchain and costs gas to read/write.
- `memory` is temporary — it exists only during the function call and is much cheaper.

Since we’re just **creating this struct temporarily** to push it into an array (`workoutHistory[msg.sender]`), we don’t need to store it permanently as a standalone variable. Using `memory` makes this operation **more efficient and gas-friendly**.

Once it’s pushed into the array (which lives in storage), the data is saved where it needs to be — and the temporary `newWorkout` instance can disappear.

So: temporary data → use `memory`. Persistent data → use `storage`.

Efficient, clean, and exactly what we need in this case.

---

### 3. Store the Workout in the User’s History

The new workout is then added to the user’s workout history — which is stored as a dynamic array in the `workoutHistory` mapping:

```solidity
   
workoutHistory[msg.sender].push(newWorkout);

```

This means each user has a complete personal log of every workout they’ve ever recorded.

---

### 4. Update Aggregate Stats

Next, we keep a running total of:

- How many workouts the user has done
- How far they’ve traveled in total (cumulative distance)

```solidity
   
totalWorkouts[msg.sender]++;
totalDistance[msg.sender] += _distance;

```

These values are useful for leaderboard-style features, progress tracking, and milestone detection.

---

### 5. Emit the `WorkoutLogged` Event

Now that the data is stored, we let the frontend know something just happened:

```solidity
   
emit WorkoutLogged(
    msg.sender,
    _activityType,
    _duration,
    _distance,
    block.timestamp
);

```

This event provides all the info the frontend needs to display the new activity, update graphs, animate progress bars, or trigger real-time feedback like sounds or messages.

---

### 6. Detect and Celebrate Milestones

Here’s where we make the app feel **alive**.

We check whether the user hit a key milestone — like completing their 10th or 50th workout — and emit a `MilestoneAchieved` event when they do:

```solidity
   
if (totalWorkouts[msg.sender] == 10) {
    emit MilestoneAchieved(msg.sender, "10 Workouts Completed", block.timestamp);
} else if (totalWorkouts[msg.sender] == 50) {
    emit MilestoneAchieved(msg.sender, "50 Workouts Completed", block.timestamp);
}

```

We also check if the user **crossed 100K kilometers total distance**:

```solidity
   
if (totalDistance[msg.sender] >= 100000 && totalDistance[msg.sender] - _distance < 100000) {
    emit MilestoneAchieved(msg.sender, "100K Total Distance", block.timestamp);
}

```

Notice how we compare the **new total** with the **previous total (by subtracting the current distance)** — this ensures we only fire the milestone **once**, right when the user crosses the threshold

---

### getUserWorkoutCount()

```solidity
 
function getUserWorkoutCount() public view onlyRegistered returns (uint256) {
    return workoutHistory[msg.sender].length;
}

```

And finally, a handy read-only function. This tells a user how many workouts they’ve logged so far.

Perfect for dashboards or stats displays.

---

### Wrapping Up

So there you have it — a simple but super powerful contract.

Yes, the implementation looks short. But the **events** are doing all the heavy lifting behind the scenes.

Every action is being broadcast to the outside world — making this contract feel alive.

And that’s the real magic.

Because when smart contracts and frontends work together like this — you’re not just storing data… you’re telling stories.

Workout by workout. Step by step.