# PollStation Contract

So far, we have worked with **different data types** like numbers (`uint256`) and strings (`string`). We have seen how **state variables** store values on the blockchain and how functions can **modify or retrieve them**.

But smart contracts don’t just store individual values—they often need to **organize and structure** data efficiently.

For example, in a **polling system**, we aren’t just tracking a **single** candidate or vote. We need to store **multiple candidates and their corresponding votes**.

<aside>
💻

Here is the complete contract 👇🏼

[https://github.com/snehasharma76/30daysSolidity_Web3Compass/blob/master/PollStation.sol](https://github.com/snehasharma76/30daysSolidity_Web3Compass/blob/master/PollStation.sol)

</aside>

To do this, Solidity provides **arrays** (to store lists of data) and **mappings** (to associate values with unique keys).

In this tutorial, we’ll explore how **arrays and mappings** work in Solidity while breaking down the **PollStation** contract.

---

## **Understanding the PollStation Contract**

This contract allows users to:

1. **Add candidates** to a poll.
2. **Retrieve the list of candidates**.
3. **Cast votes for candidates**.
4. **Check the total votes** a candidate has received.

To achieve this, we need two key components:

- **An array** to store the names of all candidates.
- **A mapping** to keep track of how many votes each candidate has received.

---

## **Declaring the Candidate List and Vote Tracking**

Before writing functions, we need to **store** candidates and their votes efficiently.

```solidity
 
string[] public candidateNames;
mapping(string => uint256) voteCount;

```

### **1️⃣ Arrays – Storing a List of Candidates**

```solidity
 
string[] public candidateNames;

```

This line declares an **array** to store candidate names.

### **What is an Array?**

An **array** is a **list of elements** of the same type. In Solidity, an array can hold numbers, strings, addresses, or other data types.

Here, we are using an array to store **multiple candidates’ names**.

### **Why Use an Array?**

- **Arrays store multiple values in an ordered list.**
- **We can retrieve all elements at once**—helpful when displaying all candidates.
- **We can dynamically add new elements** using `.push()`.

### **Example Usage**

If we add two candidates:

```solidity
 
addCandidateNames("Alice");
addCandidateNames("Bob");

```

Our array will now look like this:

```solidity
 
candidateNames = ["Alice", "Bob"];

```

**Note:** Since `candidateNames` is `public`, Solidity automatically creates a **getter function** for it, meaning users can retrieve the list of candidates without writing a separate function.

---

### **2️⃣ Mappings – Storing Vote Counts**

```solidity
 
mapping(string => uint256) voteCount;

```

This line creates a **mapping** to track votes for each candidate.

### **What is a Mapping?**

A **mapping** is like a **dictionary** that links a key (in this case, a `string` representing the candidate’s name) to a value (a `uint256` representing their vote count).

### **Why Use a Mapping Instead of an Array?**

Mappings allow **instant lookups**. Instead of searching an array to find a candidate’s vote count, we can access it directly using their name.

### **Example Usage**

Let’s say we store two candidates and their votes:

```solidity
 
voteCount["Alice"] = 3;
voteCount["Bob"] = 5;

```

Now, we can retrieve votes instantly:

```solidity
 
voteCount["Alice"]; // Returns 3
voteCount["Bob"];   // Returns 5

```

Mappings are **extremely efficient** because they provide **direct access** to stored values without needing to loop through a list.

---

## **Adding Candidates – The `addCandidateNames()` Function**

```solidity
 
function addCandidateNames(string memory _candidateNames) public {
    candidateNames.push(_candidateNames);
    voteCount[_candidateNames] = 0;
}

```

### **Breaking It Down:**

1️⃣ **Accepts a candidate name as input (`_candidateNames`)**

- This is a **function parameter**, meaning the user provides a candidate’s name when calling the function.
- We use `memory` because strings are dynamic in Solidity and must be explicitly stored in temporary memory inside functions.

2️⃣ **Stores the candidate in the array**

- `candidateNames.push(_candidateNames);` adds the name to our `candidateNames` array.

3️⃣ **Initializes the candidate’s vote count to zero**

- `voteCount[_candidateNames] = 0;` ensures each candidate starts with zero votes.

Now, when someone calls:

```solidity
 
addCandidateNames("Alice");

```

- `"Alice"` is added to `candidateNames[]`.
- `"Alice"` is initialized in the `voteCount` mapping with `0` votes.

---

## **Retrieving the List of Candidates – The `getCandidateNames()` Function**

```solidity
 
function getcandidateNames() public view returns (string[] memory) {
    return candidateNames;
}

```

### **Understanding `view` Functions**

- `public view` → Since this function only **reads** data (and does not modify state variables), it is **free to call**.
- `returns (string[] memory)` → Returns an **array of strings** stored in `candidateNames`.
- Since `candidateNames` is `public`, Solidity **automatically** provides a getter function.

This function allows users to see **who is running in the poll**.

---

## **Voting for a Candidate – The `vote()` Function**

```solidity
 
function vote(string memory _candidateNames) public {
    voteCount[_candidateNames] += 1;
}

```

### **How This Works:**

- `public` → Allows anyone to call this function.
- `string memory _candidateNames` → Takes the candidate’s name as input.
- `voteCount[_candidateNames] += 1;` → Increases the vote count for that candidate by 1.

### **Potential Issues**

Right now, **anyone can vote as many times as they want**.

A real-world implementation should **prevent duplicate voting**—but for now, we’re keeping things simple.

---

## **Checking a Candidate’s Votes – The `getVote()` Function**

```solidity
 
function getVote(string memory _candidateNames) public view returns (uint256) {
    return voteCount[_candidateNames];
}

```

### **Breaking It Down:**

- `public view` → Since this function **only reads data**, it does not cost gas.
- `returns (uint256)` → Returns the vote count for the given candidate.
- `voteCount[_candidateNames]` → Retrieves the candidate’s votes from the mapping.

---

## **How This Contract Works in Action**

### **Scenario 1 – Setting Up the Poll**

1. **Deploy the contract** to the blockchain.
2. Call `addCandidateNames("Alice")`.
3. Call `addCandidateNames("Bob")`.
4. Now, the list of candidates is: `["Alice", "Bob"]`.

### **Scenario 2 – Voting**

1. Call `vote("Alice")` → Alice gets 1 vote.
2. Call `vote("Alice")` again → Alice now has 2 votes.
3. Call `vote("Bob")` → Bob gets 1 vote.

### **Scenario 3 – Checking Votes**

1. Call `getVote("Alice")` → Returns `2`.
2. Call `getVote("Bob")` → Returns `1`.

---

## **Key Takeaways**

1. **Arrays (`string[]`) allow us to store multiple candidate names.**
2. **Mappings (`mapping(string => uint256)`) allow us to associate each candidate with a vote count.**
3. **The `view` keyword ensures read-only functions are free to call.**
4. **Functions modify or retrieve data based on state variables.**
5. **Without voter restrictions, this contract allows unlimited voting—this should be improved in real-world applications.**

---

## **Possible Improvements**

1. **Prevent Multiple Votes** – Add a `mapping(address => bool)` to track if a user has already voted.
2. **Check If Candidate Exists** – Ensure users cannot vote for a non-existent candidate.