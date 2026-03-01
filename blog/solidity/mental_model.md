
@def title = "Thinking About Smart Contracts: A Discrete Math Mental Model"
@def published = "1 March 2026"
@def tags = ["solidity", "general"]

# Thinking About Smart Contracts: A Discrete Math Mental Model

> **The problem:** Thinking in Solidity is slow. You get bogged down in `mapping`, `require`, `msg.sender`, gas, and lose sight of the *problem* you're solving. This post gives you a formal framework — rooted in discrete math — to reason about smart contracts *before* writing a single line of code. Design the system on paper, then translate to Solidity mechanically.

---

## The Core Object: A Guarded Transition System

Every smart contract is a **guarded transition system**. Forget Solidity for a moment. Here's the mathematical object:

$$\mathcal{C} = (S, s_0, A, G, \delta, O)$$

where:

| Symbol | Name | What it is |
|--------|------|------------|
| $S$ | State space | All possible configurations of the contract |
| $s_0 \in S$ | Initial state | Configuration at deployment |
| $A$ | Action set | All possible calls (function name + arguments + caller) |
| $G: S \times A \to \{\text{true}, \text{false}\}$ | Guard predicates | Conditions that must hold for an action to be allowed |
| $\delta: S \times A \rightharpoonup S$ | Transition function | How the state changes (partial: undefined when guard fails) |
| $O: S \times A \to \mathcal{E}^*$ | Output function | Events emitted (observable side effects) |

**That's it.** Every smart contract ever written is an instantiation of this tuple. The Solidity syntax is just notation for specifying each component.

---

## Component 1: State Space $S$

The state space is the **Cartesian product** of all your state variables:

$$S = D_1 \times D_2 \times \cdots \times D_n$$

where each $D_i$ is the domain of a state variable.

### Example: Token Contract

A token has:
- Balances: who owns how much
- Total supply: how many tokens exist
- Allowances: who is authorized to spend on whose behalf

$$S = \underbrace{(\text{Addr} \to \mathbb{N})}_{\text{balances}} \times \underbrace{\mathbb{N}}_{\text{totalSupply}} \times \underbrace{(\text{Addr} \times \text{Addr} \to \mathbb{N})}_{\text{allowances}}$$

A specific state is a point in this space:

$$s = (\text{bal}, \text{supply}, \text{allow})$$

where $\text{bal}: \text{Addr} \to \mathbb{N}$ is the balance function, etc.

> **Key insight:** Before writing any code, enumerate your state variables and their domains. The state space is the product. This is your "database schema."

### Example: Auction Contract

$$S = \underbrace{\text{Addr}}_{\text{highestBidder}} \times \underbrace{\mathbb{N}}_{\text{highestBid}} \times \underbrace{(\text{Addr} \to \mathbb{N})}_{\text{pendingReturns}} \times \underbrace{\{\text{Open}, \text{Closed}\}}_{\text{phase}} \times \underbrace{\mathbb{N}}_{\text{endTime}}$$

Note: the phase variable is a **finite set** (an enum). This makes the contract a *finite state machine* in that dimension, while infinite in the others (balances can be any natural number).

---

## Component 2: Action Set $A$

An action is a **function call with context**. Formally:

$$a = (f, \vec{x}, c, v, t) \in A$$

where:
- $f$ = function name
- $\vec{x}$ = arguments
- $c \in \text{Addr}$ = caller (`msg.sender`)
- $v \in \mathbb{N}$ = value sent (`msg.value`)
- $t \in \mathbb{N}$ = timestamp (`block.timestamp`)

In Solidity, you write `function f(uint x, address y) public payable`. In math, you define the action:

$$a_f = (f, (x, y), c, v, t)$$

> **Why model the caller as part of the action?** Because in smart contracts, *who* calls a function is as important as *what* they call it with. The same function called by the owner vs. a random user may have completely different behavior.

### Enumerate Your Actions

For a token contract:

$$A = \{ \text{transfer}(to, amt, c), \text{approve}(spender, amt, c), \text{transferFrom}(from, to, amt, c), \text{mint}(to, amt, c) \}$$

Each action is a distinct input to the system. List them all before writing code.

---

## Component 3: Guard Predicates $G$

This is where smart contracts get interesting. A guard is a **precondition** that must be satisfied for an action to execute:

$$G(s, a) = \begin{cases} \text{true} & \text{action is allowed in state } s \\ \text{false} & \text{action reverts} \end{cases}$$

Guards encode your **business rules**, **access control**, and **safety constraints**.

### Example: Token Transfer Guards

For $a = \text{transfer}(to, amt, c)$ in state $s = (\text{bal}, \text{supply}, \text{allow})$:

$$G(s, a) = \underbrace{(\text{bal}(c) \geq amt)}_{\text{sufficient balance}} \wedge \underbrace{(to \neq \mathbf{0})}_{\text{valid recipient}} \wedge \underbrace{(amt > 0)}_{\text{positive amount}}$$

Each conjunct is a `require` statement in Solidity:

```solidity
require(balances[msg.sender] >= amount, "Insufficient balance");
require(to != address(0), "Invalid recipient");
require(amount > 0, "Amount must be positive");
```

### Example: Auction Bid Guards

For $a = \text{bid}(c, v, t)$:

$$G(s, a) = \underbrace{(s.\text{phase} = \text{Open})}_{\text{auction is active}} \wedge \underbrace{(t < s.\text{endTime})}_{\text{not expired}} \wedge \underbrace{(v > s.\text{highestBid})}_{\text{outbids current}}$$

### Guard Composition

Guards compose naturally with logical connectives:

$$G_{\text{combined}} = G_1 \wedge G_2 \wedge G_3$$

This is why Solidity modifiers stack:

```solidity
function bid() public payable onlyDuring(Phase.Open) beforeDeadline {
    require(msg.value > highestBid, "Bid too low");
}
```

Each modifier is one conjunct. The `require` in the body is another. The full guard is their conjunction.

> **Design principle:** Write out your guard predicates as logical formulas first. Each conjunct becomes a `require` or modifier. If you can't express a guard formally, you don't understand your business rule well enough.

---

## Component 4: Transition Function $\delta$

The transition function defines how state changes:

$$\delta(s, a) = s' \quad \text{(new state)}$$

It's a **partial function** — defined only when $G(s, a) = \text{true}$.

### Example: Token Transfer

For $a = \text{transfer}(to, amt, c)$:

$$\delta(s, a) = s' \text{ where } \begin{cases} s'.\text{bal}(c) = s.\text{bal}(c) - amt \\ s'.\text{bal}(to) = s.\text{bal}(to) + amt \\ s'.\text{supply} = s.\text{supply} & \text{(unchanged)} \\ s'.\text{allow} = s.\text{allow} & \text{(unchanged)} \end{cases}$$

In Solidity:

```solidity
balances[msg.sender] -= amount;
balances[to] += amount;
```

### Key Property: Conservation Laws

Notice that in the transfer above:

$$s'.\text{bal}(c) + s'.\text{bal}(to) = s.\text{bal}(c) + s.\text{bal}(to)$$

Total tokens are **conserved**. This is an *invariant* — a property that holds across all transitions. You should identify invariants before writing code:

$$\forall s, a: G(s,a) \implies I(\delta(s,a))$$

"For every state and action, if the guard passes, the invariant still holds after the transition."

### Example Invariants

| Contract | Invariant |
|----------|-----------|
| Token | $\sum_a \text{bal}(a) = \text{totalSupply}$ |
| Auction | $\text{highestBid} \leq \text{address(this).balance}$ |
| Escrow | $\text{balance} = \text{buyerDeposit} + \text{sellerDeposit}$ |
| Voting | $\sum_p \text{votes}(p) \leq \text{totalVoters}$ |

**If you can't state your invariants, you have a bug waiting to happen.**

---

## Component 5: State Diagram

Now comes the visual tool. For contracts with a **finite phase variable** (enum), draw the state diagram:

```
                bid()
    ┌──────────────────────┐
    │                      │
    ▼                      │
┌─────────┐  endAuction() ┌─────────┐  withdraw()  ┌───────────┐
│  Open   │──────────────→│ Closed  │─────────────→│ Finalized │
└─────────┘               └─────────┘              └───────────┘
     │                         │
     │ cancel()                │ (no more transitions)
     ▼                         │
┌──────────┐                   │
│ Canceled │←──────────────────┘
└──────────┘    (if no bids)
```

Each node is a **phase** (value of the enum). Each edge is an **action** that transitions between phases. The guard predicates determine which edges are enabled.

### Drawing the Diagram: A Procedure

1. **List your phases** (the finite part of $S$). These become nodes.
2. **List your actions** ($A$). These become candidate edges.
3. **For each action, ask:** "Which phase must I be in?" and "Which phase do I end up in?" Draw the edge.
4. **Label each edge** with its guard predicates.
5. **Identify absorbing states** — states with no outgoing edges (e.g., "Finalized").

This is a **labeled transition system** (LTS) in formal methods terminology.

### When There's No Enum

Some contracts (e.g., a simple token) don't have an explicit phase variable. The "state diagram" collapses to a single node with self-loops:

```
        transfer()
    ┌──────────────────┐
    │                  │
    ▼                  │
┌────────────┐  approve()  
│  Active    │──────────┘
│ (only      │
│  phase)    │──── mint()
└────────────┘
```

Here the interesting structure is in the **guard predicates** and **invariants**, not the phase transitions.

> **Rule of thumb:** If your contract has 2+ phases, draw the state diagram. If it has 1 phase, focus on invariants.

---

## Component 6: Roles and Permissions — A Relation

Access control is a **relation** between addresses and actions:

$$R \subseteq \text{Addr} \times A$$

$(c, a) \in R$ means "caller $c$ is authorized to perform action $a$."

In practice, you define **roles** as equivalence classes:

$$\text{Roles} = \{\text{Owner}, \text{Admin}, \text{Minter}, \text{User}, \ldots\}$$

And a role assignment function:

$$\rho: \text{Addr} \to \mathcal{P}(\text{Roles})$$

(Each address maps to a *set* of roles it holds.)

Then guards include role checks:

$$G(s, a) = \cdots \wedge (\text{role} \in \rho(c))$$

### Permission Matrix

Draw this before writing modifiers:

| Action | Owner | Admin | Minter | Anyone |
|--------|-------|-------|--------|--------|
| `mint` | ✓ | ✓ | ✓ | ✗ |
| `burn` | ✓ | ✓ | ✗ | ✗ |
| `pause` | ✓ | ✗ | ✗ | ✗ |
| `transfer` | ✓ | ✓ | ✓ | ✓ |
| `grantRole` | ✓ | ✗ | ✗ | ✗ |

Each row becomes a function. Each "✓" column becomes a guard conjunct. This table *is* your access control design.

---

## Putting It All Together: The Design Procedure

Before writing any Solidity:

### Step 1: Define $S$ (State Space)

List every piece of data your contract needs to remember.

$$S = D_1 \times D_2 \times \cdots \times D_n$$

Ask: *What does the contract need to know between calls?*

### Step 2: Define $s_0$ (Initial State)

What are the initial values? Who is the deployer? What's the starting phase?

### Step 3: Define $A$ (Actions)

List every operation a user can perform. Include the caller as part of the action.

### Step 4: Define $G$ (Guards)

For each action, write the preconditions as a logical formula.

$$G(s, a) = \phi_1 \wedge \phi_2 \wedge \cdots \wedge \phi_k$$

### Step 5: Define $\delta$ (Transitions)

For each action, specify how each state variable changes (or stays the same).

### Step 6: State Invariants

What properties must *always* hold?

$$\forall s \in \text{Reachable}(s_0): I(s) = \text{true}$$

### Step 7: Draw the State Diagram

If you have phases, draw the LTS. Label edges with actions and guards.

### Step 8: Permission Matrix

Who can do what? Draw the role × action table.

### Step 9: Translate to Solidity

Now — and only now — open your editor.

| Math | Solidity |
|------|----------|
| $S = D_1 \times \cdots \times D_n$ | State variables (`uint256`, `mapping`, `enum`, ...) |
| $s_0$ | Constructor |
| $A$ | Functions |
| $G(s, a) = \phi_1 \wedge \cdots$ | `require(...)` + modifiers |
| $\delta(s, a) = s'$ | Function body (state mutations) |
| $I(s)$ | `assert(...)` (invariant checks) |
| $O(s, a)$ | `emit Event(...)` |
| $\rho(c)$ | `mapping(address => Role)` + `onlyRole` modifiers |
| Phase enum | `enum Phase { ... }` |
| State diagram edges | Functions with `require(phase == ...)` guards |

---

## Worked Example: Crowdfunding Contract

Let's design a crowdfunding contract — purely in math, then translate.

### Step 1: State Space

$$S = \underbrace{(\text{Addr} \to \mathbb{N})}_{\text{pledges}} \times \underbrace{\mathbb{N}}_{\text{goal}} \times \underbrace{\mathbb{N}}_{\text{deadline}} \times \underbrace{\text{Addr}}_{\text{creator}} \times \underbrace{\{\text{Funding}, \text{Succeeded}, \text{Failed}\}}_{\text{phase}} \times \underbrace{\mathbb{N}}_{\text{totalPledged}}$$

### Step 2: Initial State

$$s_0 = (\lambda a.\, 0,\; \text{goal},\; \text{now} + \text{duration},\; \text{deployer},\; \text{Funding},\; 0)$$

### Step 3: Actions

$$A = \{\text{pledge}(c, v, t), \;\text{claim}(c, t), \;\text{refund}(c, t), \;\text{cancel}(c)\}$$

### Step 4: Guards

**pledge:**
$$G_{\text{pledge}} = (s.\text{phase} = \text{Funding}) \wedge (t < s.\text{deadline}) \wedge (v > 0)$$

**claim** (creator withdraws if goal met):
$$G_{\text{claim}} = (c = s.\text{creator}) \wedge (s.\text{phase} = \text{Succeeded})$$

**refund** (pledger gets money back if failed):
$$G_{\text{refund}} = (s.\text{phase} = \text{Failed}) \wedge (s.\text{pledges}(c) > 0)$$

**finalize** (anyone can trigger phase change after deadline):
$$G_{\text{finalize}} = (s.\text{phase} = \text{Funding}) \wedge (t \geq s.\text{deadline})$$

$$\delta_{\text{finalize}}.\text{phase} = \begin{cases} \text{Succeeded} & \text{if } s.\text{totalPledged} \geq s.\text{goal} \\ \text{Failed} & \text{otherwise} \end{cases}$$

### Step 5: Transitions

**pledge:**
$$\delta: \quad s'.\text{pledges}(c) = s.\text{pledges}(c) + v, \quad s'.\text{totalPledged} = s.\text{totalPledged} + v$$

**claim:**
$$\delta: \quad \text{transfer } s.\text{totalPledged} \text{ to } s.\text{creator}$$

**refund:**
$$\delta: \quad \text{transfer } s.\text{pledges}(c) \text{ to } c, \quad s'.\text{pledges}(c) = 0$$

### Step 6: Invariants

$$I_1: \quad s.\text{totalPledged} = \sum_a s.\text{pledges}(a)$$
$$I_2: \quad s.\text{phase} = \text{Succeeded} \implies s.\text{totalPledged} \geq s.\text{goal}$$
$$I_3: \quad s.\text{phase} = \text{Funding} \implies \text{no funds have been withdrawn}$$

### Step 7: State Diagram

```
              pledge()                 finalize()
          ┌───────────────┐      [total ≥ goal]
          │               │     ┌──────────────┐
          ▼               │     ▼              │
     ┌──────────┐    finalize()    ┌────────────┐  claim()  ┌─────────┐
     │ Funding  │─────────────────→│ Succeeded  │─────────→│ Claimed │
     └──────────┘                  └────────────┘          └─────────┘
          │
          │ finalize()
          │ [total < goal]
          ▼
     ┌──────────┐  refund()  ┌──────────┐
     │  Failed  │───────────→│  Failed  │ (self-loop: each pledger refunds)
     └──────────┘            └──────────┘
```

### Step 8: Permission Matrix

| Action | Creator | Pledger | Anyone |
|--------|---------|---------|--------|
| `pledge` | ✓ | ✓ | ✓ |
| `finalize` | ✓ | ✓ | ✓ |
| `claim` | ✓ | ✗ | ✗ |
| `refund` | ✗ | ✓ (self only) | ✗ |

### Step 9: Translate

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

contract Crowdfund {
    // S = pledges × goal × deadline × creator × phase × totalPledged
    enum Phase { Funding, Succeeded, Failed }

    mapping(address => uint256) public pledges;
    uint256 public goal;
    uint256 public deadline;
    address public creator;
    Phase public phase;
    uint256 public totalPledged;

    // s₀
    constructor(uint256 _goal, uint256 _duration) {
        goal = _goal;
        deadline = block.timestamp + _duration;
        creator = msg.sender;
        phase = Phase.Funding;
    }

    // G_pledge ∧ δ_pledge
    function pledge() external payable {
        require(phase == Phase.Funding, "Not funding");
        require(block.timestamp < deadline, "Deadline passed");
        require(msg.value > 0, "Must send ETH");

        pledges[msg.sender] += msg.value;
        totalPledged += msg.value;
    }

    // G_finalize ∧ δ_finalize
    function finalize() external {
        require(phase == Phase.Funding, "Already finalized");
        require(block.timestamp >= deadline, "Too early");

        if (totalPledged >= goal) {
            phase = Phase.Succeeded;
        } else {
            phase = Phase.Failed;
        }
    }

    // G_claim ∧ δ_claim
    function claim() external {
        require(msg.sender == creator, "Not creator");
        require(phase == Phase.Succeeded, "Not succeeded");

        phase = Phase.Succeeded; // stays (or add Claimed phase)
        payable(creator).transfer(totalPledged);
    }

    // G_refund ∧ δ_refund
    function refund() external {
        require(phase == Phase.Failed, "Not failed");
        uint256 amount = pledges[msg.sender];
        require(amount > 0, "Nothing to refund");

        pledges[msg.sender] = 0;  // CEI: update state before transfer
        payable(msg.sender).transfer(amount);
    }
}
```

Notice: every `require` is a conjunct from $G$. Every state mutation is from $\delta$. The enum is the phase variable from the state diagram. **The math designed the contract. Solidity just transcribed it.**

---

## Why This Works Better Than "Thinking in Solidity"

| Thinking in Solidity | Thinking in Math |
|---------------------|------------------|
| "I need a mapping and a require" | "What's the state space? What are the guards?" |
| Bottom-up: syntax → structure | Top-down: structure → syntax |
| Easy to forget edge cases | Guards are explicit logical formulas — enumerate conjuncts |
| Hard to spot missing invariants | Invariants are stated upfront, verified against all transitions |
| Access control is ad hoc | Permission matrix is a table you fill in |
| Phase transitions are implicit | State diagram makes them visual |

The math gives you a **specification** that's independent of any programming language. You could implement the same spec in Solidity, Vyper, Cairo, or even a traditional database. The smart contract is just one realization of the formal model.

---

## Cheat Sheet: Math → Solidity Translation

| You're thinking about... | Math notation | Solidity |
|--------------------------|--------------|----------|
| What data do I store? | $S = D_1 \times \cdots \times D_n$ | State variables |
| What can users do? | $A = \{a_1, a_2, \ldots\}$ | Functions |
| When is an action allowed? | $G(s, a) = \phi_1 \wedge \cdots$ | `require` + modifiers |
| What changes? | $\delta(s, a) = s'$ | Assignment statements |
| What must always be true? | $\forall s: I(s)$ | `assert` (or off-chain tests) |
| What happened? | $O(s, a) = e_1, e_2, \ldots$ | `emit Event(...)` |
| Who can do what? | $R \subseteq \text{Addr} \times A$ | `onlyOwner`, `onlyRole(...)` |
| What phases exist? | Finite set $\{p_1, \ldots, p_k\}$ | `enum` |
| How do phases change? | LTS edges | Functions with `require(phase == ...)` |

---

## The Takeaway

**Before you write Solidity, answer these questions on paper:**

1. What does the contract need to remember? → **State space** $S$
2. What can users do? → **Action set** $A$
3. Under what conditions? → **Guard predicates** $G$
4. What changes when they do it? → **Transition function** $\delta$
5. What must always be true? → **Invariants** $I$
6. Who can do what? → **Permission matrix** $R$
7. What are the phases? → **State diagram**

If you can answer all seven, the Solidity writes itself. If you can't, you don't understand the problem yet — and no amount of coding will fix that.
