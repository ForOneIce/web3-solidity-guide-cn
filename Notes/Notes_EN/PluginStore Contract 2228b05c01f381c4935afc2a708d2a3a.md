# PluginStore Contract

Alright, imagine you're building a Web3 game — something with lore, battles, character progression, maybe even trading. Naturally, every player has a **profile** — their name, avatar, equipped weapons, achievements, social links, battle logs… the works.

Now here's the catch:

If you try to cram all of that into a single smart contract, things get bloated fast. You’ll hit storage limits, upgrade issues, and you'll lose flexibility.

So what if we **modularized** the whole thing?

Instead of one gigantic contract, we build a **lightweight core profile contract** that just stores name and avatar...

...and let users "install" optional **feature plugins** — little smart contracts that handle specific things like achievements, inventory, battle stats, or social activity.

Like **mods for your player**. Plug and play. On-chain.

But for that to work, the core contract needs to call those plugin contracts somehow. That’s where the magic comes in:

> Welcome to the world of calls in Solidity — your gateway to modularity, upgradability, and smart contract flexibility.
> 

---

## 🧩 Let’s Talk Calls: `call`, `delegatecall`, and `staticcall`

Solidity gives us a few different ways to talk to other contracts. And understanding these will help us see how this system ticks.

---

### 🔁 `call`

The most common one.

- You're telling **another contract** to do something.
- That contract uses **its own state** and **its own storage**.
- Usually used for interacting with external systems (like an ERC-20 token).

📦 *Example:* “Hey Token contract, transfer 100 tokens to Bob.”

---

### 🧠 `delegatecall`

This is the clever one.

- You’re **borrowing logic** from another contract…
- But you’re running it in **your contract’s storage context**.
- Meaning the data lives in your contract, but the logic comes from somewhere else.

🎮 *Example:* “Hey contract, use your function to store this data — but store it in my storage, not yours.”

This is how we build  upgradable contracts, or proxy patterns, etc

---

### 🔍 `staticcall`

Just like `call`, but read-only.

- No storage changes allowed.
- Great for view or pure functions.

📖 *Example:* “Hey contract, just show me what is stored in the contract”

---

## 🎮 Game Plan: Building Our Modular Player System

Here’s what we’re building:

At the center of our game world is the **PluginStore** — the core contract that stores each player’s **profile**: just their name and avatar. Simple, clean, focused.

But players in this world want more than just a name and a face.

They want achievements. Weapons. Badges. Maybe even a clan system.

So instead of bloating the core contract with all those features, we let players **attach feature modules** — what we call **plugins**.

Each plugin is a separate contract that handles a specific feature. And since plugins are modular, we can upgrade, replace, or add new ones at any time — no need to redeploy the entire system.

---

## 🧩 Our Plugin Arsenal

To start, we’ll build:

- 🏆 **`AchievementsPlugin`** – stores a player’s most recent achievement
- ⚔️ **`WeaponStorePlugin`** – stores the weapon a player has equipped

These plugins don’t run in isolation — the **PluginStore** calls them dynamically:

- For state-changing actions (like setting a weapon), we use **`call`**
- For read-only queries (like checking a player's achievement), we use **`staticcall`**

And if we wanted to share storage (like with an upgradeable system), we could even swap in **`delegatecall`** — but for now, `call` and `staticcall` keep things simple and secure.

---

# 🔌 `PluginStore` – The Core of Our Modular Game System

This contract defines the **player profile hub** — every player has a basic profile (name and avatar), and they can connect additional functionality through plugins.

Let’s dive in line by line and explain what’s going on.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract PluginStore {
    struct PlayerProfile {
        string name;
        string avatar;
    }

    mapping(address => PlayerProfile) public profiles;

    // === Multi-plugin support ===
    mapping(string => address) public plugins;

    // ========== Core Profile Logic ==========

    function setProfile(string memory _name, string memory _avatar) external {
        profiles[msg.sender] = PlayerProfile(_name, _avatar);
    }

    function getProfile(address user) external view returns (string memory, string memory) {
        PlayerProfile memory profile = profiles[user];
        return (profile.name, profile.avatar);
    }

    // ========== Plugin Management ==========

    function registerPlugin(string memory key, address pluginAddress) external {
        plugins[key] = pluginAddress;
    }

    function getPlugin(string memory key) external view returns (address) {
        return plugins[key];
    }

    // ========== Plugin Execution ==========
function runPlugin(
    string memory key,
    string memory functionSignature,
    address user,
    string memory argument
) external {
    address plugin = plugins[key];
    require(plugin != address(0), "Plugin not registered");

    bytes memory data = abi.encodeWithSignature(functionSignature, user, argument);
    (bool success, ) = plugin.call(data);
    require(success, "Plugin execution failed");
}

function runPluginView(
    string memory key,
    string memory functionSignature,
    address user
) external view returns (string memory) {
    address plugin = plugins[key];
    require(plugin != address(0), "Plugin not registered");

    bytes memory data = abi.encodeWithSignature(functionSignature, user);
    (bool success, bytes memory result) = plugin.staticcall(data);
    require(success, "Plugin view call failed");

    return abi.decode(result, (string));
}

}

```

---

### 🧱 Contract Setup

```solidity
  
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

```

Standard license and compiler version. We’re using Solidity `0.8.x`, which comes with built-in overflow checks.

---

## 👤 Player Profiles

```solidity
  
struct PlayerProfile {
    string name;
    string avatar;
}

```

We define a struct to store each player’s **name** and **avatar**. This keeps the basic profile lightweight and focused.

---

```solidity
  
mapping(address => PlayerProfile) public profiles;

```

This mapping connects each Ethereum address (player) to their `PlayerProfile`.

- `profiles[msg.sender] = ...` will store a profile for the caller.
- Since it’s marked `public`, Solidity auto-generates a getter.

---

## 🧩 Plugin Registry

```solidity
  
mapping(string => address) public plugins;

```

We use this mapping to **register plugins** by a string key (like `"achievements"` or `"weapons"`) and map them to deployed contract addresses.

This is the plugin directory. Every plugin must be registered before it can be used.

---

## 

### `setProfile`

```solidity
  
function setProfile(string memory _name, string memory _avatar) external {
    profiles[msg.sender] = PlayerProfile(_name, _avatar);
}

```

### ✅ What it does:

Lets a player **create or update** their profile by setting a name and avatar.

### 🔍 Line-by-line:

- `function setProfile(...)`: This function is **publicly callable**, so any player can set their own profile.
- `msg.sender`: This is the address of the person calling the function — we use it as the key to store their profile.
- `PlayerProfile(_name, _avatar)`: We're creating a struct instance directly inline.
- `profiles[msg.sender] = ...`: We're assigning the struct to the caller’s address in the mapping. This writes to contract storage.

---

### `getProfile`

```solidity
  
function getProfile(address user) external view returns (string memory, string memory) {
    PlayerProfile memory profile = profiles[user];
    return (profile.name, profile.avatar);
}

```

### ✅ What it does:

Allows **anyone** to look up a player’s profile using their wallet address.

### 🔍 Line-by-line:

- `external view`: This function doesn’t change state — it’s read-only.
- `PlayerProfile memory profile = ...`: We fetch the profile from storage and copy it to memory.
- `return (...)`: Returns the name and avatar for display in a frontend or UI.

---

## 🧩 Plugin Management Functions

---

### `registerPlugin`

```solidity
  
function registerPlugin(string memory key, address pluginAddress) external {
    plugins[key] = pluginAddress;
}

```

### ✅ What it does:

Registers a plugin contract with a human-readable key like `"achievements"` or `"weapons"`.

### 🔍 Line-by-line:

- `string memory key`: The name used to identify the plugin — think of this like a URL slug or plugin name.
- `address pluginAddress`: The smart contract address for the plugin you’re registering.
- `plugins[key] = pluginAddress`: Adds the plugin to the system so it can be used later.

📌 This makes your plugin *discoverable* by the core contract.

---

### `getPlugin`

```solidity
  
function getPlugin(string memory key) external view returns (address) {
    return plugins[key];
}

```

### ✅ What it does:

Returns the address of a plugin using its key.

Useful for UI tools or to verify whether a plugin is already registered.

---

## ⚙️ Plugin Execution Functions

---

### `runPlugin`

```solidity
  
function runPlugin(
    string memory key,
    string memory functionSignature,
    address user,
    string memory argument
) external {
    address plugin = plugins[key];
    require(plugin != address(0), "Plugin not registered");

    bytes memory data = abi.encodeWithSignature(functionSignature, user, argument);
    (bool success, ) = plugin.call(data);
    require(success, "Plugin execution failed");
}

```

### ✅ What it does:

Sends a **state-changing function call** to the plugin contract.

This is where the real flexibility happens — it lets you call *any* function on the plugin using its signature.

### 🔍 Line-by-line:

1. `address plugin = plugins[key];`
    - We retrieve the plugin address using the key like `"achievements"`.
2. `require(plugin != address(0), ...)`
    - Makes sure the plugin was actually registered — otherwise we abort.
3. `abi.encodeWithSignature(...)`
    - This is the trickiest part:
        - We're building a low-level function call from a string.
        - `functionSignature` looks like: `"setAchievement(address,string)"`
        - We combine it with the provided arguments (`user`, `argument`) to build raw bytecode.
4. `plugin.call(data)`
    - Low-level `call` sends the request to the plugin contract.
    - The plugin executes in **its own storage context**, **not** the PluginStore's.
5. `require(success, ...)`
    - If the plugin failed (due to invalid args or logic errors), the whole transaction reverts.

📦 This allows **plugin methods that write to state** — like updating a player’s weapon or achievement.

---

### `runPluginView`

```solidity
  
function runPluginView(
    string memory key,
    string memory functionSignature,
    address user
) external view returns (string memory) {
    address plugin = plugins[key];
    require(plugin != address(0), "Plugin not registered");

    bytes memory data = abi.encodeWithSignature(functionSignature, user);
    (bool success, bytes memory result) = plugin.staticcall(data);
    require(success, "Plugin view call failed");

    return abi.decode(result, (string));
}

```

### ✅ What it does:

Calls a **read-only function** on the plugin contract and returns the result.

### 🔍 Line-by-line:

1. `address plugin = plugins[key];`
    - Fetches the plugin just like before.
2. `abi.encodeWithSignature(...)`
    - Prepares the data for the function call using just the user’s address.
3. `plugin.staticcall(data)`
    - Unlike `call`, `staticcall` **cannot change state** — it’s read-only.
    - Perfect for functions like `getWeapon()` or `getAchievement()`.
4. `require(success, ...)`
    - If the plugin fails (e.g., invalid address or wrong signature), the call is reverted.
5. `abi.decode(...)`
    - Converts the returned bytes into a string so we can return it to the caller.

📌 This function is amazing for **frontend apps** that want to **fetch plugin data** efficiently without risk.

## 🏆 AchievementsPlugin — Add-On Logic for Player Milestones

This contract is a **plugin** designed to store each player’s **latest unlocked achievement** — things like `"First Blood"`, `"Master Collector"`, or `"Top 1%"`.

It’s meant to be used *by* the `PluginStore` contract through a function like `runPlugin(...)`. That’s why the logic is simple and focused — it’s an **isolated module** responsible for one thing: **tracking achievements**.

Let’s break it down.

---

### ✅ Full Contract (For Context)

```solidity
  
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;
contract AchievementsPlugin {
    // user => achievement string
    mapping(address => string) public latestAchievement;

    // Set achievement for a user (called by PluginStore)
    function setAchievement(address user, string memory achievement) public {
        latestAchievement[user] = achievement;
    }

    // Get achievement for a user
    function getAchievement(address user) public view returns (string memory) {
        return latestAchievement[user];
    }
}

```

---

### 📦 State Variable

```solidity
  
mapping(address => string) public latestAchievement;

```

### ✅ What it does:

Keeps track of the most recent achievement unlocked by each player.

- The `address` is the player’s wallet.
- The `string` is the name of the achievement (e.g. `"First Kill"` or `"Treasure Hunter"`).
- It’s marked `public`, which means Solidity automatically creates a getter function for free:
    
    → `latestAchievement(address) → string`
    

### Why it's structured this way:

This is a **simple and direct way** to associate a string with each player address.

---

### 🛠️ `setAchievement`

```solidity
  
function setAchievement(address user, string memory achievement) public {
    latestAchievement[user] = achievement;
}

```

### ✅ What it does:

Updates the latest achievement string for a specific user.

### 🔍 Line-by-line:

- `function setAchievement(...)`: This is the main setter function that modifies state.
- `address user`: The player whose achievement is being updated. This is passed in manually (rather than using `msg.sender`) because the **PluginStore** is the one calling this, on behalf of the player.
- `string memory achievement`: The name of the achievement to store.
- `latestAchievement[user] = achievement;`: Updates the mapping. Simple assignment.

### 🧠 Why it’s written like this:

- **The PluginStore handles access control** — this function is intentionally left open so the plugin can be reused anywhere.
- We use a standard setter pattern so the PluginStore can delegate calls to it.

---

### 🔍 `getAchievement`

```solidity
  
function getAchievement(address user) public view returns (string memory) {
    return latestAchievement[user];
}

```

### ✅ What it does:

Fetches the latest achievement unlocked by a specific user.

### 🔍 Line-by-line:

- `public view`: This is a **read-only** function.
- `returns (string memory)`: It returns the string value from the mapping.
- `latestAchievement[user]`: Simple lookup from the mapping.

### Why use a custom getter if there’s a public one?

This gives more flexibility — for example, you could later:

- Add formatting
- Combine with metadata
- Return multiple achievements

So this getter is explicitly defined for future-proofing and clarity.

## ⚔️ WeaponStorePlugin — Track Equipped Weapons in a Modular Way

In many games, each player can carry or equip a weapon — like a sword, bow, laser gun, or some custom item. This plugin handles exactly that.

Just like `AchievementsPlugin`, this contract is designed to be used **via the main `PluginStore` contract**. It stores each player's **currently equipped weapon** and lets the core contract **set** and **fetch** that weapon.

Let’s walk through the code and explain it piece by piece.

---

### 🔍 Full Contract

```solidity
  
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

/**
 * @title WeaponStorePlugin
 * @dev Stores and retrieves a user's equipped weapon. Meant to be called via PluginStore.
 */
contract WeaponStorePlugin {
    // user => weapon name
    mapping(address => string) public equippedWeapon;

    // Set the user's current weapon (called via PluginStore)
    function setWeapon(address user, string memory weapon) public {
        equippedWeapon[user] = weapon;
    }

    // Get the user's current weapon
    function getWeapon(address user) public view returns (string memory) {
        return equippedWeapon[user];
    }
}

```

---

### 🗃️ State Variable

```solidity
  
mapping(address => string) public equippedWeapon;

```

### ✅ What it does:

- Keeps track of which weapon each player has equipped.
- The `address` represents the **player**.
- The `string` stores the **weapon name** — like `"Flaming Sword"` or `"Golden AK"`.

Because it’s marked `public`, Solidity auto-generates a getter:

```solidity
  
function equippedWeapon(address) external view returns (string memory);

```

So this plugin already exposes the weapon info without writing a custom getter — though we’ll see in a second why a custom one still exists.

---

### 🛠️ `setWeapon`

```solidity
  
function setWeapon(address user, string memory weapon) public {
    equippedWeapon[user] = weapon;
}

```

### ✅ What it does:

This function lets us update the currently equipped weapon for a player.

### 🔍 Line-by-line:

- `function setWeapon(...)`: Standard setter for weapon assignment.
- `address user`: The player whose weapon is being updated.
    
    This is **manually passed in** because the `PluginStore` is calling this on behalf of the player. We’re not using `msg.sender`.
    
- `string memory weapon`: The name of the new weapon.
- `equippedWeapon[user] = weapon;`: Updates the mapping.

### 🧠 Why this setup?

- It keeps plugin logic **decoupled** from how access control is handled.
    
    The PluginStore is expected to validate who is allowed to call this.
    
- This function can be reused across different contexts or contracts.

---

### 🔍 `getWeapon`

```solidity
  
function getWeapon(address user) public view returns (string memory) {
    return equippedWeapon[user];
}

```

### ✅ What it does:

Fetches the currently equipped weapon for a user.

### 🔍 Line-by-line:

- `public view`: It's a **read-only** function.
- `returns (string memory)`: It returns the weapon string.
- `equippedWeapon[user]`: Simple lookup from the mapping.

Even though a public getter exists because of the mapping declaration, writing this function explicitly is helpful for:

- Semantic clarity
- Future-proofing (e.g. maybe we want to later format the name or fetch metadata)

---

### 🔌 How It Fits Into the Plugin System

Let’s say the `PluginStore` has this plugin registered under the key `"weapon"`.

Now if a player wants to equip a new weapon:

```solidity
  
pluginStore.runPlugin(
  "weapon",
  "setWeapon(address,string)",
  msg.sender,
  "Golden Axe"
);

```

And if we want to know what weapon they’re using:

```solidity
  
pluginStore.runPluginView(
  "weapon",
  "getWeapon(address)",
  userAddress
);

```

Just like installing a mod in your game, this plugin becomes a **reusable piece of game logic**.

## 🧩 Wrapping Up: A Flexible Future for On-Chain Games

What we’ve just built is more than just a few smart contracts — it’s a **pattern**.

A pattern for building modular, extensible systems on-chain — where the core stays simple, and the features plug in as needed.

Let’s recap what we learned:

### ✅ The Core Idea

We created a `PluginStore` contract that acts as the **hub** for player profiles. It stores the basics (like name and avatar), but offloads specialized features — like achievements and weapons — to **plugin contracts**.

Each plugin contract implements its own logic and data structures, but the main contract can talk to them using:

- `call` — to trigger state changes in external contracts
- `delegatecall` — when we want plugin logic but to **store data inside the main contract**
- `staticcall` — for efficient, read-only queries

This separation of concerns gives us **modularity**, **upgradeability**, and **maintainability** — which are huge advantages for complex Web3 games.

---

### 🔌 Why This Matters

In a traditional setup, every feature (inventory, achievements, friends, etc.) would bloat a single contract. Any upgrade would require redeployment, state migration, and headaches.

But with this plugin model, we can:

- Add new features as new contracts
- Upgrade plugins without touching the core
- Keep storage lean and logic clean
- Let players customize their experience

### 🎮 Next Steps You Can Explore

If you want to extend this system, try building:

- A **FriendsPlugin** for managing player connections
- A **BattleLogPlugin** to track combat history
- A **TokenInventoryPlugin** to track ERC-20 or NFT ownership

And if you're feeling adventurous — try swapping `call` for `delegatecall` and keep all storage in `PluginStore`. But be careful: with great power comes great responsibility (and tricky storage layout risks).