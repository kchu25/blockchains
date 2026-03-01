
@def title = "On-Chain Reputation: Beyond Lending and DeFi"
@def published = "1 March 2026"
@def tags = ["solidity", "code"]

# On-Chain Reputation: Beyond Lending and DeFi

> **The Core Insight**: Smart contracts aren't just financial ledgers. They're **trust machines** that record verifiable actions on-chain. A reputation system is a contract that:
> 1. **Records events** (user completes task, seller ships goods, reviewer validates claim)
> 2. **Computes metrics** (score, badge, leaderboard) from those events
> 3. **Exposes data** via a public interface for any web app to query
>
> No centralized database. No moderator needed. No single point of failure. Third parties are automated away.

---

## Why On-Chain Reputation?

**Traditional systems:**
- Yelp/Amazon: centralized database, can delete reviews, takes 30% commission
- LinkedIn: you don't own your profile; they control your reputation
- eBay: they decide dispute outcomes, can ban you without appeal

**On-Chain:**
- Immutable ledger of actions (permanent, public, verifiable)
- Transparent calculation (you can audit the scoring formula)
- Composable (other contracts can read your score and react)
- Decentralized (anyone can build a UI on top of the same data)

---

## Simple Example: Task Completion Reputation

Imagine a gig marketplace. Instead of Fiverr (centralized), users build reputation through completed work tracked on-chain.

### Contract Structure

```solidity
pragma solidity ^0.8.24;

contract TaskReputation {
    // Event: Task completed
    event TaskCompleted(
        address indexed worker,
        uint256 taskId,
        uint256 timestamp,
        uint256 amount
    );

    // Event: Review posted
    event ReviewPosted(
        address indexed reviewer,
        address indexed worker,
        uint256 taskId,
        uint8 rating,  // 1-5 stars
        string feedback
    );

    // Struct: Task
    struct Task {
        address requester;
        address worker;
        uint256 amount;
        bool completed;
        uint8 rating;
    }

    // Struct: Reputation
    struct ReputationMetrics {
        uint256 tasksCompleted;
        uint256 totalEarned;
        uint8 avgRating;  // 1-5, stored as uint8
        uint256 reviewCount;
    }

    // Storage
    mapping(uint256 => Task) public tasks;
    mapping(address => ReputationMetrics) public reputation;
    mapping(address => uint8[]) public ratings;  // All ratings for a worker
    
    uint256 nextTaskId = 0;

    // Create task
    function createTask(address worker, uint256 amount) external payable {
        require(msg.value >= amount, "Insufficient payment");
        
        tasks[nextTaskId] = Task({
            requester: msg.sender,
            worker: worker,
            amount: amount,
            completed: false,
            rating: 0
        });
        
        nextTaskId++;
    }

    // Worker completes task
    function completeTask(uint256 taskId) external {
        Task storage task = tasks[taskId];
        require(msg.sender == task.worker, "Not task worker");
        require(!task.completed, "Already completed");
        
        task.completed = true;
        
        // Update reputation
        reputation[msg.sender].tasksCompleted += 1;
        reputation[msg.sender].totalEarned += task.amount;
        
        // Transfer payment
        payable(msg.sender).transfer(task.amount);
        
        emit TaskCompleted(msg.sender, taskId, block.timestamp, task.amount);
    }

    // Requester rates completed task
    function rateTask(uint256 taskId, uint8 rating) external {
        Task storage task = tasks[taskId];
        require(msg.sender == task.requester, "Not requester");
        require(task.completed, "Task not completed");
        require(rating >= 1 && rating <= 5, "Rating 1-5 only");
        
        task.rating = rating;
        
        // Update worker reputation
        address worker = task.worker;
        ratings[worker].push(rating);
        
        uint256 sum = 0;
        for (uint256 i = 0; i < ratings[worker].length; i++) {
            sum += ratings[worker][i];
        }
        reputation[worker].avgRating = uint8(sum / ratings[worker].length);
        reputation[worker].reviewCount = ratings[worker].length;
        
        emit ReviewPosted(msg.sender, worker, taskId, rating, "");
    }

    // Query reputation (public, anyone can call)
    function getReputation(address worker) 
        external 
        view 
        returns (ReputationMetrics memory) 
    {
        return reputation[worker];
    }

    // Get all ratings for a worker
    function getRatings(address worker) 
        external 
        view 
        returns (uint8[] memory) 
    {
        return ratings[worker];
    }
}
```

### How a Web App Uses This

```javascript
// Fetch worker reputation from blockchain
const workerAddress = "0x1234...";
const contract = new ethers.Contract(
  REPUTATION_CONTRACT_ADDRESS,
  CONTRACT_ABI,
  provider
);

const metrics = await contract.getReputation(workerAddress);

// Display on website
document.querySelector("#tasks-completed").textContent = metrics.tasksCompleted;
document.querySelector("#avg-rating").textContent = 
  (metrics.avgRating / 10).toFixed(1) + "/5";
document.querySelector("#total-earned").textContent = 
  ethers.formatEther(metrics.totalEarned) + " ETH";
```

The web app doesn't store anything. It **queries** the contract in real-time. Multiple web apps can exist, all showing the same trustworthy data.

---

## More Complex: Staking-Based Reputation

What if users can **stake** to earn reputation? Higher stake = higher reputation (assuming you have skin in the game).

```solidity
pragma solidity ^0.8.24;

contract StakedReputation {
    mapping(address => uint256) public staked;
    mapping(address => uint256) public reputation;

    // Stake ETH to participate
    function stake() external payable {
        require(msg.value > 0, "Stake > 0");
        staked[msg.sender] += msg.value;
        reputation[msg.sender] += msg.value;  // 1 wei staked = 1 rep
    }

    // Unstake (only if in good standing)
    function unstake(uint256 amount) external {
        require(staked[msg.sender] >= amount, "Insufficient stake");
        staked[msg.sender] -= amount;
        payable(msg.sender).transfer(amount);
    }

    // Slashing (external oracle or DAO votes to penalize)
    function slash(address user, uint256 amount) external onlyOracle {
        reputation[user] = (reputation[user] > amount) 
            ? reputation[user] - amount 
            : 0;
        staked[user] = (staked[user] > amount) 
            ? staked[user] - amount 
            : 0;
    }

    function getReputation(address user) 
        external 
        view 
        returns (uint256) 
    {
        return reputation[user];
    }
}
```

**Use case:** Network participants (validators, curators, content reviewers) stake to earn the right to participate. Misbehavior → slash → reputation drop.

---

## Real-World Examples

### 1. **DeSci Reputation** (Decentralized Science)
- Researchers publish findings on-chain (IPFS hash + metadata)
- Peer reviewers stake tokens to validate papers
- Reviews recorded on-chain; reviewer builds reputation
- High-reputation reviewers' endorsements carry more weight
- Funding decisions weighted by reputation

### 2. **Marketplace Trust** (Opensea for Services)
- Freelancer: posts availability, completes jobs, gets rated
- Client: posts budget, rates work, earns reputation for fair dealing
- Both sides' histories are **transparent and immutable**
- Third-party arbiters (DAO members) can dispute and slash bad actors

### 3. **DAO Voting Power**
- Reputation ≠ token holding. Reputation = past contributions (work, tests, reviews)
- DAO members earn voting power by doing work, not by buying tokens
- Whale with 1M tokens but no contributions = no voting power
- Small contributor with 100 tasks completed = more voting power

### 4. **On-Chain Certification**
- Platform issues **SBT (Soulbound Token)** for completed course modules
- Each SBT logs difficulty level, completion date, test score
- Employer queries: "Does this developer have 5 SBTs in cryptography?"
- No centralized credential database; credential is a token you own

---

## Key Design Decisions

### Storage vs. Immutability

**Option 1: Store all history**
```solidity
mapping(address => Event[]) public history;  // Every action ever
```
Pros: Transparent, auditable, composable  
Cons: Gas expensive, storage bloat

**Option 2: Store only aggregates**
```solidity
mapping(address => uint256) public totalTasks;
mapping(address => uint256) public avgRating;
```
Pros: Cheap, efficient  
Cons: Can't audit the historical record; have to trust the aggregation logic

**Hybrid:** Emit events (logged on-chain, queryable but cheaper) + store aggregates.

### Who Can Update Reputation?

**Centralized (single oracle):**
```solidity
function updateReputation(address user, uint256 newScore) external onlyOwner {}
```
Fast, cheap. But defeats the purpose—you're trusting the owner.

**Decentralized (DAO votes or multiple oracles):**
```solidity
function updateReputation(address user, uint256 newScore) external {
    require(hasMultiSigApproval(), "Need 3-of-5 approval");
}
```
Trustless but slow and complex.

**Peer-driven (anyone can report, but with stake):**
```solidity
function challenge(address user) external payable {
    require(msg.value > minBond, "Post bond");
    // DAO votes; loser loses bond
}
```
Incentive-compatible, decentralized.

### Temporal Decay

Reputation from 5 years ago shouldn't count as much as last month. But Solidity can't trigger time-based updates automatically. Options:

1. **Off-chain compute**: Web3.js fetches history, computes decayed score, displays it
2. **On-demand decay**: User calls `updateReputation()`, contract recalculates with decay
3. **Staking decay**: Reputation decreases if you stop staking

---

## Connecting to Web Apps

### The Architecture

```
┌─────────────────┐
│  Smart Contract │ ← Records all reputation events
│  (on Ethereum)  │   (immutable, public)
└────────┬────────┘
         │ Events emitted
         ├─→ Indexer (The Graph, Alchemy)
         │   Listens to blockchain, stores in queryable database
         │
         ├─→ Web App Backend
             ├─ Queries indexer for historical data
             ├─ Caches computed metrics
             └─ Serves REST API to frontend
         │
         └─→ Frontend (React/Vue)
             ├─ Fetches live metrics from backend
             ├─ Or queries contract directly (slow)
             └─ Displays leaderboards, profiles, ratings
```

### Example: The Graph Query

```graphql
{
  workers(first: 10, orderBy: reputation, orderDirection: desc) {
    id
    tasksCompleted
    avgRating
    totalEarned
  }
}
```

The Graph automatically indexes your contract's events and provides GraphQL queries. Your web app doesn't need to understand Solidity; it just queries like any API.

---

## Why This Matters

Traditional systems:
- **Single point of failure**: Platform goes down → reputation gone
- **Rent extraction**: Intermediary takes 30% because they control the trust layer
- **Censorship**: They can delete your account or reviews
- **Data lock-in**: Your reputation can't move to another platform

On-chain reputation:
- **Portable**: Your reputation exists independently of any platform
- **Composable**: Any contract can read it; any website can display it
- **Permanent**: Can't be deleted (though it can decay)
- **Transparent**: Calculation is verifiable code, not a black-box algorithm

The power isn't in payments (DeFi can handle that). The power is in **trustless coordination**: systems where participants can verify each other's track records without needing a central authority to mediate.

---

## Gotchas

1. **Oracle problem**: Who feeds in real-world data? If your metric depends on "was the work actually good?", you need off-chain judges. Smart contracts alone can't judge subjective quality.

2. **Sybil attacks**: Without cost, anyone can create 1000 accounts and fake reputation. Mitigate with staking (cost to participate) or SBTs (hard to forge).

3. **Gas costs**: Storing all history is expensive. Aggregate and use events instead.

4. **Immutability**: Once bad data is on-chain, it's permanent. Be careful with initialization.

5. **Privacy**: All reputation is public. Anonymity and reputation are in tension.

---

## Conclusion

Smart contracts enable **decentralized reputation**: a trustworthy record of actions without intermediaries. Not just for lending. For marketplaces, certification, voting power, community trust, scientific peer review, and any system that currently relies on a centralized authority to maintain credibility.

The contract is the database. The web app is the UI. Both are decoupled.
