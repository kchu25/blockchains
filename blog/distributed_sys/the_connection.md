@def title = "Distributed Systems & Blockchains"
@def published = "27 December 2025"
@def tags = ["distributed-systems"]

# Distributed Systems & Blockchains

## What's a Distributed System?

A distributed system is multiple computers (nodes) working together to appear as a single coherent system. Think of it like parallel computing, but instead of cores on one machine, you have independent machines connected by a network.

**The graph view**: You can model any distributed system as a graph where:
- **Nodes** = machines/processes
- **Edges** = communication channels (network links)
- **Edge weights** = latency, or reliability (probability of message delivery)
- **Node states** = current local state of each machine

The key challenges are fundamentally different from single-machine systems:
- **Partial failures**: Some nodes can fail (node removal from graph)
- **Network unreliability**: Edges can fail or have variable latency (edge failures, changing weights)
- **No global clock**: You can't assume synchronized time across nodes
- **Network partitions**: The graph can split into disconnected components

## The Core Problem: Consensus

When you have multiple nodes, how do they agree on a shared state? This is the **consensus problem**.

In ML terms, imagine training a model across multiple machines - you need to agree on parameter updates. But what if some machines are malicious or fail mid-computation?

**Graph interpretation**: Consensus is about reaching agreement across all nodes in the graph despite:
- Node failures (vertices disappearing)
- Message loss (edges failing)
- Network partitions (graph splitting into disconnected components)

The challenge: Each node only has a **local view** of the graph. It doesn't know which other nodes are alive, which messages were lost, or if the graph is partitioned. Yet all nodes must converge to the same decision.

### The CAP Theorem

You can only pick 2 of 3:
- **C**onsistency: All nodes see the same data
- **A**vailability: Every request gets a response
- **P**artition tolerance: System works despite network splits

Since network partitions *will* happen, you're really choosing between C and A.

**Graph view**: 
- **Partition** = graph splits into disconnected components
- **Consistency** = all reachable nodes in the graph have the same state
- **Availability** = every reachable node can respond to queries

When the graph partitions, you must choose: Do nodes in different components give the same answer (C) or do they all respond, possibly with different answers (A)?

## Key Protocols

### Two-Phase Commit (2PC)
Coordinator asks all nodes: "Can you commit?" Then either commits or aborts everywhere.

Problem: If coordinator fails after some nodes commit, you're stuck.

**Graph view**: This is a star topology - coordinator at center, all others at edges. Single point of failure at the hub. If hub fails, the graph becomes disconnected and protocol stalls.

### Paxos & Raft
More fault-tolerant consensus algorithms. Raft is basically "Paxos for humans" - easier to understand and implement.

Key idea: Elect a leader, leader decides the order of operations, followers replicate. If leader fails, elect a new one.

> **What's Raft?**
> 
> Raft is a consensus algorithm designed explicitly to be understandable. The authors literally said "Paxos is too hard to teach and implement, let's make something clearer."
> 
> **The core idea**: Strong leader model with three explicit roles:
> - **Leader**: Handles all client requests, tells everyone else what to do
> - **Follower**: Passively replicates what the leader says
> - **Candidate**: Trying to become the new leader during an election
> 
> **Graph view**: Raft creates a **dynamic star topology**:
> - Leader is the hub, followers are spokes
> - All communication flows through the leader (hub)
> - When leader fails, the graph reconfigures with a new hub via election
> - This is why it's simpler than Paxos - the topology is always a star, just with different centers over time
> 
> **How it works:**
> 
> 1. **Normal operation** (leader exists):
>    - Client sends request to leader
>    - Leader appends to its log, sends to all followers (hub → spokes)
>    - Once majority confirms (majority of spokes respond), leader commits
>    - Leader responds to client
> 
> 2. **Leader election** (leader crashed = hub disappeared):
>    - Followers have timeouts; if no heartbeat, become candidate
>    - Candidate votes for itself, asks others for votes (attempting to become new hub)
>    - Needs majority to win; includes randomized timeouts to avoid ties
>    - Winner becomes leader (new hub), starts sending heartbeats
> 
> 3. **Log replication** (keeping consistent):
>    - Each log entry has: term number (election cycle) + command + index
>    - Leader never overwrites its log, only appends
>    - If follower's log conflicts, leader forces follower to match
>    - **Graph property**: This ensures all paths from leader eventually carry identical information
> 
> **Why it's clearer than Paxos:**
> - Paxos: Any node can propose (fully connected graph behavior)
> - Raft: Only leader proposes (enforced star topology)
> - Paxos: Multiple roles can overlap (proposer/acceptor/learner)
> - Raft: Clean state machine, explicit transitions
> - Paxos: Must handle learning previous decisions via complex rules
> - Raft: Logs are append-only and always flow from leader → followers
> 
> **Safety guarantee**: If two logs contain an entry at the same index and term, they're identical up to that point. This comes from the election restriction: a candidate can only win if its log is at least as up-to-date as a majority.
> 
> **Quorum requirement**: Leader needs responses from majority of nodes. In graph terms, it needs to reach more than half the vertices. This ensures any two majorities (quorums) overlap by at least one node, preventing split-brain scenarios even during partitions.
> 
> **Quick pseudo-code:**
> ```python
> # Node state
> state = FOLLOWER  # or LEADER or CANDIDATE
> current_term = 0
> voted_for = None
> log = []
> commit_index = 0
> 
> # Follower timeout → become candidate
> def election_timeout():
>     current_term += 1
>     state = CANDIDATE
>     voted_for = self
>     votes = 1
>     
>     for node in other_nodes:
>         response = node.request_vote(current_term, len(log))
>         if response.vote_granted:
>             votes += 1
>     
>     if votes > majority:
>         state = LEADER
>         send_heartbeats()
> 
> # Leader appends entry
> def append_entry(command):
>     log.append(Entry(current_term, command))
>     
>     acks = 1
>     for node in other_nodes:
>         if node.append_entries(log):
>             acks += 1
>     
>     if acks > majority:
>         commit_index = len(log)
>         apply_to_state_machine(command)
> ```
> 
> In practice, Raft is used in etcd (Kubernetes), Consul, CockroachDB, and many other production systems. It's become the go-to consensus algorithm because developers can actually understand and correctly implement it.

> **What's Paxos?**
> 
> Paxos is Leslie Lamport's algorithm for getting multiple machines to agree on a single value, even when some machines crash or messages get lost.
> 
> The intuition: Imagine trying to pick a restaurant with friends via text, but some messages arrive late or not at all. Paxos solves this through rounds of proposals and voting:
> 
> 1. **Prepare phase**: "I propose restaurant X with proposal number N"
> 2. **Promise phase**: Others respond: "OK, I haven't seen a higher proposal number, I promise to reject anything lower"
> 3. **Accept phase**: If enough promises, send "Accept restaurant X"
> 4. **Accepted phase**: Once majority accepts, decision is final
> 
> The clever bit: Proposal numbers ensure that even if multiple people try to lead simultaneously (split brain), the system converges to one decision. Higher proposal numbers always win.
> 
> It's notoriously tricky to understand and implement correctly, which is why Raft was created - it does essentially the same thing but with clearer roles (leader/follower/candidate) and explicit state machines.
>
> **What algorithms does it use?**
> 
> It's actually pretty straightforward - just state machines and message passing! Nothing fancy like graph algorithms or dynamic programming. The core is:
> - Monotonically increasing proposal numbers (think: logical clocks, like Lamport timestamps)
> - Majority voting (quorum = $\lfloor n/2 \rfloor + 1$ nodes)
> - State tracking (remember highest proposal seen)
> 
> **Pseudo-code (simplified proposer side):**
> ```python
> # Proposer trying to get value v accepted
> def propose(v):
>     n = generate_unique_proposal_number()
>     
>     # Phase 1: Prepare
>     promises = []
>     for acceptor in majority_of_acceptors():
>         response = send(acceptor, PREPARE(n))
>         if response.type == PROMISE:
>             promises.append(response)
>     
>     if len(promises) < quorum:
>         return FAIL  # Try again with higher n
>     
>     # Check if any acceptor already accepted a value
>     highest_accepted = max(promises, key=lambda p: p.accepted_n)
>     if highest_accepted.value:
>         v = highest_accepted.value  # Must use this value!
>     
>     # Phase 2: Accept
>     accepts = []
>     for acceptor in majority_of_acceptors():
>         response = send(acceptor, ACCEPT(n, v))
>         if response.type == ACCEPTED:
>             accepts.append(response)
>     
>     if len(accepts) >= quorum:
>         return SUCCESS(v)
>     else:
>         return FAIL
> ```
> 
> **Acceptor side:**
> ```python
> # Acceptor state
> highest_promised_n = -1
> highest_accepted_n = -1
> accepted_value = None
> 
> def handle_prepare(n):
>     if n > highest_promised_n:
>         highest_promised_n = n
>         return PROMISE(highest_accepted_n, accepted_value)
>     else:
>         return REJECT
> 
> def handle_accept(n, v):
>     if n >= highest_promised_n:
>         highest_promised_n = n
>         highest_accepted_n = n
>         accepted_value = v
>         return ACCEPTED
>     else:
>         return REJECT
> ```
> 
> The tricky part isn't the algorithm itself - it's reasoning about all possible message orderings and failures!

### Byzantine Fault Tolerance (BFT)

> **What's BFT?**
> 
> BFT stands for **Byzantine Fault Tolerance** - it's the property of a system that can tolerate Byzantine faults and still reach consensus. A BFT algorithm guarantees that honest nodes will agree on a value even when some nodes are Byzantine.
> 
> **PBFT** (Practical Byzantine Fault Tolerance) is a specific BFT algorithm from 1999 that made BFT efficient enough for real systems. Before PBFT, BFT algorithms were too slow to be practical.
> 
> PBFT works through three phases (similar to Paxos but with extra rounds to catch Byzantine behavior):
> 1. **Pre-prepare**: Leader proposes a value with sequence number
> 2. **Prepare**: All nodes broadcast what they received (catches equivocation - if leader sent different values to different nodes, this reveals it)
> 3. **Commit**: Once nodes see matching prepares from $2f + 1$ nodes, they broadcast commit
> 4. **Execute**: Once $2f + 1$ matching commits seen, execute the operation
> 
> The extra communication rounds (compared to crash-tolerant consensus) are needed to detect and work around Byzantine nodes lying to different neighbors.

> **What's a Byzantine node?**
> 
> Yes, "Byzantine" is just the technical term for a node exhibiting arbitrary malicious or faulty behavior. The name comes from the "Byzantine Generals Problem" - imagine generals surrounding a city, needing to agree on whether to attack or retreat. Some generals might be traitors trying to cause confusion.
> 
> **Byzantine fault** = worst-case arbitrary behavior. A Byzantine node can:
> - **Lie arbitrarily**: Send "value = 5" to Alice, "value = 9" to Bob
> - **Crash at convenient times**: Fail right after spreading misinformation
> - **Equivocate**: Claim "I voted yes" to some nodes, "I voted no" to others
> - **Collude**: Coordinate with other Byzantine nodes to maximize damage
> - **Ignore protocol**: Send garbage, replay old messages, stay silent
> 
> Crucially, Byzantine nodes have **unlimited computational power** in the model - they can break any cryptographic assumption. The only thing they can't do is forge digital signatures (we assume signature schemes are secure).
> 
> **Contrast with simpler faults:**
> - **Crash fault**: Node just stops (think: power outage)
> - **Omission fault**: Node drops messages but otherwise follows protocol
> - **Byzantine fault**: Arbitrary malicious behavior (could be actual malice, software bugs, or hardware failures that cause random behavior)
> 
> Byzantine is the hardest fault model because you can't distinguish a Byzantine node from an honest one that's experiencing network issues - it's adversarially adaptive.

What if some nodes are Byzantine? Classical BFT (like PBFT) can tolerate up to $f$ Byzantine nodes out of $3f + 1$ total nodes.

> **Why $3f + 1$ nodes for $f$ Byzantine faults?**
> 
> **Quorum** = the minimum number of nodes that must agree for a decision to be valid. Think of it like a voting threshold - you need responses from at least this many nodes to proceed.
> 
> In most systems, quorum = majority = $\lfloor n/2 \rfloor + 1$ nodes. The key property: any two quorums must overlap (share at least one node).
> 
> **Why quorums matter**: If you make decisions based on quorum responses, and any two quorums overlap, then information can't be lost or inconsistent - at least one node saw both decisions.
> 
> Intuition for $3f + 1$: You need enough honest nodes that they outnumber Byzantine nodes in any quorum intersection.
> 
> - Quorum size: $2f + 1$ (majority of $3f + 1$)
> - Two quorums must overlap by at least: $(2f + 1) + (2f + 1) - (3f + 1) = f + 1$ nodes
> - If at most $f$ nodes are Byzantine, the intersection has at least 1 honest node
> - This honest node ensures both quorums saw consistent information
> 
> **Why not $2f + 1$ total nodes?**
> - With $2f + 1$ nodes, quorums are size $f + 1$
> - Two quorums could intersect in exactly $f + 1$ nodes
> - If all $f$ Byzantine nodes are in the intersection, you can't tell if the quorum is honest
> - Byzantine nodes could make one quorum think "decided A" and another think "decided B"
> 
> **Concrete example** with $f = 1$ (one traitor):
> - Need $3(1) + 1 = 4$ nodes total
> - Quorum = 3 nodes
> - Any two quorums overlap by at least 2 nodes
> - At most 1 Byzantine, so intersection has at least 1 honest node ✓
> 
> With only 3 nodes and $f=1$: quorums of size 2 could intersect in just 1 node, which might be the Byzantine one ✗

This is where blockchains come in.

**Graph view**: Byzantine nodes are **adversarial vertices** that can:
- Send different messages along different edges (lie to different neighbors)
- Collude with other Byzantine nodes (form adversarial subgraph)
- Try to partition the graph or create inconsistent views

The $3f + 1$ requirement means you need enough honest nodes that any two quorums (majorities) share at least $f + 1$ nodes, guaranteeing at least one honest node in the intersection. This prevents adversarial subgraphs from creating divergent views of the system state.

## How Blockchains Fit In

Blockchains are distributed systems designed for **trustless** environments - you can't assume nodes are honest.

**Graph perspective**: Blockchains operate on **dynamic, permissionless graphs**:
- Nodes can join/leave freely (vertices added/removed dynamically)
- You don't know the full graph topology (decentralized discovery)
- Adversarial nodes can have arbitrary connectivity (sybil attacks = fake vertices)
- Communication is eventually broadcast to all connected components

### Key Properties

1. **Distributed ledger**: Everyone maintains a copy of the transaction history
2. **Cryptographic linking**: Each block contains hash of previous block
   $h_i = \text{Hash}(h_{i-1} || \text{data}_i || \text{nonce}_i)$
   This creates a **directed acyclic graph (DAG)** of blocks, though typically pruned to a single chain
3. **Consensus without coordinator**: No fixed hub in the graph topology - any node can propose blocks

### Bitcoin's Innovation: Proof of Work

Instead of BFT's assumption of "less than 1/3 malicious nodes," Bitcoin uses computational cost:

- To propose a block, solve: $\text{Hash}(\text{block}) < \text{target}$
- This is expensive (requires $\sim 2^{256}/\text{target}$ attempts)
- Attacking requires 51% of total compute power

It's essentially converting the Byzantine problem into an economic game - attacking costs more than it's worth.

**Graph insight**: Instead of counting nodes (which can be faked via sybil attacks - creating many fake vertices), Bitcoin weights by computational power. The "real" graph is weighted by hash rate, not vertex count. An adversary must control >50% of the total edge weight (hash power) in the network to consistently win the longest chain race.

### The Blockchain Trilemma

Like CAP, blockchains face tradeoffs between:
- **Decentralization**: Many independent nodes
- **Security**: Resistance to attacks
- **Scalability**: Transaction throughput

Bitcoin chose decentralization + security, sacrificing scalability (~7 tx/sec).

### Modern Alternatives

- **Proof of Stake**: Replace computation with economic stake (Ethereum 2.0) - graph weighted by staked tokens instead of hash power
- **DAGs**: Directed acyclic graphs instead of chains (IOTA, Hedera) - embrace the natural DAG structure rather than forcing linearization
- **Sharding**: Partition the network to parallelize (Ethereum, Polkadot) - split the graph into subgraphs that process transactions in parallel
- **Layer 2**: Do most transactions off-chain, settle on-chain (Lightning Network) - create overlay networks (graphs on top of graphs) for fast local transactions

## The Connection

Blockchains are a specific type of distributed system that:
1. Optimize for **trustlessness** over performance
2. Use **incentive mechanisms** (economics) to enforce correct behavior
3. Prioritize **immutability** and **transparency**

Traditional distributed systems (like databases) assume a trusted environment and optimize for performance and consistency. Blockchains assume an adversarial environment and optimize for security and decentralization.

From a theoretical CS perspective, blockchains showed that you can achieve consensus in a permissionless setting - anyone can join without identity verification. Classical BFT protocols required knowing the set of participants ahead of time.

**Graph theory connection**: Most distributed systems assume a **known, static graph** (you know all nodes and their connections). Blockchains solve consensus on **unknown, dynamic graphs** where:
- Topology changes constantly (nodes join/leave)
- Adversary can spawn unlimited fake nodes (sybil attacks)
- You only see your local neighborhood, not global structure
- Network delays create temporarily inconsistent views of the graph state

This is why blockchains are so much slower - they're solving a fundamentally harder graph problem.

## Why Study Distributed Systems?

### The Mindset Shift

Most CS education teaches you to think about programs running on one machine with perfect execution. Distributed systems forces you to think fundamentally differently:

**From deterministic to probabilistic**: Your code doesn't "work" or "not work" - it works with some probability under some failure conditions. You start reasoning about liveness (eventually makes progress) vs. safety (never does the wrong thing).

**From correctness to tradeoffs**: There's no "correct" distributed system, only tradeoffs. You learn to articulate: "This design gives strong consistency but sacrifices 20% throughput" or "This approach is partition-tolerant but eventually consistent."

**From debugging to reasoning**: You can't step through a distributed system with a debugger when five machines are running simultaneously. You learn to reason about systems formally - invariants, state machines, message orderings. It's closer to mathematical proof than traditional debugging.

### What You Actually Gain

After a distributed systems course, you develop intuitions that other developers lack:

1. **Failure thinking**: You instinctively ask "what if this node crashes right here?" for any design. You see failure modes others miss.

2. **Scalability reasoning**: You can estimate: "This design does O(n²) messages per operation, won't scale past 100 nodes" or "This needs 3 round-trips, adds 300ms latency."

3. **Real-world architecture**: You understand how systems you use daily actually work - how does Google Spanner provide global consistency? Why does DynamoDB choose eventual consistency? How does Kafka guarantee ordering?

4. **The "distributed systems smell"**: You recognize when someone's design won't work at scale. You've seen enough consensus algorithms to know when someone's "simple new protocol" is reinventing Paxos badly.

### Career Impact

This knowledge puts you in a different tier for certain roles:

- **Infrastructure engineering**: You can actually design the systems (databases, message queues, storage systems) rather than just use them
- **Big tech interviews**: Companies like Google, Amazon, Meta heavily weight distributed systems in senior interviews
- **Startup CTO/architect**: You can make informed decisions about CAP tradeoffs, not just cargo-cult "use MongoDB because it scales"
- **Research**: If you go into systems research, this is foundational - ML systems, databases, cloud computing all build on these primitives

### The Humility Factor

Paradoxically, studying distributed systems makes you more cautious. You've seen so many subtle bugs and edge cases that you become skeptical of "simple" solutions to hard problems. When someone proposes a new blockchain or database, you immediately think: "Did they handle split-brain? What about the two-generals problem?"

It's the difference between a developer who thinks "I'll just add a cache" and one who thinks "I'll add a cache, but invalidation is hard - what consistency model do I need? What if the cache and DB disagree? What's my failure mode?"

## Further Reading

- **Lamport's "Time, Clocks, and the Ordering of Events"** - foundational distributed systems paper
- **Bitcoin whitepaper** - surprisingly readable, only 9 pages
- **Raft paper** - "In Search of an Understandable Consensus Algorithm"
- **Designing Data-Intensive Applications** (Martin Kleppmann) - the best book for practitioners