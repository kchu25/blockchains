
@def title = "What Can Smart Contracts Actually Do? A Capabilities Inventory"
@def published = "1 March 2026"
@def tags = ["solidity", "smart-contracts", "general"]

# What Can Smart Contracts Actually Do?

> **The question:** Given the syntax and design constraints of Solidity (deterministic, gas-bounded, no I/O, no floating point, public state), what is the **full envelope** of things a smart contract can achieve? This post is an inventory of capabilities derived from first principles—not a list of existing projects, but a map of what's *possible*.

---

## Start From the Primitives

A Solidity contract is a state machine $(\Sigma, s_0, \mathcal{T})$ with these raw ingredients:

| Ingredient | What it gives you |
|------------|-------------------|
| **Persistent state** (mappings, arrays, structs) | Memory that survives across calls |
| **`msg.sender`** | Authenticated identity (cryptographically verified) |
| **`msg.value`** | Native value transfer (ETH) |
| **`block.timestamp`** | Time awareness |
| **`block.number`** | Sequence awareness |
| **Modifiers / `require`** | Enforceable rules |
| **Events** | Observable, indexable logs |
| **Inter-contract calls** | Composability (contracts calling contracts) |
| **`keccak256`** | Cryptographic commitments |
| **Immutability** | Code can't be changed after deployment |

Every application below is a **composition** of these primitives. Nothing more. The power comes from combining them in a trustless, permissionless environment where no single party can alter the rules.

---

## The Complete Capabilities Map

### 1. Tokenization — Representing Ownership

**What:** Turn any discrete or fungible thing into a transferable on-chain asset.

**How:** A mapping `address → uint256` (fungible) or `uint256 → address` (non-fungible) with transfer rules.

```solidity
// Fungible (ERC-20): "who owns how much"
mapping(address => uint256) public balanceOf;

// Non-fungible (ERC-721): "who owns which item"
mapping(uint256 => address) public ownerOf;
```

**What you can tokenize:**
- Currency, stablecoins
- Equity shares, bonds, real estate fractions
- Carbon credits, energy certificates
- Intellectual property rights
- Loyalty points, in-game items
- Academic credentials (Soulbound Tokens — non-transferable)

**Why finance came first:** Tokenization of money is the simplest case — a mapping of addresses to balances. It requires no oracle, no external data, no subjective judgment. Pure state accounting.

---

### 2. Automated Compliance — Rules That Can't Be Bypassed

**What:** Encode legal or business rules directly into the transfer logic.

**How:** Modifiers and `require` statements that gate every state transition.

```solidity
modifier kycApproved(address user) {
    require(kycRegistry[user], "KYC not approved");
    _;
}

modifier transferLimit(uint256 amount) {
    require(amount <= dailyLimit[msg.sender], "Exceeds daily limit");
    _;
}

function transfer(address to, uint256 amount) 
    public 
    kycApproved(msg.sender) 
    kycApproved(to)
    transferLimit(amount) 
{
    balanceOf[msg.sender] -= amount;
    balanceOf[to] += amount;
}
```

**What you can enforce:**
- KYC/AML checks (via oracle-linked registry)
- Transfer caps (daily, per-transaction, lifetime)
- Vesting schedules (tokens locked until timestamp)
- Geographic restrictions (address → jurisdiction mapping)
- Accredited investor gates
- Regulatory reporting (events emitted on every transfer)

**The insight:** In traditional finance, compliance is a *layer on top* — auditors check after the fact. In smart contracts, compliance is the *code itself*. Non-compliant transactions don't execute. They revert.

---

### 3. Escrow & Conditional Release — Trustless Intermediation

**What:** Hold funds and release them only when conditions are met. No third-party escrow agent needed.

**How:** Contract holds ETH/tokens; `require` checks conditions before releasing.

```solidity
contract Escrow {
    address public buyer;
    address public seller;
    uint256 public amount;
    bool public buyerApproved;
    bool public sellerDelivered;

    function confirmDelivery() external {
        require(msg.sender == buyer, "Not buyer");
        buyerApproved = true;
        if (sellerDelivered && buyerApproved) {
            payable(seller).transfer(amount);
        }
    }
}
```

**Use cases:**
- Real estate closings (funds released when title transfers)
- Freelance work (payment on milestone completion)
- Cross-border trade (release on shipment confirmation via oracle)
- Domain name sales
- Any two-party transaction with mutual distrust

---

### 4. Voting & Governance — Collective Decision-Making

**What:** Transparent, auditable, tamper-proof voting.

**How:** Mapping of `address → bool` (voted?), tallying, time-bounded proposals.

```solidity
struct Proposal {
    string description;
    uint256 forVotes;
    uint256 againstVotes;
    uint256 deadline;
    mapping(address => bool) hasVoted;
}

function vote(uint256 proposalId, bool support) external {
    Proposal storage p = proposals[proposalId];
    require(block.timestamp < p.deadline, "Voting ended");
    require(!p.hasVoted[msg.sender], "Already voted");
    
    p.hasVoted[msg.sender] = true;
    if (support) p.forVotes += votingPower[msg.sender];
    else p.againstVotes += votingPower[msg.sender];
}
```

**What you can govern:**
- Protocol parameter changes (interest rates, fees)
- Treasury spending (DAO allocates funds)
- Membership decisions (admit/remove members)
- Upgrade approvals (proxy contract points to new logic)
- Grant allocation (community votes on funding proposals)
- Any collective decision that currently requires a board, committee, or parliament

**Why this matters:** The vote tally is public. The counting is deterministic. No one can stuff ballots. No one can change the rules mid-vote. The code *is* the election commission.

---

### 5. Time-Locked Mechanisms — Scheduled Execution

**What:** Actions that become available or expire based on `block.timestamp`.

**How:** Simple comparisons against time.

```solidity
uint256 public unlockTime;

constructor(uint256 _lockDuration) payable {
    unlockTime = block.timestamp + _lockDuration;
}

function withdraw() external {
    require(block.timestamp >= unlockTime, "Still locked");
    payable(owner).transfer(address(this).balance);
}
```

**Use cases:**
- **Vesting**: Employee tokens unlock linearly over 4 years
- **Timelocks**: Governance decisions have 48-hour delay before execution (safety buffer)
- **Auctions**: Bidding period opens and closes at specific times
- **Insurance**: Claims must be filed within 30 days
- **Subscriptions**: Access expires after payment period
- **Dead man's switch**: If owner doesn't check in for 1 year, funds go to beneficiary

---

### 6. Automated Market Making — Algorithmic Exchange

**What:** Exchange assets without an order book or market maker. Price determined by math.

**How:** Two token reserves in a contract; price computed from the ratio.

```solidity
// Constant product: x * y = k
uint256 public reserveA;
uint256 public reserveB;

function getPrice(uint256 inputAmount) public view returns (uint256) {
    // Δy = (y * Δx) / (x + Δx)
    return (reserveB * inputAmount) / (reserveA + inputAmount);
}

function swap(uint256 inputAmount) external {
    uint256 outputAmount = getPrice(inputAmount);
    // Transfer tokens...
    reserveA += inputAmount;
    reserveB -= outputAmount;
}
```

**Why this works on-chain:** The pricing function is pure math (no oracle needed). The reserves are public state. Anyone can provide liquidity, anyone can trade. No exchange operator.

**Extends to:** Lending rates (supply/demand curves), insurance premiums (risk pools), prediction markets (outcome shares).

---

### 7. Commit-Reveal Schemes — Hiding Information Temporarily

**What:** Let users commit to a value without revealing it, then reveal later.

**How:** `keccak256` hashing for commitment; two-phase protocol.

```solidity
mapping(address => bytes32) public commitments;

// Phase 1: Commit (hash hides the value)
function commit(bytes32 hash) external {
    commitments[msg.sender] = hash;
}

// Phase 2: Reveal (prove your commitment)
function reveal(uint256 value, bytes32 salt) external {
    require(keccak256(abi.encodePacked(value, salt)) == commitments[msg.sender], 
            "Invalid reveal");
    // Process the revealed value
}
```

**Use cases:**
- **Sealed-bid auctions**: Bidders commit bids, reveal after deadline
- **Random number generation**: Multiple parties commit seeds, combine after reveal
- **Voting**: Hide votes until tally (prevents bandwagon effect)
- **Games**: Rock-paper-scissors, poker hands
- **DNS registration**: Prevent front-running of domain claims

**The constraint:** Blockchain state is public. You *cannot* hide data on-chain natively. Commit-reveal is the workaround — hide behind a hash, reveal when safe.

---

### 8. Access Control & Role Management — Permissioned Systems

**What:** Define who can do what, enforced cryptographically.

**How:** Mappings of `address → role` with modifier gates.

```solidity
mapping(bytes32 => mapping(address => bool)) public roles;

bytes32 public constant ADMIN = keccak256("ADMIN");
bytes32 public constant MINTER = keccak256("MINTER");
bytes32 public constant PAUSER = keccak256("PAUSER");

modifier onlyRole(bytes32 role) {
    require(roles[role][msg.sender], "Unauthorized");
    _;
}

function mint(address to, uint256 amount) external onlyRole(MINTER) { ... }
function pause() external onlyRole(PAUSER) { ... }
```

**What you can build:**
- Multi-tier organizational hierarchies
- Delegated authority (admin grants minter role)
- Temporary permissions (role + expiry timestamp)
- Separation of duties (minter ≠ pauser ≠ admin)

---

### 9. Reputation & Attestation — Trustless Track Records

**What:** Record verifiable actions and compute trust metrics on-chain.

**How:** Events + aggregated state per address.

```solidity
struct Reputation {
    uint256 tasksCompleted;
    uint256 totalStaked;
    uint8 avgRating;
}

mapping(address => Reputation) public reputation;
```

**Use cases:**
- Freelancer ratings (permanent, portable across platforms)
- Peer review scoring (DeSci)
- DAO contribution tracking (voting power from work, not wealth)
- Soulbound Tokens (non-transferable credentials: degrees, certifications, badges)
- Credit scoring without a central bureau

(See the [reputation systems post](/blog/solidity/reputation_systems/) for a deep dive.)

---

### 10. Composability — Contracts Calling Contracts

**What:** Contracts can call other contracts' public functions. This is the *killer feature*.

**How:** Interface declarations + external calls.

```solidity
interface IERC20 {
    function transferFrom(address from, address to, uint256 amount) external;
}

contract LendingPool {
    IERC20 public token;
    
    function deposit(uint256 amount) external {
        // Pull tokens from user into this contract
        token.transferFrom(msg.sender, address(this), amount);
        // Update internal accounting...
    }
}
```

**Why this is revolutionary:** In traditional software, APIs are behind authentication, rate limits, terms of service, and can be revoked. On-chain, any contract can call any public function of any other contract, *permissionlessly*. This enables:

- **Flash loans**: Borrow → use → repay in a single transaction (no collateral needed if atomicity is guaranteed)
- **Aggregators**: One contract queries 5 DEXes and routes to the best price
- **Yield strategies**: Contract A deposits into B, which stakes into C, which farms D
- **Composable identity**: Lending contract checks your reputation contract before approving a loan

**The "money lego" metaphor:** Each contract is a building block. Anyone can snap them together in new combinations without permission.

---

### 11. Insurance & Risk Pooling — Automated Claims

**What:** Pool funds from many participants; pay out when predefined conditions are met.

**How:** Deposits into a pool; oracle-triggered or vote-triggered payouts.

```solidity
contract CropInsurance {
    mapping(address => uint256) public premiums;
    uint256 public pool;
    
    function payPremium() external payable {
        premiums[msg.sender] += msg.value;
        pool += msg.value;
    }
    
    // Oracle reports drought
    function triggerPayout(address farmer) external onlyOracle {
        uint256 payout = premiums[farmer] * 3;  // 3x premium
        require(pool >= payout, "Pool depleted");
        pool -= payout;
        payable(farmer).transfer(payout);
    }
}
```

**Use cases:**
- Crop insurance (weather oracle triggers payout)
- Flight delay insurance (flight data oracle)
- Smart contract hack insurance (governance vote triggers payout)
- Health insurance pools (medical oracle + privacy layer)

**The constraint:** The hard part is the oracle — who decides if the drought happened? The contract can only enforce rules on data it receives. It can't observe the real world.

---

### 12. Supply Chain Tracking — Provenance Ledger

**What:** Record the journey of a physical item through a supply chain.

**How:** Events and state transitions representing checkpoints.

```solidity
enum Stage { Manufactured, Shipped, InTransit, Delivered, Verified }

struct Item {
    uint256 id;
    Stage stage;
    address currentHolder;
    uint256 lastUpdated;
}

mapping(uint256 => Item) public items;

event StageChanged(uint256 indexed itemId, Stage newStage, address indexed by);

function advanceStage(uint256 itemId) external {
    Item storage item = items[itemId];
    require(msg.sender == item.currentHolder, "Not holder");
    require(uint8(item.stage) < uint8(Stage.Verified), "Already final");
    
    item.stage = Stage(uint8(item.stage) + 1);
    item.lastUpdated = block.timestamp;
    
    emit StageChanged(itemId, item.stage, msg.sender);
}
```

**What you can track:**
- Food: farm → processor → distributor → retailer → consumer
- Pharmaceuticals: manufacturer → warehouse → pharmacy → patient
- Luxury goods: authenticity verification (is this bag really Hermès?)
- Diamonds: mine → cutter → dealer → jeweler (conflict-free certification)

**The constraint:** The contract trusts whoever calls `advanceStage`. If the warehouse lies about shipping, the blockchain faithfully records the lie. **Garbage in, garbage out.** Physical-world verification still requires trusted participants or IoT devices.

---

### 13. Identity & Credentials — Self-Sovereign Identity

**What:** Users own their identity data; contracts verify claims without a central authority.

**How:** Attestation contracts + Soulbound Tokens (non-transferable NFTs).

```solidity
contract CredentialRegistry {
    // Issuer attests: "this address completed course X"
    mapping(address => mapping(bytes32 => bool)) public credentials;
    
    function issue(address recipient, bytes32 credentialHash) 
        external 
        onlyIssuer 
    {
        credentials[recipient][credentialHash] = true;
        emit CredentialIssued(recipient, credentialHash, msg.sender);
    }
    
    // Anyone can verify
    function verify(address user, bytes32 credentialHash) 
        external 
        view 
        returns (bool) 
    {
        return credentials[user][credentialHash];
    }
}
```

**Use cases:**
- University degree verification (no calling the registrar)
- Professional certifications (on-chain, permanent)
- KYC attestation (verified once, reusable across platforms)
- Age verification (zero-knowledge proof: "I'm over 18" without revealing birthdate)

---

### 14. Subscription & Recurring Payments

**What:** Authorize a contract to pull payments on a schedule.

**How:** Allowance pattern + timestamp checks.

```solidity
struct Subscription {
    uint256 amount;
    uint256 interval;    // e.g., 30 days
    uint256 lastCharged;
}

mapping(address => Subscription) public subscriptions;

function charge(address subscriber) external {
    Subscription storage sub = subscriptions[subscriber];
    require(block.timestamp >= sub.lastCharged + sub.interval, "Too early");
    
    sub.lastCharged = block.timestamp;
    token.transferFrom(subscriber, address(this), sub.amount);
}
```

**The constraint:** Solidity can't auto-execute. Someone (a bot, a keeper network like Chainlink Keepers) must call `charge()` at the right time. The contract only enforces that charging *can't happen too early*.

---

### 15. Prediction Markets — Betting on Outcomes

**What:** Users bet on future events; payouts determined by outcomes.

**How:** Users buy shares in outcomes; oracle resolves; winners claim proportional payout.

```solidity
// Simplified: binary prediction market
mapping(address => uint256) public yesShares;
mapping(address => uint256) public noShares;
uint256 public totalYes;
uint256 public totalNo;
bool public resolved;
bool public outcome;

function buyYes() external payable {
    yesShares[msg.sender] += msg.value;
    totalYes += msg.value;
}

function resolve(bool _outcome) external onlyOracle {
    resolved = true;
    outcome = _outcome;
}

function claim() external {
    require(resolved, "Not resolved");
    uint256 payout;
    if (outcome) {
        payout = (yesShares[msg.sender] * (totalYes + totalNo)) / totalYes;
    }
    // Transfer payout...
}
```

**Use cases:**
- Election outcomes
- Sports results
- Scientific reproducibility ("Will this paper replicate?")
- Weather events
- Any binary or categorical future event

---

## What Smart Contracts **Cannot** Do

Understanding the constraints is just as important:

| Constraint | Why | Workaround |
|-----------|-----|------------|
| **No external I/O** | EVM is sandboxed; can't fetch URLs, read files, call APIs | Oracles (Chainlink, UMA) feed external data in |
| **No floating point** | Determinism requires exact math across all nodes | Fixed-point arithmetic (multiply by $10^{18}$, divide at the end) |
| **No randomness** | `block.timestamp` and `blockhash` are manipulable by miners | Chainlink VRF, commit-reveal schemes |
| **No auto-execution** | Contract code only runs when someone calls it | Keeper networks (Chainlink Keepers, Gelato) |
| **Public state** | All storage is readable by anyone | Commit-reveal, zero-knowledge proofs, off-chain computation |
| **Gas limits** | Loops can't be unbounded; computation is expensive | Off-chain computation with on-chain verification |
| **Immutable code** | Deployed contracts can't be modified | Proxy pattern (delegatecall to upgradeable logic) |
| **No string processing** | String ops are gas-expensive and limited | Hash strings instead; process off-chain |
| **~12s latency** | Transactions aren't instant | Layer 2 solutions (Optimism, Arbitrum) for faster finality |

---

## Why Finance Came First

It's not an accident that DeFi dominates. Financial applications are the **path of least resistance** given the constraints:

1. **No oracle needed**: Token balances are native on-chain data. Swapping token A for token B requires no external information.
2. **Pure math**: Interest rates, exchange rates, collateral ratios — all computable with integer arithmetic.
3. **High value density**: Moving \$1M costs the same gas as moving \$1. The value proposition scales trivially.
4. **Immediate composability**: One financial primitive (swap) feeds into another (lend) which feeds into another (yield farm).
5. **Clear incentive alignment**: Users are directly financially motivated to use the system correctly.

**Non-financial applications** face harder problems: they need oracles (supply chain), subjective judgment (reputation), legal recognition (identity), or privacy (health data). These are solvable — but they add layers of complexity.

---

## The Essence, Distilled

A smart contract gives you exactly one thing: **a set of rules that no single party can change or violate, enforced by cryptography and consensus.**

From this, you can build:

$$\underbrace{\text{Tokenization}}_{\text{ownership}} + \underbrace{\text{Conditions}}_{\texttt{require}} + \underbrace{\text{Time}}_{\texttt{block.timestamp}} + \underbrace{\text{Identity}}_{\texttt{msg.sender}} + \underbrace{\text{Composability}}_{\text{contract calls}} = \text{Trustless Coordination}$$

Every application in this post is a combination of these five primitives. The constraint is always the same: the contract can only reason about data that exists on-chain. For everything else, you need an oracle — and the oracle is where trust re-enters the system.

> **The mental model:** A smart contract is a **vending machine** — not a human clerk. It has fixed rules, accepts specific inputs, and produces deterministic outputs. It can't go check the weather or call your bank. But within its rules, it is incorruptible, tireless, and available 24/7 to anyone with an Ethereum address.
