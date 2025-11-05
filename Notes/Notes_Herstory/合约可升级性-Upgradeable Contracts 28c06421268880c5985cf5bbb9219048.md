# 合约可升级性-Upgradeable Contracts

Day: Day 17
ID: 17
译者: unisy
难度等级: 中级

![deepseek_mermaid_20251018_4fe96b.svg](%E5%90%88%E7%BA%A6%E5%8F%AF%E5%8D%87%E7%BA%A7%E6%80%A7-Upgradeable%20Contracts%2028c06421268880c5985cf5bbb9219048/deepseek_mermaid_20251018_4fe96b.svg)

想象一下，你已经将一个智能合约推向市场。它运行顺畅，用户纷纷订阅，ETH不断流入……然后，突然，砰地一声💥——你发现了一个漏洞。或者你想增加新功能。又或者你意识到自己的逻辑完全可以更高效。

但问题来了：

> 智能合约一旦部署，就无法更改。
> 

至少传统意义上不行。

在Web2中，如果你修复漏洞或添加功能，只需将更新推送到服务器即可。但在Web3呢？智能合约是不可更改的。一旦部署，就被锁定。

除非……你将它设计成**可升级的**。

---

## 🔄 可升级的真正含义？

当我们说**可升级合约**时，我们指的是将**存储**与**逻辑**分离。

其思想是：

-你部署一个存储**数据**的合约——我们称之为**代理（proxy）**。

-你部署另一个包含**逻辑**的合约——这是实际的代码。

-代理使用 `delegatecall` 来执行外部合约的逻辑——但使用的是**它自己的存储**。

所以，如果你需要改变行为，你不必动代理——你只需将它指向一个新的逻辑合约。所有数据都保持安全。

很酷，对吧？

---

## 🧰 我们将构建什么：一个可升级的订阅系统

让我们通过构建一个**模块化订阅管理器**（你可以在 SaaS 应用或 dApp 中使用的那种）来将这个想法变为现实。

Here’s the game plan:

### 1. **`SubscriptionStorageLayout.sol`**

这是蓝图。它定义了：

- 谁是所有者
- 逻辑合约的地址在哪里
- 实际的存储布局：用户订阅、套餐价格、持续时间等。

可以把它想象成代理和逻辑合约都能理解的共享大脑。

---

### 2. **`SubscriptionStorage.sol`**

这是**代理合约**：

- 它拥有数据它拥有数据
- 它通过 `delegatecall` 将所有逻辑委托给外部合约执行
- 它可以随时升级到新的逻辑合约

所以用户与此合约交互——但在幕后，它只是将用户的调用转发给它当前指向的任何逻辑合约。

---

### 3. **`SubscriptionLogicV1.sol`**

我们逻辑的第一个版本：

- 添加订阅套餐
- 让用户订阅
- 检查用户是否活跃

简单、清晰——并且完美运行。

---

### 4. **`SubscriptionLogicV2.sol`**

一个具有**额外功能**的升级版本：

- V1 的所有功能 ✅
- 但现在你可以暂停或恢复用户账户 🔒

当我们准备升级时，我们将把代理指向这个合约。

---

## 🎯 为什么这种设置很强大

假设你有成千上万的用户订阅了。他们的订阅信息保存在代理中。然后你意识到：“我想让用户可以暂停他们的订阅。”

在一个不可升级的世界里，你会被卡住。你要么：

- 部署一个新合约并丢失所有旧数据，或者
- 迁移所有数据——这很麻烦且对用户不友好

但在这里呢？

你只需编写一个带有暂停/恢复功能的新逻辑合约…然后**调用 `upgradeTo()`**。

无需迁移。无需停机。不影响用户。

---

**所以在接下来的部分，我们将逐个合约地讲解——解释它的作用、工作原理，以及为什么这种架构是可升级安全且节省 gas 的。**

让我们从共享布局开始。

## 📦  `SubscriptionStorageLayout.sol` – 共享内存蓝图

```solidity
 
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract SubscriptionStorageLayout {
    address public logicContract;
    address public owner;

    struct Subscription {
        uint8 planId;
        uint256 expiry;
        bool paused;
    }

    mapping(address => Subscription) public subscriptions;
    mapping(uint8 => uint256) public planPrices;
    mapping(uint8 => uint256) public planDuration;
}

```

### ✅ 合约声明

```solidity
 
contract SubscriptionStorageLayout {

```

这是一个独立的合约，**只保存状态变量**——它不包含任何函数（除了后面继承的逻辑）。其思想是**将存储与逻辑分离**，这是代理升级模式的关键部分。

这个布局合约就像一个**蓝图**，定义了代理和逻辑合约的**内存结构**。

通过导入和继承这个布局，两个合约可以**共享和操作相同的数据**，前提是它们的内存布局顺序相同——这对于 `delegatecall` 的正确工作至关重要。

---

### 🔑 `logicContract`

```solidity
 
    address public logicContract;

```

- 这存储了**当前实现合约的地址**——即包含实际功能的逻辑合约。
- 代理合约用它来知道**在哪里使用 `delegatecall` 转发调用**。
- 你以后可以通过代理中的 `upgradeTo()` 函数更新这个地址，以切换到新版本的逻辑。

---

### 👑 `owner`

```solidity
 
    address public owner;

```

- 这记录了合约的**管理员或部署者**——唯一可以升级到新逻辑版本的人。
- 你以后可以扩展它，使用基于角色的访问控制或多签所有权以提高安全性。

---

### 📦  `Subscription` 结构体

```solidity
 
    struct Subscription {
        uint8 planId;
        uint256 expiry;
        bool paused;
    }

```

我们来解析一下：

- `uint8 planId`:
    
    用户套餐的标识符。一个小的数字，如 `1`, `2`, 或 `3`，代表不同的层级（例如，基础版、专业版、高级版）。
    
    → 我们使用 `uint8` 来**节省 gas**（与 `uint256` 相比）。
    
- `uint256 expiry`:
    
    一个时间戳，指示订阅何时到期。
    
    → 我们在这里使用 `uint256`，因为 Unix 时间戳是大数字。
    
- `bool paused`:
    
    一个开关，用于**在不删除的情况下**临时停用用户的订阅。
    
    → 可用于允许用户暂停或恢复他们的套餐。
    

---

### 🗂 订阅映射

```solidity
 
    mapping(address => Subscription) public subscriptions;

```

- 每个用户（`address`）都有自己的 `Subscription` 对象。
- 这允许我们**跟踪每个用户的有效套餐**、其到期时间和暂停状态。

```solidity
 
    mapping(uint8 => uint256) public planPrices;

```

- 这定义了每个套餐需要多少 ETH。
- 例如，`planPrices[1] = 0.01 ether`, `planPrices[2] = 0.05 ether`。

```solidity
 
    mapping(uint8 => uint256) public planDuration;

```

- 这告诉我们每个套餐**持续多久**（以秒为单位）。
- 例如，`planDuration[1] = 30 days`, `planDuration[2] = 365 d`

## 🧭  `SubscriptionStorage.sol` – 代理合约

```solidity
 
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "./SubscriptionStorageLayout.sol";

contract SubscriptionStorage is SubscriptionStorageLayout {
    modifier onlyOwner() {
        require(msg.sender == owner, "Not owner");
        _;
    }

    constructor(address _logicContract) {
        owner = msg.sender;
        logicContract = _logicContract;
    }

    function upgradeTo(address _newLogic) external onlyOwner {
        logicContract = _newLogic;
    }

    fallback() external payable {
        address impl = logicContract;
        require(impl != address(0), "Logic contract not set");

        assembly {
            calldatacopy(0, 0, calldatasize())
            let result := delegatecall(gas(), impl, 0, calldatasize(), 0, 0)
            returndatacopy(0, 0, returndatasize())

            switch result
            case 0 { revert(0, returndatasize()) }
            default { return(0, returndatasize()) }
        }
    }

    receive() external payable {}
}

```

### 头部

```solidity
 
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "./SubscriptionStorageLayout.sol";

```

- 通常的许可证和版本编译指令。
- 我们**导入**共享的存储布局——这确保了代理与逻辑合约具有相同的变量结构。 如果你还记得，`delegatecall` 意味着**代码从逻辑合约运行但存储属于代理**，所以两者必须共享完全相同的布局。

---

### 🔧 合约声明

```solidity
 
contract SubscriptionStorage is SubscriptionStorageLayout {

```

我们定义了一个名为 `SubscriptionStorage` 的合约。这不是你的业务逻辑所在的地方——这是**用户将与之交互的合约**，但它会**将所有实际工作委托**给逻辑合约。

它**继承**自 `SubscriptionStorageLayout`，意味着它现在拥有：

- `logicContract`（指向当前逻辑的指针）
- `owner`
- 所有的映射（`subscriptions`, `planPrices`, `planDuration`）

---

### 🔐 所有者检查修饰器

```solidity
 
    modifier onlyOwner() {
        require(msg.sender == owner, "Not owner");
        _;
    }

```

- 这个修饰器用于**保护敏感函数**——比如升级合约。
- 只有部署者（或合约的所有者）可以更改正在使用的逻辑。

---

### 🏗️ 构造函数

```solidity
 
    constructor(address _logicContract) {
        owner = msg.sender;
        logicContract = _logicContract;
    }

```

这是在代理首次部署时**运行一次**的函数。

- `owner = msg.sender`: 部署者成为所有者。
- `logicContract = _logicContract`: 你传入初始逻辑合约的地址——通常是 `SubscriptionLogicV1`。

所以现在你的代理知道当用户开始与之交互时使用哪个逻辑。

---

### 🔄 逻辑升级

```solidity
 
    function upgradeTo(address _newLogic) external onlyOwner {
        logicContract = _newLogic;
    }

```

这个函数使得整个可升级架构成为可能。

- 它将 `logicContract` 更新为指向一个**新合约**（如 `SubscriptionLogicV2`）。
- 受 `onlyOwner` 保护，因此只有部署者可以升级。
- 当这种情况发生时，**存储保持不变**，但所有新的交互将使用新的逻辑。

🧠 为什么这很强大？

你可以在**不触及用户数据或要求人们重新部署**的情况下修复错误、添加功能或重构代码。

---

### ✨Fallback 函数 – 魔法发生的地方

```solidity
 
    fallback() external payable {
        address impl = logicContract;
        require(impl != address(0), "Logic contract not set");

        assembly {
            calldatacopy(0, 0, calldatasize())
            let result := delegatecall(gas(), impl, 0, calldatasize(), 0, 0)
            returndatacopy(0, 0, returndatasize())

            switch result
            case 0 { revert(0, returndatasize()) }
            default { return(0, returndatasize()) }
        }
    }

```

这是整个代理设置中**最关键的部分**。

让我们分解一下：

### 🧭 什么是 `fallback()`？

- 它是一个特殊函数，当用户调用此代理合约中**不存在的函数**时会被触发。
- 这很完美，因为这个代理**自身没有业务逻辑**。
- 所以，每次用户尝试与我们其他合约中的函数(如 `subscribe()` 或 `isActive()`)交互时，都会触发这个函数。.

### 🛠️ 内联汇编做了什么？

让我们一步步解码它：

```solidity
 
address impl = logicContract;
require(impl != address(0), "Logic contract not set");

```

- 确保已设置逻辑合约。
- 将其存储在 `impl` 中。

---

```solidity
 
calldatacopy(0, 0, calldatasize())

```

- 将**输入数据**（函数签名 + 参数）复制到内存槽 `0`。

```solidity
 
let result := delegatecall(gas(), impl, 0, calldatasize(), 0, 0)

```

- 这是主要部分。
- 我们在说：“嘿，在逻辑合约（`impl`）上运行这个输入…”
- `delegatecall` 运行逻辑代码，但使用**此代理的存储**和**此代理的上下文**。

---

```solidity
 
returndatacopy(0, 0, returndatasize())

```

- 将逻辑合约执行返回的任何内容复制到内存中。
- 可能是返回值或错误消息。

```solidity
 
switch result
case 0 { revert(0, returndatasize()) }
default { return(0, returndatasize()) }

```

- 如果逻辑调用**失败**，我们回退（revert）并返回错误。
- 否则，我们将结果返回给原始调用者——就像代理自己执行了它一样。

---

### 💸 `receive()` 函数

```solidity
 
    receive() external payable {}

```

- 一个安全网，允许代理**接受原始 ETH 转账**。
- 在这里你可能不需要它，但当合约直接接收 ETH 时（例如，在支付期间）通常很有用。

## 🧩  `SubscriptionLogicV1.sol` – 第一个逻辑合约

```solidity
 
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "./SubscriptionStorageLayout.sol";

contract SubscriptionLogicV1 is SubscriptionStorageLayout {
    function addPlan(uint8 planId, uint256 price, uint256 duration) external {
        planPrices[planId] = price;
        planDuration[planId] = duration;
    }

    function subscribe(uint8 planId) external payable {
        require(planPrices[planId] > 0, "Invalid plan");
        require(msg.value >= planPrices[planId], "Insufficient payment");

        Subscription storage s = subscriptions[msg.sender];
        if (block.timestamp < s.expiry) {
            s.expiry += planDuration[planId];
        } else {
            s.expiry = block.timestamp + planDuration[planId];
        }

        s.planId = planId;
        s.paused = false;
    }

    function isActive(address user) external view returns (bool) {
        Subscription memory s = subscriptions[user];
        return (block.timestamp < s.expiry && !s.paused);
    }
}

```

## 🧩 `SubscriptionLogicV1.sol` – 第一个逻辑合约

```solidity
  
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "./SubscriptionStorageLayout.sol";

```

- 标准的 SPDX 许可证和 Solidity 版本。
- 我们**导入共享的存储布局**，以便这个逻辑合约可以访问**相同的状态变量**（如 `subscriptions`, `planPrices`, `planDuration` 等）。
- 这**至关重要**，因为所有存储更新都将发生在代理的内存中（通过 `delegatecall`），所以两个合约必须共享完全相同的内存布局。

---

```solidity
  
contract SubscriptionLogicV1 is SubscriptionStorageLayout {

```

- 我们定义逻辑合约并继承 `SubscriptionStorageLayout` 以便在通过委托调用（delegatecall）时，我们可以访问代理的存储。

这个合约处理：

- 添加新套餐
- 用户订阅
- 检查活跃状态

现在，让我们分解每个函数。

---

### 1️⃣ `addPlan()`

```solidity
  
function addPlan(uint8 planId, uint256 price, uint256 duration) external {
    planPrices[planId] = price;
    planDuration[planId] = duration;
}

```

🧠 **它的作用：**

- 允许所有者（或任何调用它的人）注册一个新的订阅套餐。
- 每个 `planId` 代表一个唯一的套餐（例如 `1 = 基础版`, `2 = 专业版`）。
- 我们将套餐的价格存储在 `planPrices[planId]` 中。
- 我们还使用 `planDuration[planId]` 设置套餐的持续时间。

📌 **为什么这有用：**

- 这使得订阅系统可定制——你可以定义具有不同定价层级和持续时间的多个套餐。
- 由于这个合约可以升级，套餐模型也可以随着时间的推移而发展。

---

### 2️⃣ `subscribe()`

```solidity
  
function subscribe(uint8 planId) external payable {
    require(planPrices[planId] > 0, "Invalid plan");
    require(msg.value >= planPrices[planId], "Insufficient payment");

    Subscription storage s = subscriptions[msg.sender];
    if (block.timestamp < s.expiry) {
        s.expiry += planDuration[planId];
    } else {
        s.expiry = block.timestamp + planDuration[planId];
    }

    s.planId = planId;
    s.paused = false;
}

```

🧠 **它的作用：**

- 让用户通过发送 ETH 来**订阅**特定的套餐。
- 首先，它检查：
    - 套餐是否有效 (`planPrices[planId] > 0`)
    - 用户是否发送了足够的 ETH (`msg.value >= price`)
- 然后我们从 `subscriptions` 映射中获取调用者的订阅记录。

📦 两种情况：

1. 如果用户还有剩余时间 (`block.timestamp < s.expiry`)：
    - 将新的持续时间添加到当前到期时间。这让他们可以**延长**订阅。
2.  如果订阅已过期：
    - 将到期时间重置为 `当前时间 + 持续时间`。这是一个**全新的订阅**。

然后我们：

- 设置 `s.planId = planId` 来记录他们选择的套餐。
- 通过设置 `s.paused = false` 来取消暂停订阅。

📌**为什么这很聪明：**

- 它简单且节省 gas。
- 它在一个函数中支持新用户和现有用户。
- 它会自动“恢复”已暂停的订阅（对于 V2 等功能很有用）。

---

### 3️⃣ `isActive()`

```solidity
  
function isActive(address user) external view returns (bool) {
    Subscription memory s = subscriptions[user];
    return (block.timestamp < s.expiry && !s.paused);
}

```

🧠 **它的作用：**

- 让任何人检查用户的订阅当前是否活跃。

它返回 `true` **仅当**：

- 当前时间在订阅到期之前
- 并且订阅未暂停

📌 **为什么这很重要：**

- 这是你将在以下场景中使用的只读辅助函数：
    - 前端（显示订阅状态）
    - 对高级功能进行门控访问
    - 显示续订提示

## 🚀`SubscriptionLogicV2.sol` – 升级版（带有暂停/恢复功能）

```solidity
 
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "./SubscriptionStorageLayout.sol";

contract SubscriptionLogicV2 is SubscriptionStorageLayout {
    function addPlan(uint8 planId, uint256 price, uint256 duration) external {
        planPrices[planId] = price;
        planDuration[planId] = duration;
    }

    function subscribe(uint8 planId) external payable {
        require(planPrices[planId] > 0, "Invalid plan");
        require(msg.value >= planPrices[planId], "Insufficient payment");

        Subscription storage s = subscriptions[msg.sender];
        if (block.timestamp < s.expiry) {
            s.expiry += planDuration[planId];
        } else {
            s.expiry = block.timestamp + planDuration[planId];
        }

        s.planId = planId;
        s.paused = false;
    }

    function isActive(address user) external view returns (bool) {
        Subscription memory s = subscriptions[user];
        return (block.timestamp < s.expiry && !s.paused);
    }

    function pauseAccount(address user) external {
        subscriptions[user].paused = true;
    }

    function resumeAccount(address user) external {
        subscriptions[user].paused = false;
    }
}

```

## 🚀 `SubscriptionLogicV2.sol`  – 升级版（带有暂停/恢复功能）

```solidity
  
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "./SubscriptionStorageLayout.sol";

```

这个设置和之前一样——我们导入共享的存储布局，以便在 `delegatecall` 期间可以**安全地使用代理的存储**。

---

```solidity
  
contract SubscriptionLogicV2 is SubscriptionStorageLayout {

```

这定义了我们逻辑合约的第二个版本。通过继承布局，它可以完全访问与代理完全相同的存储结构。

现在让我们逐行讲解每个函数。

---

### 1️⃣ `addPlan()`

```solidity
  
function addPlan(uint8 planId, uint256 price, uint256 duration) external {
    planPrices[planId] = price;
    planDuration[planId] = duration;
}

```

✅ **它的作用：**

- 添加或更新订阅套餐。
- 使用 `planId` 作为套餐的标识符。
- 存储它的成本和持续时间。

🧠 **与 V1 相同**——我们这里没有改变任何东西，因为它已经工作得很好。

---

### 2️⃣ `subscribe()`

```solidity
  
function subscribe(uint8 planId) external payable {
    require(planPrices[planId] > 0, "Invalid plan");
    require(msg.value >= planPrices[planId], "Insufficient payment");

    Subscription storage s = subscriptions[msg.sender];
    if (block.timestamp < s.expiry) {
        s.expiry += planDuration[planId];
    } else {
        s.expiry = block.timestamp + planDuration[planId];
    }

    s.planId = planId;
    s.paused = false;
}

```

✅ **它的作用：**

- 用户调用此函数来订阅或续订他们的套餐。
- 如果套餐仍然有效，则延长到期时间。
- 如果已过期，则从现在开始设置一个新的到期时间。
- 它还确保订阅是**未暂停的**——如果用户休息后回来，这很有用。

🧠 同样，这与 V1 相同——不需要改动已经工作的部分。

---

### 3️⃣ `isActive()`

```solidity
  
function isActive(address user) external view returns (bool) {
    Subscription memory s = subscriptions[user];
    return (block.timestamp < s.expiry && !s.paused);
}

```

✅ **它的作用：**

- 返回 `true` 如果：
    - 订阅尚未过期
    - 订阅未暂停

🧠 被前端应用程序或其他智能合约用来**检查是否有权使用高级功能**。

---

### 🆕 4️⃣ `pauseAccount()`

```solidity
  
function pauseAccount(address user) external {
    subscriptions[user].paused = true;
}

```

🧠 **它的作用：**

- 手动暂停用户的账户。
- 可以由管理员使用，或者在未来的版本中委托给用户自己使用。

📌 **为什么这很重要：**

- 有些用户可能想暂时冻结他们的订阅。
- 或者，作为管理员，你可能由于滥用或付款失败而想要暂停一个账户。

🧪**它不触及到期时间**——所以时钟仍在滴答走，但账户被阻止访问。

---

### 🆕 5️⃣ `resumeAccount()`

```solidity
  
function resumeAccount(address user) external {
    subscriptions[user].paused = false;
}

```

✅  **它的作用：**

- 重新启用已暂停的订阅。

🧠 在以下情况下有用：

- 用户想要自己取消暂停
- 管理员在解决问题后想要重新启用

这简单地将 `paused` 标志翻回 `false`，恢复访问。

# 🛠️ 运行可升级订阅管理器（在 Remix 中）

我们将使用 `delegatecall` 部署一个基于代理的可升级逻辑订阅系统。按照以下步骤在 **Remix** 中进行测试。

---

## 🔧 步骤 1：创建 3 个合约

打开 Remix，然后：

1.  在**文件资源管理器**中，创建这 3 个新文件：
    - `SubscriptionStorage.sol`
    - `SubscriptionLogicV1.sol`
    - `SubscriptionLogicV2.sol`
2. 将前面解析中各自的代码粘贴到每个文件中。

---

## 🧱 步骤 2：编译所有合约

1.  点击 **Solidity Compiler 标签**（左侧边栏 – 第二个图标）。
2. 确保编译器版本设置为 `0.8.x`（0.8 范围内的任何版本）。
3. 编译所有三个合约：
    - `SubscriptionStorage.sol`
    - `SubscriptionLogicV1.sol`
    - `SubscriptionLogicV2.sol`

✅ 你应该看到每个都显示 `Compilation successful`。

---

## 🚀 步骤 3：部署逻辑合约 (V1)

1. 转到 **Deploy & Run Transactions** 标签（第三个图标）。
2.  在 **Contract** 下拉菜单中，选择 `SubscriptionLogicV1`。
3. 点击 **Deploy**。
4. 复制部署的合约地址——下一步你会需要它。

---

## 📦 步骤 4：部署代理合约

1. 在 **Contract** 下拉菜单中，选择 `SubscriptionStorage`。
2. 在 `Deploy` 按钮旁边的输入框中，粘贴 **V1 逻辑地址**（带引号）：
    
    ```
    
    "0x1234...abcd"
    
    ```
    
3. 点击 **Deploy**。

🎉 你现在有了一个连接到你的第一个逻辑合约的代理！

- `logicV1 = <address of SubscriptionLogicV1>`
- `proxy = <address of SubscriptionStorage>`

---

## 🧠 步骤 5：通过代理与 V1 交互

由于代理本身不暴露任何逻辑函数，Remix 不会自动显示像 `addPlan()` 或 `subscribe()` 这样的按钮。

以下是使用 **V1 ABI** 进行交互的方法：

### ### ➕➕ 使用 V1 ABI 加载代理

1.  在 Deploy 标签页中，向下滚动到 **"At Address"** 部分。
2. 确保 **Contract** 下拉菜单中选择了 `SubscriptionLogicV1`。
3. 在输入框中，粘贴你部署的**代理地址**。
4. 点击 **At Address**。
5. 现在v1实例已经部署完毕，现在你可以交互它了。

✅ 你现在会看到 V1 的函数，如 `addPlan()` 和 `subscribe()`——但它们是通过 `delegatecall` **通过代理执行的**。

---

## 💰 步骤 6：测试订阅流程 (V1)

过加载的代理界面尝试这些操作：

1. 调用：
    
    ```solidity
       
    addPlan(1, 10000000000000000, 60)
    
    ```
    
    → 添加一个套餐：**0.01 ETH 持续 60 秒**
    
2. 调用：
    
    ```solidity
       
    subscribe(1)
    
    ```
    
    → 确保**发送 0.01 ETH** 随交易一起
    
3. 调用：
    
    ```solidity
       
    isActive(<your wallet address>)
    
    ```
    
    → 应该返回 `true`
    

✅ 所有这些逻辑都来自 `SubscriptionLogicV1`，但是**通过代理执行的**。

---

## 🔄 步骤 7：升级到 V2 逻辑

1. 在 **Contract** 下拉菜单中，选择 `SubscriptionLogicV2`。
2. 点击 **Deploy**。
3. 复制部署的 V2 地址：
    
    ```
    
    logicV2 = <address of SubscriptionLogicV2>
    
    ```
    
4. 向下滚动到你已部署的 **SubscriptionStorage** 实例。
5. 调用 `upgradeTo` 函数，参数是 V极速地址：
    
    ```solidity
       
    upgradeTo(logicV2)
    
    ```
    
    ✅ 这告诉代理现在使用 V2 合约来处理逻辑。
    

---

## 🧪 步骤 8：通过同一个代理使用 V2 功能

你现在可以调用**新的暂停/恢复功能**——仍然使用同一个代理地址。

1. 再次向下滚动到 **"At Address"** 部分。
2. 在 **Contract** 下拉菜单中，选择 `SubscriptionLogicV2`。
3. 再次粘贴你的**代理地址**并点击 **At Address**。
4. 现在，你将看到 V2 的一个新实例。

你现在应该看到额外的函数，如：

- `pauseAccount(address)`
- `resumeAccount(address)`

### 测试它们：

1. 调用：
    
    ```solidity
       
    pauseAccount(<your wallet address>)
    
    ```
    
2. 然后调用：
    
    ```solidity
       
    isActive(<your wallet address>)
    
    ```
    
    → 应该返回 `false`
    
3. 调用：
    
    ```solidity
       
    resumeAccount(<your wallet address>)
    
    ```
    
    → `isActive()` 现在应该再次返回 `true`
    

---

## 🎉你做到了！

你刚刚：

- 部署了一个逻辑合约 (V1)
- 通过代理路由调用
- 升级到了一个新的逻辑合约 (V2)
- 在同一存储中保留了所有数据

欢迎来到**可升级智能合约**的世界——OpenZeppelin 的 UUPS 和代理系统等巨头使用的相同模式。