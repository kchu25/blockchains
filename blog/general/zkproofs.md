@def title = "Zero-Knowledge Proofs: The Math, The Intuition, and a Julia Playground"
@def published = "21 February 2026"
@def tags = ["general", "code"]

# Zero-Knowledge Proofs: The Math, The Intuition, and a Julia Playground

**TL;DR:** A zero-knowledge proof lets you convince someone that a statement is true *without revealing why it's true*. This post builds the theory from scratch with formal definitions, then implements a toy ZK system in Julia that you can actually run, break, and extend.

---

## What Is a Zero-Knowledge Proof?

Imagine you're colorblind and I want to prove to you that two balls are different colors—without telling you which is red and which is green.

Here's the protocol:
1. You hold one ball in each hand behind your back.
2. You either swap them or don't (your choice, hidden from me).
3. You show me the balls again. I tell you whether you swapped.

If the balls were the same color, I'd have to guess—50% chance. But if they're truly different colors, I can always tell. Repeat this 100 times, and the probability I'm lying drops to $2^{-100} \approx 10^{-30}$. You're convinced they're different colors, but you still don't know which is red and which is green.

That's zero-knowledge in a nutshell: **conviction without information transfer.**

> **Why should I care about this for blockchains?**
>
> Three killer applications:
> 1. **Privacy**: Prove you have enough funds to make a transaction without revealing your balance (Zcash, Tornado Cash)
> 2. **Scalability**: Prove that 10,000 transactions were executed correctly with a single tiny proof that anyone can verify quickly (ZK-Rollups like zkSync, StarkNet)
> 3. **Identity**: Prove you're over 18 without revealing your birthdate, prove you're a citizen without revealing your passport number
>
> ZK proofs are arguably the most important cryptographic primitive for the future of blockchains. They solve the blockchain trilemma's scalability leg without sacrificing security.

---

## The Formal Definition (Every Symbol Explained)

Let's be precise—but let me first explain every piece of notation so nothing is mysterious.

### The Setup: Statements, Languages, and Witnesses

A ZK proof is about a **statement**. In math, we call the statement $x$ (just a label—it could be a number, a string, a data structure, anything). The collection of all *true* statements is called the **language** $L$.

> **Concrete example to ground every symbol:**
>
> Suppose the statement is: "I know a password whose SHA-256 hash is `0xABCD...`"
>
> - $x$ = the public hash value `0xABCD...` (this is the **statement** — the thing everyone can see)
> - $L$ = the set of all hash values that *actually have* a preimage (i.e., all valid hashes someone could know a password for)
> - $x \in L$ means "this hash really does correspond to some password" (the statement is true)
> - $x \notin L$ means "this hash has no valid preimage" (the statement is false)
> - The **witness** $w$ = the actual password itself (the secret that *proves* the statement is true)
>
> The prover knows both $x$ (the hash) and $w$ (the password). The verifier only sees $x$ (the hash). The ZK proof convinces the verifier that a valid password exists, *without revealing the password*.

A **zero-knowledge proof system** for a language $L$ is an interactive protocol between two parties:

- **Prover** $\mathcal{P}$: knows the statement $x$ *and* the secret witness $w$. Wants to convince the verifier that $x \in L$.
- **Verifier** $\mathcal{V}$: knows only the statement $x$. Wants to be convinced, but should learn nothing beyond "$x \in L$" (the statement is true).

The protocol must satisfy three properties:

### 1. Completeness

If the statement is true and both parties follow the protocol honestly, the verifier always accepts.

$$x \in L \implies \Pr\bigl[\mathcal{P} \leftrightarrow \mathcal{V} \text{ accepts on input } x\bigr] = 1$$

*"An honest prover can always convince an honest verifier."*

> **Notation clarification:** $\mathcal{P} \leftrightarrow \mathcal{V}$ just means "prover and verifier run the interactive protocol together." Some textbooks write $\langle \mathcal{P}, \mathcal{V} \rangle(x)$ for this—the angle brackets $\langle \cdot, \cdot \rangle$ denote an **interactive session** between the two parties (it is NOT an inner product here). The $(x)$ means both parties receive the public statement $x$ as input. The whole expression evaluates to either "accept" or "reject."

### 2. Soundness

If the statement is false, no cheating prover can trick the verifier (except with negligible probability).

$$x \notin L \implies \forall\, \mathcal{P}^*:\; \Pr\bigl[\mathcal{P}^* \leftrightarrow \mathcal{V} \text{ accepts on input } x\bigr] \leq \epsilon$$

where $\epsilon$ is the **soundness error** (probability of a cheater getting lucky).

> **What's $\mathcal{P}^*$?** The star means a **malicious** (cheating) prover—one that doesn't follow the protocol and tries any trick to fool the verifier. The statement says: even if the cheating prover has unlimited computational power and uses *any strategy whatsoever* ($\forall \mathcal{P}^*$), it still can't convince the verifier of a false statement (except with tiny probability $\epsilon$). For protocols with multiple rounds, $\epsilon \leq 2^{-n}$ after $n$ rounds—it shrinks exponentially.

*"A liar gets caught."*

### 3. Zero-Knowledge

The verifier learns nothing beyond "the statement is true." Here's the formal way to say this:

There exists a **simulator** $\mathcal{S}$—an algorithm that does NOT know the secret witness $w$—that can produce fake conversation transcripts that look *identical* to real ones.

> **What's a "transcript"?** It's just the sequence of messages exchanged: what the prover sent, what the verifier sent, and the final accept/reject. Think of it as a chat log.

$$\forall\, \mathcal{V}^*: \quad \mathsf{view}(\mathcal{P} \leftrightarrow \mathcal{V}^*,\, x) \approx \mathcal{S}(x)$$

> **What does this mean, piece by piece?**
>
> - $\mathcal{V}^*$ = a possibly **malicious verifier** (the star means "cheating," just like $\mathcal{P}^*$ for the prover). We want zero-knowledge to hold even if the verifier tries tricky strategies to extract information.
> - $\mathsf{view}(\mathcal{P} \leftrightarrow \mathcal{V}^*, x)$ = the full chat log of a real protocol run between the honest prover and the (possibly cheating) verifier.
> - $\mathcal{S}(x)$ = a fake transcript generated by the simulator, which only knows the public statement $x$ and does NOT know the witness $w$.
> - $\approx$ = the two transcripts are **indistinguishable** (no efficient algorithm can tell which is real and which is simulated).
>
> **The punchline:** If a simulator can produce transcripts that look identical to real ones *without knowing the secret*, then the real transcripts can't contain any information about the secret. The verifier gained nothing it couldn't have computed on its own.
>
> **Analogy:** It's like proving a magic trick reveals no secrets by showing that a non-magician could produce the same audience experience with video editing. If the audience can't tell the difference, the magic show didn't actually reveal anything.

---

## Warm-Up: A Simple ZK Proof With Just Arithmetic

Before we get to "real" cryptographic ZK proofs, let's build intuition with a simpler example using nothing but modular arithmetic. No cyclic groups, no generators—just numbers.

### The "Secret Number" Game

Suppose I claim to know a secret number $w$ such that $w^2 \mod 77 = 4$. The public statement is "4" and the number 77. The witness is my secret $w$.

> There are actually multiple solutions here (e.g., $w = 2$ since $2^2 = 4$, or $w = 75$ since $75^2 = 5625 = 73 \times 77 + 4$). That's fine—I just need to know *one*.

Here's a protocol:

1. **I commit**: I pick a random number $r$, compute $a = r^2 \mod 77$, and send you $a$.
2. **You challenge**: You flip a coin and send me $e \in \{0, 1\}$.
3. **I respond**:
   - If $e = 0$: I send $z = r \mod 77$. You check $z^2 \equiv a \pmod{77}$.
   - If $e = 1$: I send $z = r \cdot w \mod 77$. You check $z^2 \equiv a \cdot 4 \pmod{77}$.

**Why this works:**
- **Completeness**: If I know $w$, I can always answer correctly.
- **Soundness**: If I *don't* know $w$, I can answer $e=0$ (by showing $r$) or $e=1$ (by guessing), but not both for the same commitment $a$. So I cheat with probability $\leq 1/2$. Repeat 100 times → cheating probability $\leq 2^{-100}$.
- **Zero-knowledge**: Each round, you see either $r$ (random, reveals nothing about $w$) or $r \cdot w$ (also random-looking, since $r$ is random). You never see $w$ itself.

This is simpler but has a drawback: the soundness error is $1/2$ per round, so you need ~100 rounds for strong security. The Schnorr protocol below gets negligible soundness error in a *single* round by using a larger challenge space.

---

## The Math: Schnorr's Identification Protocol

Now let's build a real ZK proof. **Schnorr's protocol** proves "I know a secret number $x$ that corresponds to a public value $y$" without revealing $x$. The core idea is the same as our warm-up, but using exponentiation modulo a prime instead of squaring.

### The One-Way Function We'll Use

We need a function that's easy to compute forward but hard to reverse. We'll use **modular exponentiation**:

$$f(x) = g^x \mod p$$

where $p$ is a large prime number and $g$ is a fixed base.

> **Why is this hard to reverse?** Computing $g^x \mod p$ is fast (repeated squaring: $O(\log x)$ multiplications). But given $y = g^x \mod p$, finding $x$ is the **discrete logarithm problem**—no known efficient algorithm exists for large primes. The best algorithms are sub-exponential: roughly $e^{O(\sqrt{\log p \cdot \log \log p})}$ operations. For a 2048-bit prime, this is astronomically infeasible.
>
> You can think of $g^x \mod p$ as a "hash" of $x$ that preserves algebraic structure. Unlike SHA-256, which destroys structure, modular exponentiation lets you do useful math with the "hashed" values—which is exactly what makes ZK proofs possible.

### What's a Cyclic Group? (And Why Do We Need One?)

When we compute $g^x \mod p$ for $x = 0, 1, 2, 3, \ldots$, the results eventually cycle back to the beginning:

$$g^0 \mod p = 1, \quad g^1 \mod p, \quad g^2 \mod p, \quad \ldots, \quad g^{q-1} \mod p, \quad g^q \mod p = 1 \text{ (back to start)}$$

This set of values $\{g^0, g^1, \ldots, g^{q-1}\}$ (all computed mod $p$) forms a **cyclic group** of order $q$. "Cyclic" just means it wraps around. "Order $q$" means there are $q$ distinct elements before it repeats.

> **Why do we care?** We need the exponents to "wrap around" in a controlled way (modulo $q$). When the prover computes $z = r + e \cdot x$, this arithmetic happens modulo $q$—the group order. If $q$ weren't prime, there could be shortcuts to find $x$. A prime-order group gives us maximum security.

### The Setup

We pick:
- A large prime $p$ (defines our modular arithmetic)
- A prime $q$ that divides $p - 1$ (the order of our cyclic subgroup)
- A generator $g$ (a number whose powers produce the entire subgroup)

Then:
- **Prover's secret** (witness): a random number $x$ from $\{1, 2, \ldots, q-1\}$
- **Public statement**: $y = g^x \mod p$ (easy to compute, hard to reverse)

### The Protocol: Three Moves

The Schnorr protocol is a **Sigma protocol** (Σ-protocol)—a three-move structure that shows up everywhere in cryptography:

$$\mathcal{P} \xrightarrow{\text{commit } a} \mathcal{V} \xrightarrow{\text{challenge } e} \mathcal{P} \xrightarrow{\text{respond } z} \mathcal{V}$$

Here's what happens:

**Step 1 — Commitment (Prover → Verifier):**

The prover picks a random $r$ uniformly from $\{1, 2, \ldots, q-1\}$ and sends:
$$a = g^r \mod p$$

> **Notation:** Some textbooks write $r \xleftarrow{\$} \mathbb{Z}_q$ for "pick $r$ randomly from the set $\{0, 1, \ldots, q-1\}$." The $\$$ sign means "sample uniformly at random" and $\mathbb{Z}_q$ is just the set of integers modulo $q$. We'll use the plain-English version.

The randomness $r$ is critical—it's what makes the protocol zero-knowledge. Without it, the verifier could extract $x$ from a single response.

**Step 2 — Challenge (Verifier → Prover):**

The verifier picks a random challenge $e$ uniformly from $\{1, 2, \ldots, q-1\}$.

> The verifier's challenge is like a pop quiz. The prover committed to $a$ *before* seeing the challenge, so they can't tailor their commitment to a specific question. This is what makes cheating hard. Note the challenge space is $\{1, \ldots, q-1\}$—much larger than the coin flip $\{0, 1\}$ in our warm-up. That's why Schnorr gets negligible soundness error ($1/q$) in a single round.

**Step 3 — Response (Prover → Verifier):**

The prover computes and sends:
$$z = r + e \cdot x \mod q$$

**Verification:**

The verifier checks:
$$g^z \stackrel{?}{=} a \cdot y^e \mod p$$

### Why Does Verification Work?

Let's expand the right side:

$$a \cdot y^e = g^r \cdot (g^x)^e = g^r \cdot g^{xe} = g^{r + xe} = g^z \mod p$$

The equation holds if and only if $z = r + ex \mod q$, which only someone who knows $x$ can compute (given that they committed to $r$ before seeing $e$).

### Why Is It Zero-Knowledge?

Here's the simulator argument. The simulator $\mathcal{S}$ (which does **not** know the secret $x$) works as follows:

1. Pick random $z$ and $e$ uniformly from $\{1, 2, \ldots, q-1\}$
2. Compute $a = g^z \cdot y^{-e} \mod p$ (note: working *backwards* from $z$ and $e$ to find $a$)
3. Output the transcript $(a, e, z)$

**Check:** Does $g^z = a \cdot y^e$?

$$a \cdot y^e = g^z \cdot y^{-e} \cdot y^e = g^z \quad \checkmark$$

The simulated transcript has the exact same distribution as a real transcript! The verifier can't tell the difference. Therefore, the real transcript reveals nothing about $x$.

> **Wait—the simulator picks $e$ and $z$ first, then computes $a$ backwards. In the real protocol, $a$ comes first. Doesn't the order matter?**
>
> Great catch. In the real protocol, $a$ is committed *before* $e$ is chosen, so the prover can't cheat. But the simulator doesn't need to cheat—it's not trying to extract $x$. It's just showing that the *final transcript* $(a, e, z)$ looks the same regardless of whether $x$ was used.
>
> The key property is that **the distribution of transcripts is identical**. The verifier only sees the final $(a, e, z)$, not the order in which values were generated internally.

### Why Is It Sound?

Recall $\mathcal{P}^*$ denotes a cheating prover (the star = "malicious, doesn't follow the rules"). If $\mathcal{P}^*$ doesn't actually know $x$, they'd need to produce a valid $z$ for a random $e$ they haven't seen yet. Since they committed to $a = g^r$ before seeing $e$, they need:

$$z = r + e \cdot x \mod q$$

Without knowing $x$, they can only guess correctly with probability $1/q$ (negligible for large $q$—compare to the warm-up where the cheating probability was $1/2$ per round).

More formally, if a prover can answer two different challenges $e_1, e_2$ for the same commitment $a$:

$$z_1 = r + e_1 \cdot x \mod q$$
$$z_2 = r + e_2 \cdot x \mod q$$

Subtracting: $z_1 - z_2 = (e_1 - e_2) \cdot x \mod q$, so:

$$x = \frac{z_1 - z_2}{e_1 - e_2} \mod q$$

This is the **knowledge extractor**—it proves the protocol has the **proof of knowledge** property. If you can answer two challenges, you *must* know $x$.

---

## The Fiat-Shamir Heuristic: Making It Non-Interactive

Interactive proofs are inconvenient for blockchains—you'd need the verifier to be online and send challenges. The **Fiat-Shamir transform** replaces the verifier's random challenge with a hash:

$$e = H(a \| y \| \text{context})$$

where $H$ is a cryptographic hash function (like SHA-256). The idea: a hash function is "random enough" that the prover can't predict $e$ before committing to $a$.

This turns the 3-move protocol into a single message $(a, z)$ that anyone can verify. This is called a **Non-Interactive Zero-Knowledge (NIZK)** proof.

> **Why does hashing work as a replacement for the verifier?**
>
> A cryptographic hash function behaves like a **random oracle**—its output is unpredictable unless you know the exact input. Since the prover must commit to $a$ before computing $e = H(a \| y)$, and changing $a$ even slightly completely changes $e$, the prover can't "shop around" for a favorable challenge. It's as if the hash function IS the verifier, asking an unpredictable question based on the prover's commitment.
>
> **Caveat**: The Fiat-Shamir heuristic is provably secure in the "random oracle model" but not in the standard model. In practice, it works great with real hash functions. Most ZK systems used in production (SNARKs, STARKs) use this transform or something similar.

---

## Implementation in Julia

Now the fun part. Let's build a complete Schnorr ZK proof system in Julia. No external crypto libraries—we'll do everything from scratch so you can see every moving part.

> **Why Julia?**
>
> Julia has native big integer support (`BigInt`), built-in modular arithmetic via `powermod`, and the syntax is clean enough that the code reads almost like the math. Plus, if you're a CS PhD who already knows Julia, you won't be fighting the language while learning the cryptography.

### Part 1: Parameter Generation

First, we need safe cryptographic parameters. We need a prime $p$ such that $q = (p-1)/2$ is also prime (a **safe prime**), and a generator $g$ of the subgroup of order $q$.

```julia
using SHA
using Random

# --- Parameter Generation ---

"""
Check if n is probably prime using Miller-Rabin.
For a toy implementation, k=20 rounds gives error probability < 4^{-20} ≈ 10^{-12}.
"""
function is_probably_prime(n::BigInt; k::Int=20)
    if n < 2
        return false
    end
    if n == 2 || n == 3
        return true
    end
    if iseven(n)
        return false
    end

    # Write n-1 = 2^s * d with d odd
    d = n - 1
    s = 0
    while iseven(d)
        d >>= 1
        s += 1
    end

    # k rounds of Miller-Rabin
    for _ in 1:k
        a = rand(big(2):n-2)
        x = powermod(a, d, n)

        if x == 1 || x == n - 1
            continue
        end

        found = false
        for _ in 1:s-1
            x = powermod(x, 2, n)
            if x == n - 1
                found = true
                break
            end
        end

        if !found
            return false
        end
    end
    return true
end

"""
Generate a safe prime p = 2q + 1 where q is also prime.
`bits` controls the size. Use small values (64-256) for experimentation,
1024+ for anything resembling real security.
"""
function generate_safe_prime(bits::Int)
    while true
        # Generate random odd number of appropriate size
        q = rand(big(2)^(bits-1):big(2)^bits - 1)
        if iseven(q)
            q += 1
        end

        if !is_probably_prime(q)
            continue
        end

        p = 2q + 1
        if is_probably_prime(p)
            return (p=p, q=q)
        end
    end
end

"""
Find a generator of the subgroup of order q in Z_p^*.
If p = 2q + 1 (safe prime), then for any h in Z_p^*,
g = h^2 mod p is a generator of the order-q subgroup
(unless g = 1, which happens with negligible probability).
"""
function find_generator(p::BigInt, q::BigInt)
    while true
        h = rand(big(2):p-2)
        g = powermod(h, 2, p)
        if g != 1
            # Verify: g^q mod p should be 1 (g has order q)
            @assert powermod(g, q, p) == 1 "Generator check failed"
            return g
        end
    end
end
```

> **Why safe primes?**
>
> A safe prime $p = 2q + 1$ ensures that the multiplicative group $\mathbb{Z}_p^*$ (that's just the set $\{1, 2, \ldots, p-1\}$ with multiplication mod $p$) has a large prime-order subgroup of order $q$. This matters because:
>
> 1. The discrete log problem is hardest in prime-order subgroups
> 2. Every element $g \neq 1$ in this subgroup is a generator (no "weak" generators)
> 3. The Schnorr protocol's security proof requires a prime-order group
>
> Without a safe prime, $p - 1$ could have small factors, and an attacker could use the [Pohlig-Hellman algorithm](https://en.wikipedia.org/wiki/Pohlig%E2%80%93Hellman_algorithm) to decompose the discrete log problem into easier sub-problems.

### Part 2: Key Generation and the Proof System

```julia
# --- Data Structures ---

"""
Public parameters shared by everyone.
These define the group we work in.
"""
struct ZKParams
    p::BigInt  # Safe prime
    q::BigInt  # Prime order of subgroup (p = 2q + 1)
    g::BigInt  # Generator of the order-q subgroup
end

"""
A non-interactive Schnorr proof (after Fiat-Shamir).
Contains the commitment `a` and the response `z`.
The challenge `e` is recomputed by the verifier via hashing.
"""
struct SchnorrProof
    a::BigInt  # Commitment: g^r mod p
    z::BigInt  # Response: r + e*x mod q
end

"""
Generate a keypair: secret x, public y = g^x mod p.
"""
function keygen(params::ZKParams)
    x = rand(big(1):params.q - 1)  # Secret key
    y = powermod(params.g, x, params.p)  # Public key
    return (secret=x, public=y)
end
```

### Part 3: The Fiat-Shamir Hash

We need a hash function that maps commitments to challenges in $\mathbb{Z}_q$. We'll use SHA-256:

```julia
"""
Fiat-Shamir challenge: e = H(g, y, a) mod q.

This replaces the verifier's random challenge in the interactive protocol.
The hash input includes all public values to prevent cross-protocol attacks.
"""
function fiat_shamir_challenge(params::ZKParams, y::BigInt, a::BigInt)
    # Concatenate all public values into a byte string
    data = string(params.g) * "|" * string(y) * "|" * string(a)
    hash_bytes = sha256(Vector{UInt8}(data))

    # Convert hash to BigInt and reduce mod q
    e = parse(BigInt, bytes2hex(hash_bytes), base=16)
    return mod(e, params.q)
end
```

> **Why hash all three values $(g, y, a)$?**
>
> Including $g$ and $y$ in the hash prevents **related-key attacks** where a proof for one key is repurposed for another. The hash should bind the challenge to the *entire context* of the proof. In production systems, you'd also include a domain separator string (like `"Schnorr-v1"`) to prevent cross-protocol attacks.

### Part 4: Prove and Verify

```julia
"""
Generate a Schnorr proof that you know x such that y = g^x mod p.

This is the prover's algorithm:
1. Pick random r, compute commitment a = g^r mod p
2. Compute challenge e = H(g, y, a) mod q  (Fiat-Shamir)
3. Compute response z = r + e*x mod q
4. Output proof (a, z)
"""
function prove(params::ZKParams, x::BigInt, y::BigInt)
    # Step 1: Commitment
    r = rand(big(1):params.q - 1)  # Random nonce
    a = powermod(params.g, r, params.p)

    # Step 2: Challenge (Fiat-Shamir)
    e = fiat_shamir_challenge(params, y, a)

    # Step 3: Response
    z = mod(r + e * x, params.q)

    return SchnorrProof(a, z)
end

"""
Verify a Schnorr proof: check that g^z ≡ a · y^e (mod p).

This is the verifier's algorithm:
1. Recompute the challenge e = H(g, y, a) mod q
2. Check: g^z mod p == a * y^e mod p
"""
function verify(params::ZKParams, y::BigInt, proof::SchnorrProof)
    # Recompute challenge
    e = fiat_shamir_challenge(params, y, proof.a)

    # Verify: g^z ≡ a · y^e (mod p)
    lhs = powermod(params.g, proof.z, params.p)
    rhs = mod(proof.a * powermod(y, e, params.p), params.p)

    return lhs == rhs
end
```

### Part 5: Let's Run It!

```julia
function demo_basic()
    println("=" ^ 60)
    println("  Schnorr Zero-Knowledge Proof Demo")
    println("=" ^ 60)

    # Generate parameters (128-bit primes for demo; use 1024+ for real security)
    println("\n[1] Generating safe prime parameters (128-bit)...")
    primes = generate_safe_prime(128)
    params = ZKParams(primes.p, primes.q, find_generator(primes.p, primes.q))
    println("    p = $(params.p)")
    println("    q = $(params.q)")
    println("    g = $(params.g)")

    # Generate keypair
    println("\n[2] Generating keypair...")
    keys = keygen(params)
    println("    Secret x = $(keys.secret)")
    println("    Public y = g^x mod p = $(keys.public)")

    # Generate proof
    println("\n[3] Generating proof (proving knowledge of x)...")
    proof = prove(params, keys.secret, keys.public)
    println("    Commitment a = $(proof.a)")
    println("    Response   z = $(proof.z)")

    # Verify proof
    println("\n[4] Verifying proof...")
    result = verify(params, keys.public, proof)
    println("    Verification: $(result ? "✅ ACCEPTED" : "❌ REJECTED")")

    # Try with wrong key (soundness check)
    println("\n[5] Soundness check: verify with wrong public key...")
    fake_y = powermod(params.g, rand(big(1):params.q-1), params.p)
    result_fake = verify(params, fake_y, proof)
    println("    Verification with wrong key: $(result_fake ? "❌ UNSOUND!" : "✅ REJECTED (correct)")")

    println("\n" * "=" ^ 60)
end

demo_basic()
```

Running this should produce output like:

```
============================================================
  Schnorr Zero-Knowledge Proof Demo
============================================================

[1] Generating safe prime parameters (128-bit)...
    p = 54890381048102939...
    q = 27445190524051469...
    g = 38271940182739401...

[2] Generating keypair...
    Secret x = 19384710293847102...
    Public y = g^x mod p = 47261839401827364...

[3] Generating proof (proving knowledge of x)...
    Commitment a = 29381047291038471...
    Response   z = 38291047382910473...

[4] Verifying proof...
    Verification: ✅ ACCEPTED

[5] Soundness check: verify with wrong public key...
    Verification with wrong key: ✅ REJECTED (correct)

============================================================
```

---

## Going Deeper: Multiple Rounds and Statistical Confidence

A single Schnorr proof already has negligible soundness error ($1/q$), but for simpler protocols (like graph 3-coloring), you need multiple rounds. Let's build a generic framework:

```julia
"""
Run the Schnorr proof `n` times independently and track statistics.

In Schnorr's protocol, a single round already has soundness error 1/q (negligible).
But this demonstrates the general principle: n independent rounds give
soundness error (1/q)^n.

For simpler ZK protocols with soundness error 1/2 per round,
you genuinely need n ≈ 128 rounds for 128-bit security.
"""
function demo_multiple_rounds(n::Int=20)
    println("\n" * "=" ^ 60)
    println("  Multiple-Round Demonstration ($n rounds)")
    println("=" ^ 60)

    primes = generate_safe_prime(128)
    params = ZKParams(primes.p, primes.q, find_generator(primes.p, primes.q))
    keys = keygen(params)

    accept_count = 0
    for i in 1:n
        proof = prove(params, keys.secret, keys.public)
        if verify(params, keys.public, proof)
            accept_count += 1
        end
    end

    println("  Honest prover: $accept_count / $n rounds accepted")
    println("  (Should be $n / $n — completeness)")

    # Now try cheating: prover doesn't know x, tries random z
    cheat_count = 0
    for i in 1:n
        # Cheating prover: picks random a and z, hopes for the best
        fake_a = powermod(params.g, rand(big(1):params.q-1), params.p)
        fake_z = rand(big(1):params.q - 1)
        fake_proof = SchnorrProof(fake_a, fake_z)
        if verify(params, keys.public, fake_proof)
            cheat_count += 1
        end
    end

    println("  Cheating prover: $cheat_count / $n rounds accepted")
    println("  (Should be ≈ 0 — soundness)")
    println("  Probability of cheating: 1/q ≈ $(Float64(1/BigFloat(params.q)))")
    println("=" ^ 60)
end

demo_multiple_rounds(50)
```

---

## Experiment 1: What Happens If the Prover Reuses Randomness?

This is a classic cryptographic catastrophe. If the prover uses the same $r$ for two different proofs, the secret key leaks!

```julia
"""
Demonstrate the CRITICAL importance of fresh randomness.

If the prover reuses the same nonce r for two proofs with different
challenges e₁ and e₂, an attacker can extract the secret key:

    z₁ = r + e₁·x mod q
    z₂ = r + e₂·x mod q
    ───────────────────
    z₁ - z₂ = (e₁ - e₂)·x mod q
    x = (z₁ - z₂) · (e₁ - e₂)⁻¹ mod q

This is EXACTLY what happened to Sony's PlayStation 3 ECDSA signing key
in 2010 — they used a fixed random number, and hackers extracted the
private key, allowing them to sign arbitrary code.
"""
function demo_nonce_reuse_attack()
    println("\n" * "=" ^ 60)
    println("  ⚠️  Nonce Reuse Attack Demo")
    println("=" ^ 60)

    primes = generate_safe_prime(128)
    params = ZKParams(primes.p, primes.q, find_generator(primes.p, primes.q))
    keys = keygen(params)
    println("  Secret key x = $(keys.secret)")

    # Prover MISTAKENLY reuses the same r for two proofs
    r = rand(big(1):params.q - 1)  # Same r used twice!
    a = powermod(params.g, r, params.p)  # Same commitment

    # But we compute e differently by adding context to change the hash.
    # In practice, this happens when signing two different messages
    # with the same nonce.

    # Proof 1: standard
    e1 = fiat_shamir_challenge(params, keys.public, a)
    z1 = mod(r + e1 * keys.secret, params.q)

    # Proof 2: simulate a different challenge by using a different "context"
    # (In reality, this happens when you sign a different message with same r)
    data2 = string(params.g) * "|" * string(keys.public) * "|" * string(a) * "|msg2"
    hash2 = sha256(Vector{UInt8}(data2))
    e2 = mod(parse(BigInt, bytes2hex(hash2), base=16), params.q)
    z2 = mod(r + e2 * keys.secret, params.q)

    println("\n  Attacker observes two proofs with same commitment a:")
    println("    Proof 1: (a, e₁=$(e1), z₁=$(z1))")
    println("    Proof 2: (a, e₂=$(e2), z₂=$(z2))")

    # Attack: extract x from the two transcripts
    delta_z = mod(z1 - z2, params.q)
    delta_e = mod(e1 - e2, params.q)
    x_recovered = mod(delta_z * invmod(delta_e, params.q), params.q)

    println("\n  Attack computation:")
    println("    z₁ - z₂ = $delta_z")
    println("    e₁ - e₂ = $delta_e")
    println("    x = (z₁ - z₂) · (e₁ - e₂)⁻¹ mod q = $x_recovered")
    println("\n  Recovered secret: $x_recovered")
    println("  Actual secret:    $(keys.secret)")
    println("  Match: $(x_recovered == keys.secret ? "🔓 SECRET KEY EXTRACTED!" : "Safe")")

    println("\n  Lesson: NEVER reuse the random nonce r.")
    println("  This is why RFC 6979 exists — deterministic nonce generation.")
    println("=" ^ 60)
end

demo_nonce_reuse_attack()
```

> **This is not hypothetical.** In 2010, [fail0verflow](https://fail0verflow.com/) discovered that Sony used a *constant* random nonce $r$ in their ECDSA implementation for PS3 game signing. The group extracted Sony's private key, enabling arbitrary code signing. The mathematical attack is *identical* to what the code above demonstrates.

---

## Experiment 2: An "OR" Proof — Proving You Know *One Of Two* Secrets

In practice, you often want to prove disjunctions: "I know the secret for account A **or** account B, but I won't tell you which." This is the basis of **ring signatures** (used in Monero for transaction privacy).

The idea: if you know $x_1$ (the discrete log of $y_1$) but not $x_2$ (for $y_2$), you can *simulate* the proof for $y_2$ and give a real proof for $y_1$, such that the verifier can't tell which leg is real.

```julia
"""
OR-Proof: Prove knowledge of x₁ OR x₂ (for y₁ = g^x₁ or y₂ = g^x₂)
without revealing WHICH one you know.

Construction (Cramer-Damgård-Schoenmakers '94):

The prover knows x₁ (WLOG). They:
1. Simulate the proof for y₂: pick random z₂, e₂, compute a₂ = g^z₂ · y₂^(-e₂)
2. Give a real commitment for y₁: pick random r₁, compute a₁ = g^r₁
3. Compute the combined challenge: e = H(a₁, a₂)
4. Set e₁ = e - e₂ mod q (so e₁ + e₂ = e)
5. Compute z₁ = r₁ + e₁·x₁ mod q

The verifier checks:
- g^z₁ = a₁ · y₁^e₁ (real proof)
- g^z₂ = a₂ · y₂^e₂ (simulated, but verifier can't tell!)
- e₁ + e₂ = H(a₁, a₂)   (challenges sum to the hash)
"""
struct ORProof
    a1::BigInt
    a2::BigInt
    e1::BigInt
    e2::BigInt
    z1::BigInt
    z2::BigInt
end

function prove_or(params::ZKParams, x1::BigInt, y1::BigInt, y2::BigInt)
    # Simulate proof for y2 (we DON'T know x2)
    z2 = rand(big(1):params.q - 1)
    e2 = rand(big(1):params.q - 1)
    # a2 = g^z2 * y2^(-e2) mod p  (reverse-engineered to make verification pass)
    a2 = mod(powermod(params.g, z2, params.p) * powermod(y2, params.q - e2, params.p), params.p)

    # Real proof for y1 (we DO know x1)
    r1 = rand(big(1):params.q - 1)
    a1 = powermod(params.g, r1, params.p)

    # Combined challenge
    data = string(a1) * "|" * string(a2) * "|" * string(y1) * "|" * string(y2)
    hash_bytes = sha256(Vector{UInt8}(data))
    e = mod(parse(BigInt, bytes2hex(hash_bytes), base=16), params.q)

    # Split challenge: e1 = e - e2, so e1 + e2 = e
    e1 = mod(e - e2, params.q)

    # Real response
    z1 = mod(r1 + e1 * x1, params.q)

    return ORProof(a1, a2, e1, e2, z1, z2)
end

function verify_or(params::ZKParams, y1::BigInt, y2::BigInt, proof::ORProof)
    # Recompute combined challenge
    data = string(proof.a1) * "|" * string(proof.a2) * "|" * string(y1) * "|" * string(y2)
    hash_bytes = sha256(Vector{UInt8}(data))
    e = mod(parse(BigInt, bytes2hex(hash_bytes), base=16), params.q)

    # Check challenge split
    if mod(proof.e1 + proof.e2, params.q) != e
        return false
    end

    # Check both legs
    lhs1 = powermod(params.g, proof.z1, params.p)
    rhs1 = mod(proof.a1 * powermod(y1, proof.e1, params.p), params.p)

    lhs2 = powermod(params.g, proof.z2, params.p)
    rhs2 = mod(proof.a2 * powermod(y2, proof.e2, params.p), params.p)

    return lhs1 == rhs1 && lhs2 == rhs2
end

function demo_or_proof()
    println("\n" * "=" ^ 60)
    println("  OR-Proof Demo: \"I know x₁ OR x₂\"")
    println("=" ^ 60)

    primes = generate_safe_prime(128)
    params = ZKParams(primes.p, primes.q, find_generator(primes.p, primes.q))

    # Two keypairs — prover only knows x1
    keys1 = keygen(params)
    keys2 = keygen(params)

    println("  Prover knows: x₁ = $(keys1.secret)")
    println("  Prover does NOT know x₂")
    println("  Public keys: y₁ = $(keys1.public)")
    println("               y₂ = $(keys2.public)")

    # Generate OR proof
    proof = prove_or(params, keys1.secret, keys1.public, keys2.public)

    # Verify
    result = verify_or(params, keys1.public, keys2.public, proof)
    println("\n  Verification: $(result ? "✅ ACCEPTED" : "❌ REJECTED")")
    println("  The verifier is convinced the prover knows x₁ or x₂,")
    println("  but CANNOT tell which one!")

    println("=" ^ 60)
end

demo_or_proof()
```

> **Why is this useful on blockchains?**
>
> Ring signatures (used in Monero) are essentially OR-proofs over all possible signers. When you spend Monero, you prove "I own one of these 11 outputs" without revealing which one. The verifier (the network) is convinced the transaction is authorized, but can't link it to your specific account. This is privacy through *plausible deniability*.

---

## Experiment 3: Pedersen Commitments — Hiding and Binding

Before we go further, we need **commitments**—a way to "lock in" a value without revealing it, then open it later. Pedersen commitments are the building block of most ZK systems.

```julia
"""
Pedersen Commitment Scheme.

Setup: Two generators g, h of the order-q subgroup, where nobody knows
log_g(h) (the discrete log of h base g). If someone knew it, they could
open commitments to different values — breaking the binding property.

Commit(v, r) = g^v · h^r mod p

Properties:
- Hiding:  Given C, you can't determine v (because r is random)
- Binding: You can't find (v', r') ≠ (v, r) with Commit(v,r) = Commit(v',r')
           (unless you know log_g(h), which is hard)

This is an information-theoretically hiding, computationally binding scheme.
"""
struct PedersenParams
    p::BigInt
    q::BigInt
    g::BigInt
    h::BigInt  # Second generator, nobody knows log_g(h)
end

struct Commitment
    C::BigInt    # The commitment value g^v · h^r mod p
    v::BigInt    # The committed value (secret until opening)
    r::BigInt    # The randomness (secret until opening)
end

function setup_pedersen(bits::Int=128)
    primes = generate_safe_prime(bits)
    p, q = primes.p, primes.q
    g = find_generator(p, q)
    h = find_generator(p, q)  # Independent generator

    return PedersenParams(p, q, g, h)
end

function commit(pp::PedersenParams, v::BigInt)
    r = rand(big(1):pp.q - 1)
    C = mod(powermod(pp.g, v, pp.p) * powermod(pp.h, r, pp.p), pp.p)
    return Commitment(C, v, r)
end

function open_commitment(pp::PedersenParams, com::Commitment)
    # Verify: C == g^v · h^r mod p
    expected = mod(powermod(pp.g, com.v, pp.p) * powermod(pp.h, com.r, pp.p), pp.p)
    return com.C == expected
end

"""
The magic property: Pedersen commitments are ADDITIVELY HOMOMORPHIC.

    Commit(v₁, r₁) · Commit(v₂, r₂) = Commit(v₁ + v₂, r₁ + r₂)

Proof:
    g^v₁ · h^r₁ · g^v₂ · h^r₂ = g^(v₁+v₂) · h^(r₁+r₂)

This means you can ADD committed values without opening them!
This is the foundation of confidential transactions.
"""
function demo_pedersen()
    println("\n" * "=" ^ 60)
    println("  Pedersen Commitments Demo")
    println("=" ^ 60)

    pp = setup_pedersen(128)

    # Commit to two values
    v1 = big(42)
    v2 = big(58)
    com1 = commit(pp, v1)
    com2 = commit(pp, v2)

    println("  Committed to v₁ = $v1 (commitment: $(com1.C))")
    println("  Committed to v₂ = $v2 (commitment: $(com2.C))")

    # Verify openings
    println("\n  Opening commitments...")
    println("  com1 valid: $(open_commitment(pp, com1) ? "✅" : "❌")")
    println("  com2 valid: $(open_commitment(pp, com2) ? "✅" : "❌")")

    # Homomorphic addition: multiply commitments
    C_sum = mod(com1.C * com2.C, pp.p)
    r_sum = mod(com1.r + com2.r, pp.q)
    v_sum = v1 + v2

    # Verify the sum commitment
    expected_sum = mod(powermod(pp.g, v_sum, pp.p) * powermod(pp.h, r_sum, pp.p), pp.p)

    println("\n  Homomorphic addition (without opening!):")
    println("  C₁ · C₂ mod p = $C_sum")
    println("  Commit(v₁+v₂, r₁+r₂) = $expected_sum")
    println("  Match: $(C_sum == expected_sum ? "✅ Homomorphism works!" : "❌")")
    println("\n  v₁ + v₂ = $v_sum — but the verifier never saw v₁ or v₂!")

    println("=" ^ 60)
end

demo_pedersen()
```

> **Why does homomorphic commitment matter for blockchains?**
>
> In **confidential transactions** (Monero, Mimblewimble/Grin), amounts are replaced by Pedersen commitments. A transaction looks like:
>
> $$C_{\text{input}_1} + C_{\text{input}_2} = C_{\text{output}_1} + C_{\text{output}_2} + C_{\text{fee}}$$
>
> Validators check that commitments balance (inputs = outputs + fee) *without knowing any amounts*. They verify the equation using only the commitments. Combined with range proofs (proving committed values are non-negative), this gives full transaction privacy.

---

## Experiment 4: A Range Proof Sketch

A **range proof** proves that a committed value $v$ lies in $[0, 2^n)$ without revealing $v$. This is critical for confidential transactions—without it, someone could commit to a negative number and create money out of thin air.

The simplest approach: prove each bit of $v$ is 0 or 1, and that they assemble to $v$.

```julia
"""
Simple (naive) range proof: prove that a committed value v is in [0, 2^n).

Idea: Decompose v into bits v = b₀ + 2b₁ + 4b₂ + ... + 2^(n-1)·b_{n-1}
and prove each bᵢ ∈ {0, 1} using an OR-proof.

A commitment to bᵢ ∈ {0,1} can be proven by showing:
  Commit(bᵢ, rᵢ) = g^bᵢ · h^rᵢ is a commitment to 0 OR 1.

If bᵢ = 0: C = h^rᵢ          (so C/g⁰ = h^rᵢ — commitment to 0)
If bᵢ = 1: C = g · h^rᵢ      (so C/g¹ = h^rᵢ — commitment to 1)

We prove: C is a commitment to 0 OR C/g is a commitment to 0.
This is an OR-proof on the discrete log of C (or C/g) base h.

Note: This is a SIMPLIFIED version. Production systems use Bulletproofs
or similar for logarithmic proof size.
"""
function demo_range_proof()
    println("\n" * "=" ^ 60)
    println("  Range Proof Sketch (8-bit)")
    println("=" ^ 60)

    pp = setup_pedersen(128)
    n_bits = 8  # Prove v ∈ [0, 256)

    v = big(173)  # Secret value to prove is in range
    println("  Secret value: v = $v")
    println("  Proving v ∈ [0, $(2^n_bits)) without revealing v")

    # Bit decomposition
    bits = digits(Int(v), base=2, pad=n_bits)  # LSB first
    println("  Bit decomposition: $(reverse(bits)) (MSB first)")

    # Commit to each bit
    bit_commitments = []
    for i in 1:n_bits
        com = commit(pp, big(bits[i]))
        push!(bit_commitments, com)
    end

    # Verify: reconstruct v from bit commitments using homomorphism
    # v = Σ 2^i · bᵢ, so Commit(v, r_total) = Π Commit(bᵢ, rᵢ)^(2^i)
    C_reconstructed = big(1)
    r_total = big(0)
    for i in 1:n_bits
        power = big(2)^(i-1)
        C_reconstructed = mod(C_reconstructed * powermod(bit_commitments[i].C, power, pp.p), pp.p)
        r_total = mod(r_total + power * bit_commitments[i].r, pp.q)
    end

    # This should equal Commit(v, r_total)
    C_v = mod(powermod(pp.g, v, pp.p) * powermod(pp.h, r_total, pp.p), pp.p)

    println("\n  Reconstructed commitment from bits: $C_reconstructed")
    println("  Direct commitment to v=$v:          $C_v")
    println("  Match: $(C_reconstructed == C_v ? "✅ Bit decomposition is consistent!" : "❌")")

    # In a full implementation, we'd also include OR-proofs for each bit
    # showing bᵢ ∈ {0, 1}. This is where Bulletproofs dramatically
    # improve efficiency — from O(n) proofs to O(log n).

    println("\n  In a full range proof, we'd also provide $(n_bits) OR-proofs")
    println("  showing each bit ∈ {0,1}. Total proof size: O($n_bits).")
    println("  Bulletproofs reduce this to O(log₂($n_bits)) = O($(Int(ceil(log2(n_bits))))).")

    println("=" ^ 60)
end

demo_range_proof()
```

---

## The Landscape: Where Does This Fit in ZK Technology?

What we built above (Schnorr proofs, OR-proofs, Pedersen commitments, range proofs) are the **classical** building blocks. Modern ZK systems build on these ideas but scale them dramatically:

| System | Proof Size | Verify Time | Trusted Setup? | Math Foundation |
|--------|-----------|-------------|----------------|-----------------|
| **Schnorr** (what we built) | $O(1)$ elements | $O(1)$ exponentiations | No | Discrete log |
| **Bulletproofs** | $O(\log n)$ | $O(n)$ | No | Discrete log + inner product |
| **Groth16 (SNARKs)** | 3 group elements (~200 bytes!) | 3 pairings (milliseconds) | **Yes** (toxic waste) | Bilinear pairings, QAP |
| **PLONK** | ~500 bytes | Fast | Universal (updatable) | Polynomial commitments |
| **STARKs** | $O(\log^2 n)$ (~100 KB) | $O(\log^2 n)$ | **No** | Hash functions, FRI |

> **What's the "trusted setup" problem?**
>
> Some ZK systems (Groth16) require generating public parameters using secret randomness that must be destroyed afterward. If anyone keeps this "toxic waste," they can forge proofs. This is a trust assumption that many systems try to eliminate:
>
> - **STARKs**: No trusted setup at all (transparent)
> - **PLONK**: Universal setup (one ceremony works for all circuits)
> - **Bulletproofs**: No trusted setup
>
> The trend is moving away from trusted setups entirely.

### The Modern ZK Stack (What's Actually Used in Production)

```
Application Layer
├── zkSync Era (ZK-Rollup on Ethereum, uses PLONK)
├── StarkNet (ZK-Rollup, uses STARKs)
├── Zcash (private transactions, uses Groth16/Halo2)
└── Mina Protocol (succinct blockchain, uses Kimchi/PLONK)

Circuit Languages (how you write ZK programs)
├── Circom (domain-specific, compiles to R1CS)
├── Noir (Rust-like, by Aztec)
├── Cairo (for STARKs, by StarkWare)
├── Leo (for Aleo, privacy-focused)
└── Halo2 (Rust library, by Zcash/EFF)

Proof Systems (the math engines)
├── Groth16 — smallest proofs, needs trusted setup
├── PLONK — universal setup, good balance
├── STARKs — no setup, quantum-resistant, large proofs
└── Bulletproofs — no setup, but slow verification
```

---

## Summary of the Math

Let's collect all the formal objects in one place:

**Schnorr Protocol** (where $x$ is the secret, $y = g^x \bmod p$ is the public value, $q$ is the group order):

$$\text{Setup: } x \in \{1, \ldots, q-1\}, \quad y = g^x \bmod p$$

$$\text{Prove: pick random } r, \quad a = g^r \bmod p, \quad e = H(g, y, a) \bmod q, \quad z = r + ex \bmod q$$

$$\text{Verify: } g^z \stackrel{?}{=} a \cdot y^e \pmod{p}$$

**Pedersen Commitment:**
$$C(v, r) = g^v \cdot h^r \bmod p$$

$$\text{Homomorphism: } C(v_1, r_1) \cdot C(v_2, r_2) = C(v_1 + v_2, r_1 + r_2)$$

**Soundness Extraction (from two transcripts with same $a$):**
$$x = (z_1 - z_2)(e_1 - e_2)^{-1} \bmod q$$

**Fiat-Shamir Transform (interactive → non-interactive):**
$$e_{\text{interactive}} \text{ (random from } \{1,\ldots,q-1\}\text{)} \quad \longrightarrow \quad e_{\text{non-interactive}} = H(a \| \text{public inputs}) \bmod q$$

---

## What To Explore Next

Now that you have a working toy ZK system, here are directions to push it:

1. **Implement Bulletproofs**: Replace the naive range proof with a logarithmic-size inner product argument. This is a significant step up but uses the same Pedersen commitments.

2. **Build a ZK Sudoku Verifier**: Prove you know a Sudoku solution without revealing it. This requires encoding the Sudoku constraints as a set of equations and proving them in zero-knowledge.

3. **Explore Polynomial Commitments**: The foundation of PLONK and modern SNARKs. Commit to a polynomial $f(x)$, then prove evaluations $f(a) = b$ without revealing $f$.

4. **Try Circom + snarkjs**: If you want to go from toy to production, [Circom](https://docs.circom.io/) lets you write ZK circuits that compile to actual SNARK proofs you can verify on Ethereum.

5. **Read the Papers**:
   - Schnorr, 1991: *Efficient Signature Generation by Smart Cards*
   - Pedersen, 1991: *Non-Interactive and Information-Theoretic Secure Verifiable Secret Sharing*
   - Bünz et al., 2018: *Bulletproofs: Short Proofs for Confidential Transactions and More*
   - Gabizon, Williamson, Ciobotaru, 2019: *PLONK*

The code in this post is intentionally simple and unoptimized—it's a playground, not a library. Break it, extend it, and use it to build intuition before diving into production ZK frameworks.
