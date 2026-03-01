@def title = "Solidity Design Patterns: Architecture Beyond Syntax"
@def published = "1 March 2026"
@def tags = ["solidity", "code"]

# Solidity Design Patterns: Architecture Beyond Syntax

**TL;DR:** Solidity has a small number of fundamental design patterns that show up everywhere. This post catalogs them: how they work, why they're necessary, and when to use each one. These patterns address real constraints of the EVM (storage cost, reentrancy, upgrades, access control). Master these and you'll recognize the structure of any contract.

---

## Part 1: What is a Design Pattern?

A design pattern is a **reusable solution to a recurrent problem** in a specific context. In Solidity, the context is:

- **Immutability** — deployed code cannot be changed (usually)
- **Expensive storage** — state writes cost 20,000 gas each
- **Single-threaded execution** — transactions are atomic, no concurrent calls
- **No private data** — blockchain is transparent
- **Determinism** — same input always produces same output (or revert)

Given these constraints, certain patterns emerge naturally. There are roughly **7 core patterns** that compose into larger architectures.

---

## Pattern 1: Access Control (The Guard)

**Problem:** You want to restrict who can call a function (e.g., only the owner).

**Solution:** Check `msg.sender` and revert if unauthorized.

```solidity
contract Owned {
    address public owner;
    
    constructor() {
        owner = msg.sender;  // Deployer is owner
    }
    
    modifier onlyOwner() {
        require(msg.sender == owner, "Only owner can call");
        _;
    }
    
    function adminAction() public onlyOwner {
        // This function only executes if msg.sender == owner
    }
}
```

**Formally:** We're restricting the transition function to a subset of callers:

$$\mathcal{T}_{\text{restricted}}(s, m) = \begin{cases}
\mathcal{T}(s, m) & \text{if } \text{authorized}(m) \\
\bot & \text{otherwise}
\end{cases}$$

**Why a modifier?** It's reusable across many functions and keeps code DRY.

**Variations:**

```solidity
// Role-based access
mapping(address => bool) isAdmin;

modifier onlyAdmin() {
    require(isAdmin[msg.sender], "Not admin");
    _;
}

// Time-based access
modifier onlyAfter(uint256 time) {
    require(block.timestamp >= time, "Too early");
    _;
}

// Multi-sig (multiple parties required)
modifier requires2of3(address a, address b, address c) {
    require(msg.sender == a || msg.sender == b || msg.sender == c);
    _;
}
```

---

## Pattern 2: State Machine (The Lifecycle)

**Problem:** A contract should move through distinct states (e.g., "pending" → "active" → "completed"), and only certain actions are valid in each state.

**Solution:** Encode the state as an enum, and guard functions with modifiers checking the current state.

```solidity
contract Auction {
    enum State { Pending, Active, Closed }
    State public state = State.Pending;
    
    modifier inState(State s) {
        require(state == s, "Invalid state");
        _;
    }
    
    function start() public inState(State.Pending) {
        state = State.Active;
    }
    
    function bid(uint256 amount) public inState(State.Active) {
        // Only works if auction is active
    }
    
    function finalize() public inState(State.Active) {
        state = State.Closed;
    }
}
```

**Why?** Prevents calling functions out of order (e.g., bidding in a closed auction).

**Formally:** Each state $s \in \mathcal{S}$ (the state space) has a valid set of transitions:

$$\mathcal{T}_s: \{f \in \mathcal{F} : \text{enabled}(f, s)\} \to \mathcal{S}$$

The modifier ensures only enabled functions can execute in state $s$.

---

## Pattern 3: Checks-Effects-Interactions (CEI)

**Problem:** When you call external contracts, they can re-enter your function and cause bugs (reentrancy).

**Solution:** Order operations carefully: (1) check preconditions, (2) modify state, (3) call external contracts.

```solidity
// ❌ VULNERABLE (wrong order)
function withdraw(uint256 amount) public {
    uint256 balance = balances[msg.sender];
    require(balance >= amount);
    
    // ⚠️  Attacker can re-enter here!
    (bool success, ) = msg.sender.call{value: amount}("");
    require(success);
    
    // This runs AFTER the call, too late
    balances[msg.sender] -= amount;
}

// ✅ SAFE (CEI order)
function withdraw(uint256 amount) public {
    // 1. Checks
    uint256 balance = balances[msg.sender];
    require(balance >= amount);
    
    // 2. Effects
    balances[msg.sender] -= amount;  // Update state FIRST
    
    // 3. Interactions
    (bool success, ) = msg.sender.call{value: amount}("");
    require(success);
}
```

**Why?** When you call `msg.sender.call()`, the recipient's fallback function runs. If that function calls `withdraw()` again, it sees the *updated* state (balance already decremented), so it can't withdraw twice.

**Formally:** We're ensuring that external calls see state *after* the operation:

$$\text{State}' \leftarrow \text{update}(\text{State}) \quad \text{then} \quad \text{externalCall}(\text{State}')$$

not:

$$\text{externalCall}(\text{State}) \quad \text{then} \quad \text{State}' \leftarrow \text{update}(\text{State})$$

---

## Pattern 4: Pull Over Push

**Problem:** Sending ether to multiple parties is gas-inefficient and error-prone (what if a call fails?).

**Solution:** Let each party withdraw their own funds (pull pattern) instead of sending (push pattern).

```solidity
// ❌ Push pattern (inefficient, risky)
contract Refunder {
    address[] public refundees;
    mapping(address => uint256) amounts;
    
    function refundAll() public {
        for (uint256 i = 0; i < refundees.length; i++) {
            address refundee = refundees[i];
            (bool success, ) = refundee.call{value: amounts[refundee]}("");
            if (!success) {
                // What do we do if one fails? Entire transaction reverts?
                revert("Refund failed");
            }
        }
    }
}

// ✅ Pull pattern (efficient, modular)
contract PullRefunder {
    mapping(address => uint256) public refunds;
    
    function claim() public {
        uint256 amount = refunds[msg.sender];
        require(amount > 0);
        
        refunds[msg.sender] = 0;  // Prevent double-claim (CEI order)
        (bool success, ) = msg.sender.call{value: amount}("");
        require(success);
    }
}
```

**Why?** 
- Each party is responsible for their own withdrawal (fault-isolated)
- No need to iterate over all parties (saves gas, avoids loops)
- Each call is independent (one failure doesn't affect others)

---

## Pattern 5: Tight Variable Packing

**Problem:** Storage is expensive (20,000 gas per write). If you have many small state variables, you're wasting space.

**Solution:** Pack multiple small types into a single 256-bit storage slot.

```solidity
// ❌ Inefficient (8 storage slots)
contract Inefficient {
    uint64 a;   // Slot 0 (wastes 192 bits)
    uint64 b;   // Slot 1 (wastes 192 bits)
    uint64 c;   // Slot 2 (wastes 192 bits)
    uint64 d;   // Slot 3 (wastes 192 bits)
}

// ✅ Efficient (1 storage slot)
contract Efficient {
    uint64 a;
    uint64 b;
    uint64 c;
    uint64 d;
    // Packed into slot 0: [d (64 bits) | c (64 bits) | b (64 bits) | a (64 bits)]
}
```

**How Solidity packs:** Variables are assigned to storage slots sequentially. Multiple variables fit into one slot (256 bits) if their combined size is ≤ 256 bits.

```solidity
// Careful ordering matters
contract Optimized {
    bool flag1;          // Slot 0: 8 bits
    bool flag2;          // Slot 0: 16 bits total
    uint64 count;        // Slot 0: 80 bits total
    address owner;       // Slot 1: 160 bits (doesn't fit in slot 0)
    uint256 balance;     // Slot 2: 256 bits
}
```

**Gas savings:** Writing to a packed slot costs the same as writing to an unpacked slot (20,000 gas), but you're storing 4x more data.

---

## Pattern 6: Upgradeability (Proxy Pattern)

**Problem:** Contracts are immutable. If you deploy buggy code, it's stuck on the blockchain forever.

**Solution:** Use a proxy: a simple contract that delegates calls to an upgradeable implementation contract.

```solidity
// The implementation contract (can change)
contract CounterV1 {
    uint256 public count = 0;
    
    function increment() public {
        count += 1;
    }
}

// The proxy contract (permanent address)
contract CounterProxy {
    address public implementation;
    address public owner;
    
    constructor(address impl) {
        implementation = impl;
        owner = msg.sender;
    }
    
    // Delegate all calls to implementation
    fallback() external payable {
        address impl = implementation;
        assembly {
            // Copy calldata, delegate call, return
            let ptr := mload(0x40)
            calldatacopy(ptr, 0, calldatasize())
            let result := delegatecall(gas(), impl, ptr, calldatasize(), 0, 0)
            returndatacopy(ptr, 0, returndatasize())
            switch result
            case 0 { revert(ptr, returndatasize()) }
            default { return(ptr, returndatasize()) }
        }
    }
    
    // Upgrade to new implementation
    function upgrade(address newImpl) public {
        require(msg.sender == owner);
        implementation = newImpl;
    }
}
```

**How it works:**
1. You deploy `CounterProxy` with `CounterV1`'s address
2. Users call `CounterProxy`, which delegates to `CounterV1`
3. If you find a bug, deploy `CounterV2` and call `upgrade(addressOfCounterV2)`
4. Users' calls now delegate to `CounterV2` (same proxy address, new logic)

**Trade-off:** Proxy adds complexity and a delegatecall (more gas), but enables upgrades.

**Formally:** The proxy is a layer of indirection:

$$\text{call}(\text{proxy}) \to \text{delegatecall}(\text{implementation}) \to \text{result}$$

---

## Pattern 7: Allowance (Approvals for Third Parties)

**Problem:** You have a token, and you want to let a third party spend some of your tokens without directly transferring all your authority to them.

**Solution:** Use an allowance mechanism. You approve an amount, and the third party can spend up to that amount.

```solidity
contract ERC20Token {
    mapping(address => uint256) public balances;
    mapping(address => mapping(address => uint256)) public allowance;
    
    // Transfer directly
    function transfer(address to, uint256 amount) public {
        require(balances[msg.sender] >= amount);
        balances[msg.sender] -= amount;
        balances[to] += amount;
    }
    
    // Approve someone else to spend your tokens
    function approve(address spender, uint256 amount) public {
        allowance[msg.sender][spender] = amount;
    }
    
    // Third party can transfer on your behalf (up to allowance)
    function transferFrom(address from, address to, uint256 amount) public {
        require(balances[from] >= amount);
        require(allowance[from][msg.sender] >= amount);
        
        balances[from] -= amount;
        balances[to] += amount;
        allowance[from][msg.sender] -= amount;  // Deduct from allowance
    }
}
```

**Why?** If a smart contract wants to spend your tokens, you approve it for a specific amount rather than giving it unlimited access.

**Flow:**
1. Alice calls `approve(SpenderContract, 100)` — permits SpenderContract to spend up to 100 of her tokens
2. SpenderContract calls `transferFrom(Alice, Recipient, 50)` — transfers 50 of Alice's tokens
3. Alice's allowance for SpenderContract is now 50

**Formally:** We're creating a delegation:

$$\text{allowance}[\text{owner}][\text{spender}] = \text{amount}$$

and the spender can only transfer up to this amount on behalf of the owner.

---

## Pattern 8: Expiry/Deadline (Time Bounds)

**Problem:** Users sign transactions that will be executed later. By the time it's executed, circumstances may have changed (price moved, conditions expired). You want to let users set a deadline.

**Solution:** Add a `deadline` parameter. Revert if the function is called after the deadline.

```solidity
contract DEXSwap {
    function swap(
        address tokenIn,
        address tokenOut,
        uint256 amountIn,
        uint256 minAmountOut,
        uint256 deadline  // User specifies deadline
    ) public {
        require(block.timestamp <= deadline, "Swap expired");
        
        uint256 amountOut = getSwapAmount(tokenIn, tokenOut, amountIn);
        require(amountOut >= minAmountOut, "Slippage too high");
        
        // Execute swap...
    }
}
```

**Why?** If a user signs a swap, but it sits in the mempool for hours while prices move, they might get a bad price. The deadline ensures it won't execute if conditions have changed too much.

**Formally:** We're checking:

$$\text{block.timestamp} \leq \text{deadline}$$

If false, the transaction reverts. This is a time-based guard, like the access control pattern.

---

## Pattern 9: Nonce (Replay Protection)

**Problem:** If you broadcast a signed transaction to the blockchain, an attacker can replay it on a different chain or fork, causing unintended consequences.

**Solution:** Include a nonce (sequential number) in the transaction. Each signature is valid only for that specific nonce.

```solidity
contract PermitToken {
    mapping(address => uint256) public nonce;
    
    function permit(
        address owner,
        address spender,
        uint256 amount,
        uint256 deadline,
        uint8 v,
        bytes32 r,
        bytes32 s
    ) public {
        require(block.timestamp <= deadline);
        
        bytes32 message = keccak256(abi.encode(
            owner,
            spender,
            amount,
            nonce[owner]++,  // Include current nonce
            deadline
        ));
        
        address recovered = ecrecover(message, v, r, s);
        require(recovered == owner);
        
        // Execute permit...
    }
}
```

**Why?** Once a nonce is used, that signature is no longer valid (even if someone replays it). The next transaction must use `nonce + 1`.

**Formally:** Each signature encodes:

$$\text{signature} = \text{sign}_{\text{private}}(\text{message}(\text{nonce}, \text{data}))$$

Once a nonce is consumed, that exact signature can never be replayed (different message → different signature required).

---

## Pattern 10: Commit-Reveal (Privacy)

**Problem:** Blockchain is transparent. If you commit to something (like a bid in an auction), everyone sees it immediately. You want to hide your action until later.

**Solution:** Split into two phases: (1) commit a hash of your data, (2) reveal the data. The hash was already on-chain, so you can't change it.

```solidity
contract SecretAuction {
    mapping(address => bytes32) commits;
    mapping(address => uint256) reveals;
    
    // Phase 1: Submit hash of your bid (keccak256(amount, salt))
    function commit(bytes32 bidHash) public {
        commits[msg.sender] = bidHash;
    }
    
    // Phase 2: Reveal your actual bid
    function reveal(uint256 amount, uint256 salt) public {
        bytes32 computedHash = keccak256(abi.encode(amount, salt));
        require(computedHash == commits[msg.sender], "Invalid reveal");
        
        // Now auction logic sees your bid
        reveals[msg.sender] = amount;
    }
}
```

**Why?** No one knew your bid until you revealed it (they only saw the hash). But you can't lie about it (hash was committed).

**Formally:** We're computing:

$$\text{commit} = \text{hash}(\text{amount}, \text{salt})$$

and later verifying:

$$\text{hash}(\text{reveal}, \text{salt}) = \text{commit}$$

The hash is cryptographically binding.

---

## Pattern 11: Oracle (External Data)

**Problem:** Smart contracts can't fetch data from the internet directly. But many contracts need external data (prices, weather, random numbers).

**Solution:** Use an oracle—a trusted third party that submits data on-chain.

```solidity
contract PriceFeed {
    address public oracle;
    uint256 public lastPrice;
    uint256 public lastUpdate;
    
    constructor(address _oracle) {
        oracle = _oracle;
    }
    
    // Oracle submits price data
    function setPrice(uint256 price) public {
        require(msg.sender == oracle, "Only oracle");
        lastPrice = price;
        lastUpdate = block.timestamp;
    }
    
    // Contract uses the price
    function getValue(uint256 tokenAmount) public view returns (uint256) {
        require(block.timestamp - lastUpdate < 1 day, "Price stale");
        return tokenAmount * lastPrice;
    }
}
```

**Why?** Blockchains are deterministic and isolated (can't fetch URLs). Oracles are the bridge to external data.

**Trade-off:** Oracles introduce trust. You're trusting them to submit accurate data.

---

## Pattern 12: Multi-Signature (Multi-Sig)

**Problem:** You want to distribute authority. No single person should be able to drain the contract; multiple approvals required.

**Solution:** Require M-of-N signatures before executing sensitive operations.

```solidity
contract MultiSig {
    address[] public signers;
    mapping(bytes32 => mapping(address => bool)) public approvals;
    
    modifier onlySigners() {
        require(isSigner(msg.sender));
        _;
    }
    
    function approve(bytes32 txHash) public onlySigners {
        approvals[txHash][msg.sender] = true;
    }
    
    function execute(
        address target,
        uint256 value,
        bytes memory data
    ) public {
        bytes32 txHash = keccak256(abi.encode(target, value, data));
        
        // Count approvals
        uint256 approvalCount = 0;
        for (uint256 i = 0; i < signers.length; i++) {
            if (approvals[txHash][signers[i]]) {
                approvalCount++;
            }
        }
        
        require(approvalCount >= 2, "Need 2 approvals");
        
        // Execute
        (bool success, ) = target.call{value: value}(data);
        require(success);
    }
}
```

**Why?** Distributes risk. No single compromised key can steal funds.

---

## How These Patterns Compose

Real contracts use **multiple patterns together**:

```solidity
contract TokenSale {
    // Pattern 1: Access Control
    address public owner;
    modifier onlyOwner() { require(msg.sender == owner); _; }
    
    // Pattern 2: State Machine
    enum State { Setup, Running, Finalized }
    State public state = State.Setup;
    modifier inState(State s) { require(state == s); _; }
    
    // Pattern 3: Allowance (from ERC20)
    mapping(address => uint256) public allowance;
    
    // Pattern 4: Time Bounds
    uint256 public endTime;
    modifier beforeDeadline() { require(block.timestamp < endTime); _; }
    
    // Pattern 3: CEI order for withdrawals
    function withdraw() public {
        // Checks
        require(state == State.Finalized);
        
        // Effects
        uint256 amount = myFunds[msg.sender];
        myFunds[msg.sender] = 0;
        
        // Interactions
        (bool success, ) = msg.sender.call{value: amount}("");
        require(success);
    }
}
```

---

## Anti-Patterns to Avoid

### 1. Timestamping Assumptions

❌ Bad:
```solidity
require(block.timestamp == someExactTime);  // Will never match!
```

✅ Good:
```solidity
require(block.timestamp >= startTime && block.timestamp <= endTime);
```

Blocks are mined at irregular intervals. Use ranges, not exact values.

### 2. Delegatecall to Untrusted Code

❌ Bad:
```solidity
address userCode = getUserProvidedAddress();
(bool success, ) = userCode.delegatecall(data);  // Gives full access to your state!
```

`delegatecall` runs the code with your contract's storage. Never delegatecall to untrusted code.

### 3. Raw Call Without Checking Return

❌ Bad:
```solidity
msg.sender.call{value: amount}("");  // Ignores return value!
```

✅ Good:
```solidity
(bool success, ) = msg.sender.call{value: amount}("");
require(success);
```

### 4. Using `tx.origin` for Authorization

❌ Bad:
```solidity
require(tx.origin == owner);  // Can be spoofed via intermediary contract
```

✅ Good:
```solidity
require(msg.sender == owner);
```

`tx.origin` is the original EOA, not the caller. Use `msg.sender`.

---

## Summary: The Patterns at a Glance

| Pattern | Problem | Solution |
|---------|---------|----------|
| **Access Control** | Who can call this? | Check `msg.sender` with modifiers |
| **State Machine** | Enforce ordering | Encode state enum, guard functions |
| **CEI** | Reentrancy attacks | Checks, Effects, Interactions order |
| **Pull over Push** | Batch transfer failures | Let users withdraw their own funds |
| **Tight Packing** | Storage cost | Pack multiple small types in one slot |
| **Proxy** | Fix bugs after deploy | Delegate to upgradeable implementation |
| **Allowance** | Third-party spending limits | Approve then transferFrom |
| **Expiry** | Stale data | Add deadline parameter |
| **Nonce** | Replay attacks | Increment nonce per signature |
| **Commit-Reveal** | Hide data until reveal | Hash first, verify later |
| **Oracle** | External data | Trusted third party submits data |
| **Multi-Sig** | Distributed authority | Require M-of-N approvals |

---

## When to Use Which Pattern

**Building a token?** → Allowance + Transfer + Access Control

**Building an auction?** → State Machine + Commit-Reveal + Deadline

**Building a DEX?** → Pull pattern + Oracles + Expiry + Multi-Sig

**Building a vault?** → Access Control + CEI + Multi-Sig

These patterns are **composable and reusable**. Most contracts are a combination of 3–5 of them.

---

## Next: Understanding Real Contracts

Now that you know:
1. **Syntax fundamentals** (types, functions, control flow)
2. **Design patterns** (access control, state machines, reentrancy guards)

You're ready to read real contracts (Uniswap, Aave, OpenZeppelin). Each will be recognizable as a combination of these patterns.

For example, Uniswap's core contract uses:
- Access Control (owner functions)
- Allowance (ERC20 approve/transferFrom)
- CEI (to prevent reentrancy)
- State Machine (swap states)
- Multi-Sig (governance)

Recognize the patterns, and the code becomes readable.
