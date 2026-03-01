@def title = "Solidity Syntax & Fundamentals: The Plumbing"
@def published = "1 March 2026"
@def tags = ["solidity", "code"]

# Solidity Syntax & Fundamentals: The Plumbing

**TL;DR:** This post covers the *syntax primitives* of Solidity—the building blocks from which all contracts are constructed. No design patterns, no architecture—just the language itself: types, values, control flow, state, and function signatures. Think of it as learning the grammar before writing essays.

---

## Part 1: Core Idea—The State Machine

Before any syntax, understand what Solidity *is*: it's a language for programming **state machines** that live on a blockchain.

Formally, a Solidity contract is a tuple:

$$\mathcal{C} = (\Sigma, s_0, \mathcal{T})$$

where:
- $\Sigma$ = the **state space** (all possible values of state variables)
- $s_0 \in \Sigma$ = the **initial state** (set in constructor)
- $\mathcal{T}: \Sigma \times \mathcal{M} \to \Sigma \cup \{\bot\}$ = the **transition function** (functions map messages to new states, or $\bot$ for revert)

When someone calls a function, they send a message $m \in \mathcal{M}$ to the contract. The contract's state transitions according to $\mathcal{T}$, or reverts if $\mathcal{T}(s, m) = \bot$.

```solidity
contract Counter {
    uint256 public count = 0;  // Part of Σ; s₀ = (count: 0)
    
    function increment() public {  // Function that defines T
        count += 1;  // State transition
    }
}
```

**Everything** in Solidity—types, functions, modifiers, events—is machinery for defining and managing this state machine.

---

## Part 2: Types—The Values in Σ

### Primitives

Solidity's primitive types form a finite, ordered set of values. Each has a fixed size (in bits) and semantics (how operations work on them).

#### Unsigned Integers

$$\text{uint}\langle n \rangle : n \in \{8, 16, 32, \ldots, 256\}, n \equiv 0 \pmod{8}$$

(Read: $n$ is divisible by 8, i.e., a multiple of 8: 8, 16, 24, ..., 256 bits)

Maps to integers $\{0, 1, \ldots, 2^n - 1\}$.

```solidity
uint8 small = 255;        // Range: [0, 255]
uint256 large = 2**256-1; // Range: [0, 2^256 - 1]
uint x = 5;               // uint ≡ uint256 (default)
```

**Overflow behavior (Solidity 0.8+):** Addition/subtraction checks for overflow; reverts if $x + y > 2^n - 1$.

```solidity
uint8 x = 255;
x += 1;  // Reverts! (255 + 1 > 255)
```

#### Signed Integers

$$\text{int}\langle n \rangle : n \in \{8, 16, 32, \ldots, 256\}, n \equiv 0 \pmod{8}$$

(Read: $n$ is divisible by 8)

Maps to integers $\{-2^{n-1}, \ldots, -1, 0, 1, \ldots, 2^{n-1} - 1\}$.

```solidity
int8 x = -128;   // Range: [-128, 127]
int256 y = -1;
```

#### Fixed-Point Numbers (Rarely Used)

$$\text{ufixed}\langle m \times n \rangle, \text{fixed}\langle m \times n \rangle$$

where $m + n = 256$. Represents rational numbers: the $n$ least-significant bits are fractional.

```solidity
ufixed128x128 price = 1.5; // Not commonly used; beware precision loss
```

#### Boolean

$$\mathbb{B} = \{\text{true}, \text{false}\}$$

```solidity
bool flag = true;
bool result = (x > 5) && (y < 10);  // Result of logical operations
```

#### Bytes (Fixed and Dynamic)

**Fixed-size:**

$$\text{bytes}\langle n \rangle : n \in \{1, 2, \ldots, 32\}$$

A fixed-size byte array. Stored as-is (no implicit padding).

```solidity
bytes32 hash = keccak256("hello");  // Result of hash function
bytes4 selector = 0x12345678;       // Function selector
bytes1 x = 0xFF;                    // Single byte
```

**Dynamic:**

$$\text{bytes} = \text{sequence of bytes of arbitrary length}$$

More on this under "Reference Types" below.

#### Address

$$\text{address} \cong \{0, 1\}^{160}$$

A 160-bit (20-byte) Ethereum address. Represents an account (externally owned or contract).

```solidity
address user = 0x742d35Cc6634C0532925a3b844Bc9e7595f42bE3;
address zero = address(0);  // Special: zero address (null)

address payable wallet = payable(user);  // Can receive ether
```

**Two subtypes:**
- `address` — read-only
- `address payable` — can send ether to it

### Composite Types: Structs

Group multiple fields together (like a struct in C):

```solidity
struct User {
    string name;
    uint256 age;
    address wallet;
}

User alice = User("Alice", 30, 0x123...);
uint256 aliceAge = alice.age;  // Field access
```

Formally, a struct is a tuple:

$$\mathcal{S} = (f_1: T_1, f_2: T_2, \ldots, f_k: T_k)$$

where each $f_i$ is a field name and $T_i$ is its type.

### Enums

A set of named constants:

```solidity
enum Status { Pending, Active, Completed }

Status currentStatus = Status.Active;
uint8 encoded = uint8(Status.Pending);  // Encodes to 0
```

Formally, an enum is a finite set: $\mathcal{E} = \{e_0, e_1, \ldots, e_{n-1}\}$, encoded as integers $\{0, 1, \ldots, n-1\}$.

### Reference Types: Arrays

#### Fixed-Size Arrays

$$T[n] = \{(a_0, a_1, \ldots, a_{n-1}) : a_i \in T\}$$

```solidity
uint256[3] fixedArray = [1, 2, 3];
address[10] addresses;

fixedArray[0] = 100;  // Index 0
addresses.length;     // Always 10
```

#### Dynamic Arrays

$$T[\,] = \bigcup_{n \geq 0} T[n]$$

Variable-length arrays. Can grow/shrink.

```solidity
uint256[] dynamicArray;

dynamicArray.push(10);      // Append
dynamicArray.pop();         // Remove last
dynamicArray.length;        // Current length
uint256 first = dynamicArray[0];  // Access
```

#### Mappings

$$T_1 \mapsto T_2 = T_1 \to T_2 \cup \{\bot\}$$

A key-value store (like a hash table). Maps keys of type $T_1$ to values of type $T_2$. Unset keys map to zero/default value.

```solidity
mapping(address => uint256) balances;  // Address → balance

balances[user] = 100;           // Set
uint256 bal = balances[user];   // Get (returns 0 if unset)
delete balances[user];          // Reset to default
```

**Key insight:** Mappings have no `.length`, no iteration, no default value—you get the type's zero value if a key is unset. This is intentional for gas efficiency.

**Allowed key types:** Any value type (not arrays, structs, or other mappings).

#### Strings

$$\text{string} = \text{sequence of UTF-8 characters}$$

Dynamically sized byte sequence (really `bytes`).

```solidity
string greeting = "hello";
bytes data = bytes(greeting);  // Convert to bytes
uint256 len = bytes(greeting).length;  // Get length
```

**Caution:** No native string operations (concatenation is expensive). Usually convert to `bytes` for manipulation.

---

## Part 3: State Variables—The Persistent $\Sigma$

State variables live on the blockchain permanently. They comprise the contract's state $\Sigma$.

```solidity
contract Storage {
    uint256 public count = 0;           // Public state variable
    address private owner = msg.sender; // Private (not readable directly)
    mapping(address => uint256) ledger; // State variable (mapping)
}
```

### Access Modifiers

- **`public`** — accessible from anywhere (outside, within contract, via calls). Automatically generates a getter function.
- **`internal`** — accessible within this contract and subcontracts only.
- **`private`** — accessible within this contract only (not even subcontracts).

**Note:** `private` and `internal` don't mean "secret"—blockchain is transparent. Anyone can read all state by querying the chain.

### Initial Values

```solidity
uint256 x = 10;              // Set at compile time
address deployer = msg.sender; // Set at deploy time (constructor runs)
```

Uninitialized state variables get the **zero value** of their type:
- `uint256`: 0
- `bool`: false
- `address`: 0x0
- `string`/`bytes`: ""
- Arrays: empty array
- Mappings: empty map

---

## Part 4: Functions—The Transition Function $\mathcal{T}$

A function defines how state changes in response to a call.

> **How does a dApp trigger a function call?**
> 
> 1. **User clicks a button in the React app** (e.g., "Withdraw 1 ETH")
> 2. **dApp frontend creates a transaction** using ethers.js or Web3.js:
>    ```javascript
>    const tx = await contract.withdraw(ethers.parseEther("1"));
>    ```
> 3. **User signs the transaction** with MetaMask (or Ledger, etc.)
> 4. **Transaction is sent to the blockchain** (JSON-RPC to an Ethereum node)
> 5. **Miners/validators include it in a block** (usually within seconds)
> 6. **EVM executes the contract function** with your signed message
> 7. **State updates** (your balance decreases, the contract's balance decreases)
> 8. **dApp listens for the block confirmation** and updates the UI
>
> **Key point:** You're not "calling" the function directly like `contract.withdraw()` in Node.js. You're signing a **transaction** that tells the blockchain "execute this function with these arguments." The blockchain then executes it deterministically on every node. This is why it costs gas and takes 12+ seconds.

### Basic Syntax

```solidity
function myFunction(uint256 x, address to) public returns (uint256, bool) {
    // Function body
    return (42, true);
}
```

**Signature:** $(T_1, T_2, \ldots, T_k) \to (R_1, R_2, \ldots, R_m)$

where $T_i$ are parameter types and $R_j$ are return types.

### Visibility Modifiers

The **four levels** control where a function can be called from:

| Modifier | Outside | Within Contract | Subcontracts | Generates External Interface |
|----------|---------|-----------------|--------------|------|
| **`public`** | ✓ | ✓ | ✓ | Yes (auto getter) |
| **`internal`** | ✗ | ✓ | ✓ | No |
| **`external`** | ✓ | ✗ | ✗ | Yes |
| **`private`** | ✗ | ✓ | ✗ | No |

**Outside** = external callers (off-chain, other contracts)  
**Within Contract** = same contract's methods  
**Subcontracts** = contracts inheriting from this one

```solidity
contract Base {
    function pub() public { }           // Anywhere
    function ext() external { }         // Outside only
    function int() internal { }         // Base + subcontracts only
    function priv() private { }         // Base only
}

contract Derived is Base {
    function test() public {
        // pub();      ✓ Can call public
        // int();      ✓ Can call internal (inherited)
        // priv();     ✗ Cannot call private (not inherited)
        // ext();      ✗ Cannot call external (contract boundary only)
    }
}

// External caller:
// base.pub()      ✓ OK
// base.ext()      ✓ OK
// base.int()      ✗ Error (private to contract)
```

**Gas tip:** Use `external` for functions never called internally—cheaper than `public`.

### State-Modifying Functions

A function that changes state:

```solidity
function increment() public {
    count += 1;  // Modifies state
}
```

No return type needed if it doesn't return anything.

### View and Pure Functions

**`view`** — reads state but doesn't modify it. Free to call (no gas cost for reading).

```solidity
function getCount() public view returns (uint256) {
    return count;  // Reads state, doesn't modify
}
```

**`pure`** — doesn't read or modify state. Computationally deterministic.

```solidity
function add(uint256 a, uint256 b) public pure returns (uint256) {
    return a + b;  // Pure function: just math
}
```

**Why distinguish?** The EVM treats `view`/`pure` specially: they execute locally (on your node) without needing consensus. State-modifying functions require a transaction and gas payment.

### Payable Functions

Functions that accept ether (ETH):

```solidity
function deposit() public payable {
    balances[msg.sender] += msg.value;  // msg.value = ether sent
}
```

Only `payable` functions can receive ether. If you send ether to a non-payable function, it reverts.

### Special Variables

Inside a function, you have access to:

| Variable | Type | Meaning |
|----------|------|---------|
| `msg.sender` | `address` | Who called this function |
| `msg.value` | `uint256` | How much ether (in wei) was sent |
| `msg.data` | `bytes` | The raw call data |
| `block.number` | `uint256` | Current block number |
| `block.timestamp` | `uint256` | Current block timestamp (Unix time) |
| `address(this)` | `address` | This contract's address |
| `this.balance` | `uint256` | This contract's ether balance |

Example:

```solidity
function withdraw() public {
    require(msg.sender == owner, "Only owner");
    payable(msg.sender).transfer(address(this).balance);
}
```

### Return Values

```solidity
// Single return
function getValue() public view returns (uint256) {
    return 42;
}

// Multiple returns (implicit naming)
function getMultiple() public view returns (uint256, bool) {
    return (42, true);
}

// Named returns (can omit explicit return)
function getNamed() public view returns (uint256 x, bool y) {
    x = 42;
    y = true;
    // implicit: return (x, y);
}
```

---

## Part 5: Control Flow

### If/Else

```solidity
if (x > 0) {
    // ...
} else if (x < 0) {
    // ...
} else {
    // ...
}
```

### Loops

```solidity
// For loop
for (uint256 i = 0; i < 10; i++) {
    // ...
}

// While loop
while (x > 0) {
    x--;
}

// Do-while
do {
    x--;
} while (x > 0);
```

**Warning:** Unbounded loops are dangerous—they can exceed gas limits and revert. Always bound loops.

### Exception Handling

**`require`** — check condition, revert if false (provides refund of unused gas):

```solidity
require(balance >= amount, "Insufficient balance");
balance -= amount;
```

**`assert`** — check condition, revert if false (consumes all gas, shouldn't be used for normal validation):

```solidity
assert(x > 0);  // Only for invariants; shouldn't fail in normal operation
```

**`revert`** — explicitly abort:

```solidity
if (x == 0) {
    revert("x cannot be zero");
}
```

**`try-catch`** — handle exceptions from external calls (rare):

```solidity
try externalContract.riskySomething() {
    // Success
} catch {
    // Handle error
}
```

---

## Part 6: Modifiers—Reusable Conditions

A **modifier** is a code snippet that guards a function (checks a condition before running).

```solidity
modifier onlyOwner() {
    require(msg.sender == owner, "Only owner");
    _;  // Underscore: "execute function body here"
}

function doAdminThing() public onlyOwner {
    // Function body only runs if modifier passes
    count += 1;
}
```

Formally, a modifier is a function transformer:

$$\text{modifier}(f) = \lambda m. \text{check}(m) \to \text{call}(f, m)$$

where `check(m)` is the guard condition, and `call(f, m)` executes the function.

You can stack modifiers:

```solidity
function complexFunction() public onlyOwner nonReentrant {
    // Runs checks from both onlyOwner and nonReentrant before executing
}
```

---

## Part 7: Events—Observable State Changes

Events are logs of what happened. They're stored in transaction receipts and indexed by clients, but *not* in contract state.

```solidity
event Transfer(address indexed from, address indexed to, uint256 amount);

function sendToken(address to, uint256 amount) public {
    balances[msg.sender] -= amount;
    balances[to] += amount;
    emit Transfer(msg.sender, to, amount);  // Log the event
}
```

**Why?** Events are cheap (logs are not state, so they don't cost storage), and they let external systems (like web frontends) listen to what happened.

**`indexed` keyword:** Makes the parameter searchable (filters can query by this value).

```solidity
event Transfer(address indexed from, address indexed to, uint256 amount);
// Clients can filter: "Give me all transfers from address X"
```

---

## Part 8: Constructor—Initialization

Runs once, when the contract is deployed:

```solidity
contract Counter {
    address public owner;
    
    constructor() {
        owner = msg.sender;  // Deployer is owner
    }
}
```

After the constructor finishes, the contract's initial state is set and the contract lives on the blockchain forever.

---

## Part 9: Type Conversions

### Implicit Conversions

Solidity allows safe implicit conversions:

```solidity
uint8 small = 5;
uint256 large = small;  // Implicit: uint8 → uint256 (safe)
```

Unsafe conversions must be explicit:

```solidity
uint256 x = 300;
uint8 y = uint8(x);  // Explicit: loses information (y = 44, wraps around)
```

### Conversions via Constructor

```solidity
address user = 0x742d35Cc6634C0532925a3b844Bc9e7595f42bE3;
uint256 encoded = uint256(uint160(user));  // Convert address to uint256

address recovered = address(uint160(encoded));  // Convert back
```

---

## Part 10: Operator Precedence & Semantics

### Arithmetic

```solidity
uint256 sum = a + b;
uint256 diff = a - b;      // Underflow reverts (0.8+)
uint256 product = a * b;
uint256 quotient = a / b;  // Integer division (truncates)
uint256 remainder = a % b;
uint256 power = a ** b;    // Exponentiation (expensive!)
```

**Division by zero reverts.**

### Comparison

```solidity
bool eq = a == b;
bool ne = a != b;
bool lt = a < b;
bool le = a <= b;
bool gt = a > b;
bool ge = a >= b;
```

### Logical

```solidity
bool and = (a && b);  // Short-circuit: if a is false, b not evaluated
bool or = (a || b);   // Short-circuit: if a is true, b not evaluated
bool not = !a;
```

### Bitwise

```solidity
uint256 and = a & b;
uint256 or = a | b;
uint256 xor = a ^ b;
uint256 not = ~a;
uint256 shl = a << n;  // Shift left
uint256 shr = a >> n;  // Shift right
```

---

## Part 11: Special Patterns

### Safe Math (0.8+)

Solidity 0.8+ has **checked arithmetic by default**:

```solidity
uint256 x = 2**256 - 1;
x += 1;  // Reverts (overflow protection)
```

To get unchecked arithmetic (for optimization), use:

```solidity
unchecked {
    uint256 x = 2**256 - 1;
    x += 1;  // Wraps around to 0 (no revert)
}
```

### Fallback Functions

Called when no matching function is found:

```solidity
fallback() external payable {
    // Handles unknown function calls
}

receive() external payable {
    // Handles plain ether transfers (no function call)
}
```

Only one `receive()` per contract; only one `fallback()` per contract.

### Errors (0.8.4+)

Modern way to express failures (cheaper than `require`):

```solidity
error InsufficientBalance(uint256 available, uint256 required);

function withdraw(uint256 amount) public {
    if (balance < amount) {
        revert InsufficientBalance(balance, amount);
    }
    balance -= amount;
}
```

---

## Part 12: Contract Inheritance

> **When is inheritance used?** Inheritance is **very common** in Solidity for code reuse:
> - **Token standards**: `ERC20`, `ERC721`, `ERC1155` (OpenZeppelin contracts). Your token inherits from `ERC20` to reuse transfer logic, balance tracking, approvals.
> - **Access control**: Base contracts like `Ownable` (one owner) or `AccessControl` (role-based) that derived contracts inherit from.
> - **Upgradeable proxies**: `UUPSUpgradeable`, `TransparentProxy` — derived contracts extend base proxy logic.
> - **Shared utilities**: Common functions (e.g., `safe math operations`, `event logging`, `validation`) live in base contracts; derived contracts focus on domain logic.
> 
> **Rule of thumb**: If you're writing a token, use inheritance. If you're writing access rules, use inheritance. Most real contracts are derived contracts, not base contracts.

Contracts can inherit from other contracts:

```solidity
contract Base {
    uint256 public x = 10;
    
    function getValue() public view returns (uint256) {
        return x;
    }
}

contract Derived is Base {
    function doubleValue() public view returns (uint256) {
        return x * 2;  // Can access base state
    }
}
```

**Multiple inheritance:** A contract can inherit from multiple contracts:

```solidity
contract C is A, B {
    // Inherits from both A and B
}
```

If A and B both define the same function, Solidity uses C3 linearization to determine which is used.

---

## Part 13: Interfaces and Abstract Contracts

### Interface

Specifies a contract's external API without implementation:

```solidity
interface IERC20 {
    function transfer(address to, uint256 amount) external returns (bool);
    function balanceOf(address owner) external view returns (uint256);
}

contract MyToken is IERC20 {
    // Must implement all functions from IERC20
    function transfer(address to, uint256 amount) external override returns (bool) {
        // Implementation
        return true;
    }
    
    function balanceOf(address owner) external view override returns (uint256) {
        // Implementation
        return 0;
    }
}
```

### Abstract Contracts

A contract with some unimplemented functions:

```solidity
abstract contract Base {
    function mustImplement() public virtual;  // No implementation
}

contract Derived is Base {
    function mustImplement() public override {
        // Implementation required
    }
}
```

---

## Part 14: Gas and Execution Model

**Gas** is the EVM's measure of computational cost. Every operation costs gas:

| Operation | Gas Cost |
|-----------|----------|
| Addition/Subtraction | 3 |
| Multiplication | 5 |
| Division | 5 |
| Storage read | 2,100 |
| Storage write | 20,000 |
| Memory read/write | 3 |
| Function call | 700+ |

**Why?** To prevent spam and ensure all nodes can keep up.

```solidity
function expensive() public {
    count += 1;  // Cheap: 3 gas for addition
    state = 100; // Expensive: 20,000 gas for storage write
}
```

**Memory vs. Storage:**
- **Storage** — persistent (costs thousands of gas to write)
- **Memory** — temporary, erased after function call (cheap)

```solidity
function process(uint256[] memory data) public {
    // 'memory' keyword: temporary array, cheap
    for (uint256 i = 0; i < data.length; i++) {
        // Process data
    }
}
```

---

## Summary: Syntax as State Machine Definition

Every piece of Solidity syntax maps to part of the state machine $(\Sigma, s_0, \mathcal{T})$:

- **State variables** → $\Sigma$ (the state space)
- **Constructor** → $s_0$ (initialization)
- **Functions** → parts of $\mathcal{T}$ (transition rules)
- **Modifiers** → guards on $\mathcal{T}$ (preconditions)
- **Events** → observable outputs of $\mathcal{T}$ (logs, not state)
- **Control flow** → internal structure of $\mathcal{T}$
- **Types** → domains and codomains of $\mathcal{T}$

Once you internalize this, every syntactic construct becomes obvious: it's machinery for defining deterministic, gas-aware state transitions on a decentralized computer.

---

## Quick Reference: Solidity Type Hierarchy

```
Value Types (passed by copy)
├── Primitives
│   ├── Integers: uint8, ..., uint256, int8, ..., int256
│   ├── Fixed-point: ufixed128x128 (rare)
│   ├── Booleans: bool
│   ├── Bytes: bytes1, ..., bytes32
│   └── Address: address, address payable
├── Structs (composite of value types)
└── Enums (encoded as uint8)

Reference Types (passed by reference)
├── Arrays
│   ├── Fixed: T[n]
│   └── Dynamic: T[]
├── Mappings: K ↦ V
├── Strings: string (UTF-8)
└── Bytes: bytes (dynamic)
```

---

## Quick Reference: Function Signatures

```solidity
// All forms
function f1() public { }                           // No params, no return
function f2(uint x) public returns (uint) { }     // One param, one return
function f3(uint x, address y) public returns (uint, bool) { }  // Multiple

// Modifiers
function f() public onlyOwner nonReentrant { }    // Stacked modifiers

// State access
function f() public view { }                       // Read-only
function f() public pure { }                       // Deterministic
function f() public payable { }                    // Accepts ether

// Special functions
constructor() { }                                  // Runs at deployment
fallback() external payable { }                   // Unknown function calls
receive() external payable { }                    // Plain ether transfers
```

That's it. You now know the complete syntax of Solidity. Everything else is library functions, design patterns, and best practices built on top of these primitives.
