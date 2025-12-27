@def title = "Solidity: A Mathematical Framework"
@def published = "27 December 2025"
@def tags = ["solidity"]

# Solidity: A Mathematical Framework

## The Core Insight

> **Why is a contract a "state space"?**
> 
> Think of your contract like a video game save file. At any moment, the game has a specific configuration: your health is 80, you have 3 potions, you're at level 5, you're in the forest area. That's the **current state**.
> 
> The **state space** is *all possible configurations* the game could be in: health from 0-100, potions from 0-99, any level, any location. It's the entire universe of "ways the game could be."
> 
> A Solidity contract works the same way:
> - **Current state** = the actual values stored right now (Alice has 100 tokens, Bob has 50 tokens)
> - **State space** = all possible values those variables could hold (anyone could have 0 to 2²⁵⁶-1 tokens)
> 
> When you call a function, you're not adding new data—you're **moving from one point in the state space to another point**. Like pressing a button in the game: your health goes from 80→100 (you moved in "health space").
> 
> **Why this matters:** The state space is *fixed* when you deploy the contract (you define what variables exist). Functions just navigate within that space. You can't suddenly add a new variable mid-game—that would be changing the rules of what states are possible.
> 
> So a contract is a state space because it defines "here are all the configurations I can be in" and functions are just "here's how to jump from one configuration to another."

> **So interactions are just navigation?**
>
> Exactly! Every function call is like telling the contract: "Move from here to there in the state space."
>
> But here's the catch—you can't just teleport anywhere. You have **constraints**:
>
> 1. **Access constraints** (the bouncer): "Are you allowed to press this button?"
>    - `onlyOwner` → only one person can navigate this way
>    - `payable` → you need to bring ETH to use this route
>
> 2. **Logic constraints** (the rules): "Is this move even legal?"
>    - Can't transfer more tokens than you have
>    - Can't withdraw if balance is zero
>    - Can't bid after auction ended
>
> 3. **Cost constraints** (the toll): "Do you have enough gas to make this move?"
>    - Each step costs computational resources
>    - Complex moves cost more than simple ones
>
> Think of it like a board game: the state is where all the pieces are, functions are the allowed moves, and constraints are the rules preventing illegal moves. You're not creating new pieces—you're just moving existing pieces within the rules.
>
> **The blockchain just enforces that everyone follows the same navigation rules.** No cheating, no backsies.

> **Does a contract have a termination state?**
>
> Here's the twist: **most contracts never actually "end"**—they just sit there forever, frozen in whatever state they're in.
>
> But yes, you can design contracts with terminal states:
>
> **Option 1: Logical termination** (the contract is "done" but still exists)
> - An auction reaches `State.Ended` → no more bids allowed, winner claimed their item
> - A vesting contract releases all tokens → nothing left to vest
> - Think: the game is over, but the save file still exists. You just can't press any meaningful buttons anymore.
>
> Mathematically: you reach a state $s_{\text{terminal}}$ where all remaining functions become no-ops or revert:
> $\forall f \in F: f(s_{\text{terminal}}, \cdot) = s_{\text{terminal}} \text{ or } \text{revert}$
>
> **Option 2: Self-destruct** (true annihilation)
> - Use `selfdestruct(recipient)` → contract code is erased, remaining ETH sent to recipient
> - The contract address becomes a tombstone—you can still send it money, but there's no code to execute
> - Think: deleting the game entirely, not just finishing it
>
> **Why most contracts don't terminate:**
> - Tokens need to exist forever (imagine if dollar bills expired!)
> - DAOs persist indefinitely
> - Many protocols are designed to run perpetually
>
> So yes, termination states exist, but they're optional design choices, not requirements. A contract can be "functionally complete" (auction ended) while still being technically alive (code exists, state persists).

Solidity isn't really about syntax—it's about **state transitions under constraints**. Think of it as defining a state machine where every function is a transition function that costs money to execute.

## The Big Picture: A Formal System

A smart contract is essentially:

$$\mathcal{C} = (S, F, \tau, \mathcal{G})$$

where:
- $S$ = state space (all possible storage configurations)
- $F$ = set of functions (state transition operators)
- $\tau$ = access control predicates (who can call what)
- $\mathcal{G}$ = gas cost function (computational price)

---

## 1. State Space $S$

Your contract's state is just a tuple of typed values:

$$S = V_1 \times V_2 \times \cdots \times V_n$$

**Example:** A simple token contract
```
uint256 totalSupply;           // V₁ ∈ [0, 2²⁵⁶-1]
mapping(address => uint256);   // V₂: Address → [0, 2²⁵⁶-1]
```

becomes:

$$S = \mathbb{Z}_{2^{256}} \times (A \to \mathbb{Z}_{2^{256}})$$

where $A$ is the set of all Ethereum addresses (20-byte values).

**Intuition:** State is just "all the variables that persist between function calls." Mathematically, it's a product space of typed domains.

---

## 2. Functions as State Transitions

Every function is a partial mapping:

$$f: S \times I \times C \rightharpoonup S$$

where:
- $I$ = function inputs (parameters)
- $C$ = context (msg.sender, msg.value, block.timestamp, etc.)
- $\rightharpoonup$ means "partial function" (can fail/revert)

**Example:** Transfer function
```solidity
function transfer(address to, uint256 amount) public {
    require(balances[msg.sender] >= amount);
    balances[msg.sender] -= amount;
    balances[to] += amount;
}
```

becomes:

$$\text{transfer}(s, \text{to}, \text{amount}, c) = s'$$

where:
- Precondition: $s.\text{balances}[c.\text{sender}] \geq \text{amount}$
- Effect: 
  - $s'.\text{balances}[c.\text{sender}] = s.\text{balances}[c.\text{sender}] - \text{amount}$
  - $s'.\text{balances}[\text{to}] = s.\text{balances}[\text{to}] + \text{amount}$

**Intuition:** Functions are just "before → after" transformations with conditions. If the condition fails, the transformation doesn't happen (revert).

---

## 3. Access Control as Predicates

> **Access control is just a yes/no gate: "Can you come in?"**
> 
> Think of it like a bouncer at a club. You walk up with your ID (your address) and some info (how much money you have, what time it is). The bouncer checks a simple rule and decides: let you in (true) or kick you out (false).
> 
> Mathematically, it's a function:
> $\tau: \text{Context} \to \{\text{true}, \text{false}\}$
> 
> **Context** = everything about this moment:
> - Who's calling? (`msg.sender`)
> - How much ETH did they send? (`msg.value`)
> - What time is it? (`block.timestamp`)
> 
> **Examples:**
> 
> `onlyOwner` modifier → $\tau(c) = (c.\text{sender} = \text{owner})$
> - "Are you the owner? Yes → come in. No → get lost."
> 
> `onlyAdmin` modifier → $\tau(c) = \text{admins}[c.\text{sender}]$
> - "Are you on the admin list? Check the mapping."
> 
> `payable` function → $\tau(c) = (c.\text{value} > 0)$
> - "Did you bring money? No money, no entry."
> 
> **The key insight:** Access control isn't some special mechanism—it's just a boolean check that happens before your function runs. If the check fails, the whole transaction reverts (like you never tried).
> 
> So yes, they're literally just booleans. But instead of checking one variable, you're checking properties of "who's calling and under what conditions."

---

## 4. Gas as a Resource Constraint

Every operation has a cost:

$$\mathcal{G}: \text{Operation} \to \mathbb{N}$$

Examples:
- ADD: $\mathcal{G}(\text{ADD}) = 3$
- SSTORE (storage write): $\mathcal{G}(\text{SSTORE}) = 20000$ (cold) or $5000$ (warm)
- Memory expansion: $\mathcal{G}(n) \propto n^2$ (quadratic cost)

A transaction has gas limit $G_{\max}$, and the function must satisfy:

$$\sum_{i=1}^{k} \mathcal{G}(\text{op}_i) \leq G_{\max}$$

**Intuition:** Gas is like "CPU credits." Every instruction spends some. Storage is super expensive because all nodes must keep it forever.

---

## 5. Data Locations: Different Cost Models

Think of data locations as different memory hierarchies:

| Location | Access Cost | Persistence | Mathematical Model |
|----------|-------------|-------------|-------------------|
| Storage | High ($\approx 20k$ gas) | Permanent | $S$ (state space) |
| Memory | Medium ($\approx 3$ gas) | Temporary | $M$ (stack/heap during execution) |
| Calldata | Low ($\approx 0$ gas) | Read-only input | $I$ (immutable input) |

**Example:** Array operations

```solidity
function badLoop(uint[] memory data) public {
    for (uint i = 0; i < data.length; i++) {
        myArray.push(data[i]);  // Each push: ~20k gas
    }
}
```

Cost: $\mathcal{G}_{\text{bad}}(n) = 20000n$

```solidity
function goodLoop(uint[] calldata data) external {
    uint len = data.length;      // Cache length: ~100 gas
    for (uint i = 0; i < len; i++) {
        myArray.push(data[i]);   // Still 20k per push
    }
}
```

Cost: $\mathcal{G}_{\text{good}}(n) = 100 + 20000n$ (slightly better, but the real win is using `calldata` over `memory`)

**Intuition:** Minimize state writes (storage). Use memory for temporary stuff. Use calldata for inputs you don't need to modify.

---

## 6. Mappings as Sparse Arrays

A mapping is a hash table with default values:

$$\text{mapping}: K \to V \quad \text{with default} \quad v_0$$

Mathematically:
$$m: K \to V \text{ where } m(k) = \begin{cases} \text{stored value} & \text{if } k \text{ was written} \\ v_0 & \text{otherwise} \end{cases}$$

**Key insight:** Mappings don't store empty entries. Lookup is always $O(1)$ with fixed gas cost.

```solidity
mapping(address => uint256) public balances;
```

This is:
$$\text{balances}: A \to \mathbb{Z}_{2^{256}} \text{ with default } 0$$

**Intuition:** Mappings are like infinite sparse arrays where unwritten values cost nothing to store. That's why they're cheap and can't be iterated.

---

## 7. Events as Projections

Events don't modify state—they're just logged outputs:

$$\text{event}: S \times C \to \text{Log}$$

They project relevant data to an external append-only log.

```solidity
event Transfer(address indexed from, address indexed to, uint256 amount);
emit Transfer(msg.sender, to, amount);
```

This is:
$$\pi_{\text{Transfer}}(s, c) = (c.\text{sender}, \text{to}, \text{amount})$$

appended to the transaction log.

**Intuition:** Events are like printf statements that get saved to an append-only database. They're cheap because they don't affect the state $S$ that all nodes must maintain.

---

## 8. Inheritance as Function Composition

Inheritance combines multiple contracts:

$$\mathcal{C}_{\text{child}} = \mathcal{C}_{\text{parent}} \oplus \mathcal{C}_{\text{override}}$$

where $\oplus$ means "merge with override."

```solidity
contract Animal {
    function speak() virtual returns (string) { return "..."; }
}
contract Dog is Animal {
    function speak() override returns (string) { return "Woof"; }
}
```

becomes:
$$\text{Dog.speak} = \text{override}(\text{Animal.speak})$$

**Intuition:** Child contracts replace parent functions. The combined state space is the union of all inherited state variables.

---

## 9. The Fundamental Design Patterns

### Pattern 1: Checks-Effects-Interactions

Every function should follow:
1. **Check:** $\tau(s, c)$ (verify preconditions)
2. **Effect:** $s' = f(s)$ (update state)
3. **Interact:** external calls (can fail, but state is safe)

Why? To prevent reentrancy: if you interact before updating state, an external contract can call back and exploit the old state.

$$\text{function } f(s, c) = \begin{cases}
s & \text{if } \neg\tau(s,c) \\
\text{interact}(s') & \text{where } s' = \text{effect}(s)
\end{cases}$$

### Pattern 2: Withdrawal Pattern

Instead of push: $s' = s[\text{recipient.balance} += x]$ (dangerous, can fail)

Use pull: 
- $s' = s[\text{pending}[\text{recipient}] += x]$
- Let recipient call: $\text{withdraw}() \to s'' = s'[\text{pending}[\text{sender}] = 0]$

**Intuition:** Don't send money to arbitrary contracts directly (they might reject or reenter). Let them pull it themselves.

---

## 10. The Security Invariant

A secure contract maintains invariants:

$$\forall f \in F, \forall s \in S, \forall c \in C: \quad \mathcal{I}(s) \implies \mathcal{I}(f(s, c))$$

where $\mathcal{I}$ is an invariant predicate.

**Example:** Token total supply invariant

$$\mathcal{I}(s) \equiv \sum_{a \in A} s.\text{balances}[a] = s.\text{totalSupply}$$

Every function must preserve this.

**Intuition:** Invariants are properties that should always be true. Functions are correct if they never break these properties.

---

## The Synthesis

Putting it together, Solidity is:

$$\boxed{\text{Smart Contract} = \text{State Machine} + \text{Access Control} + \text{Cost Model}}$$

- **State Machine:** $S$ (storage), $F$ (functions as transition operators)
- **Access Control:** $\tau$ (predicates on msg.sender)
- **Cost Model:** $\mathcal{G}$ (gas costs force you to optimize)

Everything else—events, modifiers, inheritance, patterns—is just syntactic sugar over this core model.

---

## Why This Helps

Instead of memorizing syntax like `public`, `memory`, `payable`, you understand:
- `public` = $\tau(c) = \text{true}$ (no access restriction)
- `memory` = temporary data in $M$, not in $S$
- `payable` = function accepts $c.\text{value} > 0$

The math reveals the **structure beneath the syntax**. Once you see contracts as state machines with cost constraints, the language becomes way less arbitrary.

---

**Bottom line:** Solidity is just applied automata theory with a price tag. Every line either changes state, restricts access, or costs gas. That's it.