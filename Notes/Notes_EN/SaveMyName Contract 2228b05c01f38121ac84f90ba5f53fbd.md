# SaveMyName Contract

In our last contract, we learned how to **store and update a simple number (counter)** on the blockchain. That introduced us to **state variables** and how Solidity functions can modify them.

<aside>
💻

**Here is the complete code**

[https://github.com/snehasharma76/30daysSolidity_Web3Compass/blob/master/SaveMyName.sol](https://github.com/snehasharma76/30daysSolidity_Web3Compass/blob/master/SaveMyName.sol)

</aside>

Now, we are moving forward by storing **text data**—a user’s **name and bio**. This is important because real-world applications often involve **handling user information**. Along the way, we’ll explore **strings, memory storage, function return types, and the `view` keyword** in Solidity.

Let’s break everything down step by step.

---

## **Storing a Name and Bio on the Blockchain**

When building traditional applications, storing user data is easy. You typically save it in a **database** that can be updated whenever needed. But smart contracts don’t work like that. Instead, everything is stored **permanently on the blockchain**, which makes handling text data different from what you might be used to.

Here’s how we declare two **state variables** in Solidity to store a user’s name and bio:

```solidity
  
string name;
string bio;
```

### **What’s Happening Here?**

- **`string name;` and `string bio;`** → These are **state variables**, meaning their values will be **stored permanently** on the blockchain.
- Unlike numbers (`uint256`), **strings are more complex** in Solidity and require special handling.
- These variables are **internal by default**, meaning  the value can only be accessed from within this contract or any contract derived from this one (More on that later)

Think of this like an **on-chain profile** where a user can set and update their name and bio.

---

## **The `add()` Function – Storing Data**

To allow users to store their name and bio, we use the following function:

```solidity
  
function add(string memory _name, string memory _bio) public {
    name = _name;
    bio = _bio;
}
```

### **Breaking it Down**

### **1️⃣ Function Parameters – Accepting User Input**

A function can take inputs (also called **parameters**) when a user calls it.

- `_name` and `_bio` are **placeholders** that hold the user’s input.
- When the function runs, these values are stored in the contract’s **state variables** (`name` and `bio`).

For example:

```solidity
  
add("Alice", "Blockchain Developer");
```

After calling this function, `name` is set to `"Alice"`, and `bio` is set to `"Blockchain Developer"`.

### **2️⃣ Understanding the `_` Naming Convention**

You might have noticed that we used an **underscore (`_`)** before the parameter names: `_name` and `_bio`.

This is **not a requirement** in Solidity—it’s just a common naming convention used to **differentiate function parameters from state variables**.

For example:

- `name` refers to the **state variable**.
- `_name` refers to the **function parameter**.

However, you are **free to use your own naming convention**. The following function works exactly the same:

```solidity
  
function add(string memory newName, string memory newBio) public {
    name = newName;
    bio = newBio;
}
```

The underscore is **just a style choice**, but using it can help make the code **easier to read** by distinguishing function inputs from contract storage variables.

### **3️⃣ The `memory` Keyword – Why It’s Needed**

Notice that the parameters are defined as `string memory _name, string memory _bio`.

But why do we need `memory` here?

- Solidity has two main types of storage:
    - **Storage** → Data that is permanently stored on the blockchain (like `name` and `bio`).
    - **Memory** → Temporary storage that **only exists while the function is running**.
- When a function **accepts** a string as an input, Solidity requires us to explicitly say whether it is stored in `memory` or `storage`.
- Since function parameters **do not need to be stored permanently**, we use `memory`.

Think of `memory` as a **scratchpad**—it temporarily holds the data while the function runs, and then it disappears.

---

## **The `retrieve()` Function – Fetching Data from the Blockchain**

Once the user has stored their name and bio, we need a way to **retrieve** that information. That’s where the `retrieve()` function comes in:

```solidity
  
function retrieve() public view returns (string memory, string memory) {
    return (name, bio);
}
```

### **Understanding `view` Functions**

The `view` keyword is **new** in this contract. It tells Solidity that this function **only reads data** and does not modify the blockchain.

### **Why is `view` Important?**

- Any function that **modifies** a state variable (like `add()`) **requires gas fees** because it updates the blockchain.
- A function marked as `view` **does not cost gas** when called.
- It simply **fetches** and **returns** existing data.

This is why `retrieve()` is **free to call**—it does not change anything on the blockchain. It just reads and returns the stored name and bio.

---

## **Understanding `returns (string memory, string memory)`**

Unlike `add()`, which just updates data, `retrieve()` **returns** data to whoever calls it.

```solidity
  
returns (string memory, string memory)
```

This tells Solidity that the function **returns two string values** (the name and the bio).

Just like function parameters, **returned strings must be stored in memory**, so we specify `memory` again.

When the function is called, it gives back the stored values:

```solidity
  
retrieve() → Returns ("Alice", "Blockchain Developer")
```

Think of this function like checking your **social media profile**—you can see your stored info, but you are **not modifying anything**.

---

## **Making the Contract More Efficient**

Right now, we have **two separate functions**:

1. `add()` → To store the name and bio.
2. `retrieve()` → To fetch the name and bio.

While this works fine, we can make it **more compact** by combining them into a **single function**:

```solidity
  
function saveAndRetrieve(string memory _name, string memory _bio) public returns (string memory, string memory) {
    name = _name;
    bio = _bio;
    return (name, bio);
}

```

### **How This Version Works:**

- This function **stores** the name and bio just like `add()` did.
- Instead of needing a separate `retrieve()` function, it **immediately returns the values** after saving them.

This reduces the number of function calls, making the contract **shorter** and **easier to use**.

### **But There's a Catch…**

While this approach **saves space in the contract**, it has a downside:

- Since this function **modifies the blockchain**, calling it **always costs gas**, even if the user just wants to retrieve the data.
- With the original two-function approach, `retrieve()` was **free to call** because it did not modify anything.

So, while this version is **more compact**, it **is not necessarily more efficient** in terms of gas usage.

---

## **Key Takeaways**

- **Strings require special handling** in Solidity and must be explicitly stored in `memory` inside functions.
- **State variables are stored permanently on the blockchain**, while `memory` variables only exist temporarily during function execution.
- **The `view` keyword makes a function free to call** because it doesn’t modify the blockchain.
- **The underscore (`_`) in function parameters is just a naming convention** and not a requirement.
- **Combining functions can make contracts shorter**, but it can also **increase gas costs** if not done carefully.

## **Next Steps – Improving the Contract**

Now that we have a basic name-and-bio storage contract, here are some ways we can improve it:

1. **Add more user data** – Instead of storing only  name and bio, modify the contract to store data like age, profession , etc