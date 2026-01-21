# 06 — Consistency & Consensus

> Understanding the trade-offs between consistency, availability, and partition tolerance is central to distributed systems design.

**Prerequisites:** [01 — Foundational Concepts](./01-FOUNDATIONAL-CONCEPTS.md), [05 — Distributed Patterns](./05-DISTRIBUTED-PATTERNS.md)  
**Builds toward:** [08 — Messaging & Async](./08-MESSAGING-ASYNC.md)  
**Estimated study time:** 3-4 hours

---

## Chapter Overview

This module provides a deep dive into consistency models, the CAP and PACELC theorems, replication strategies, and how real systems make these trade-offs.

```mermaid
graph TD
    subgraph "Theorems"
        CAP[CAP Theorem]
        PACELC[PACELC Extension]
    end
    
    subgraph "Consistency Models"
        Strong[Strong Consistency]
        Eventual[Eventual Consistency]
        Causal[Causal Consistency]
    end
    
    subgraph "Replication"
        SL[Single Leader]
        ML[Multi-Leader]
        LL[Leaderless]
    end
    
    CAP --> PACELC
    PACELC --> Strong
    PACELC --> Eventual
    Strong --> SL
    Eventual --> ML
    Eventual --> LL
```

---

## 1. The CAP Theorem — Deep Dive

### The Three Properties

| Property | Definition | Practical Meaning |
|----------|------------|-------------------|
| **Consistency (C)** | All nodes see the same data at the same time | After a write completes, all reads return that value |
| **Availability (A)** | Every request receives a response | System responds even if some nodes are down |
| **Partition Tolerance (P)** | System continues despite network failures | Handles network splits between nodes |

### The Fundamental Trade-off

```mermaid
graph TD
    subgraph "CAP Triangle"
        C[Consistency]
        A[Availability]
        P[Partition Tolerance]
        
        C --- A
        A --- P
        P --- C
    end
    
    CP["CP: Consistency + Partition Tolerance<br/>Sacrifice availability during partition"]
    AP["AP: Availability + Partition Tolerance<br/>Sacrifice consistency during partition"]
    CA["CA: Not practical in distributed systems<br/>Requires no partitions ever"]
    
    CP -.-> C
    CP -.-> P
    AP -.-> A
    AP -.-> P
```

**Key insight:** Network partitions are inevitable. The real choice is: during a partition, do you sacrifice consistency (serve potentially stale data) or availability (refuse to respond)?

### CAP in Practice: A Scenario

```mermaid
sequenceDiagram
    participant C as Client
    participant N1 as Node 1 (Leader)
    participant N2 as Node 2 (Replica)
    
    Note over N1,N2: Normal operation
    C->>N1: Write X=1
    N1->>N2: Replicate X=1
    N2->>N1: Ack
    N1->>C: Success
    
    Note over N1,N2: Network partition occurs
    C->>N2: Read X
    
    alt CP System
        N2->>C: Error: Cannot verify consistency
    else AP System
        N2->>C: X=1 (possibly stale)
    end
```

### CP Systems

**Behavior:** During partition, refuse requests that can't be consistently served.

| System | Behavior During Partition |
|--------|--------------------------|
| **HBase** | Unavailable for affected regions |
| **MongoDB (default)** | Writes fail if can't reach majority |
| **Redis Cluster** | Rejects writes to minority partition |
| **Spanner** | Waits for synchronous replication |

**Use when:**
- Data correctness is paramount
- Financial transactions
- Inventory management (no overselling)

### AP Systems

**Behavior:** During partition, serve requests but accept potential inconsistency.

| System | Behavior During Partition |
|--------|--------------------------|
| **Cassandra** | Accepts writes to available nodes |
| **DynamoDB** | Continues with available replicas |
| **CouchDB** | Allows writes, resolves conflicts later |
| **Riak** | Uses vector clocks for conflict detection |

**Use when:**
- Availability is critical
- Some staleness is acceptable
- Social media feeds, shopping carts (can reconcile later)

### CAP Misconceptions

| Misconception | Reality |
|---------------|---------|
| "Pick two of three" | Partition tolerance isn't optional; it's about behavior *during* partitions |
| "Always one or the other" | Most systems are tunable; behavior depends on configuration |
| "Applies all the time" | CAP trade-off only matters during partitions |
| "Binary choice" | Spectrum of consistency levels exists |

---

## 2. PACELC Theorem

### Beyond CAP

CAP only addresses behavior during partitions. What about normal operation?

**PACELC states:**

```
if (Partition) {
    trade-off between Availability and Consistency
} else {
    trade-off between Latency and Consistency
}
```

### The Complete Picture

```mermaid
graph TD
    subgraph "PACELC"
        P[Partition?]
        P -->|Yes| PAC[A vs C trade-off]
        P -->|No| ELC[L vs C trade-off]
        
        PAC -->|Favor A| PA[High Availability]
        PAC -->|Favor C| PC[Strong Consistency]
        
        ELC -->|Favor L| EL[Low Latency]
        ELC -->|Favor C| EC[Strong Consistency]
    end
```

### System Classifications

| System | During Partition | Else (Normal) | Classification |
|--------|------------------|---------------|----------------|
| **Cassandra** | Availability | Latency | PA/EL |
| **DynamoDB** | Availability | Latency | PA/EL |
| **HBase** | Consistency | Consistency | PC/EC |
| **BigTable** | Consistency | Consistency | PC/EC |
| **MongoDB** | Availability | Consistency | PA/EC |
| **PNUTS** | Consistency | Latency | PC/EL |

### PA/EL Systems (Cassandra, DynamoDB)

- Optimize for availability and low latency
- Accept eventual consistency
- Best for: high-throughput, global distribution

### PC/EC Systems (HBase, Spanner)

- Prioritize consistency always
- Accept higher latency
- Best for: financial data, inventory

### PA/EC Systems (MongoDB default)

- Available during partitions
- Consistent during normal operation
- Best for: general-purpose with some consistency needs

---

## 3. Consistency Models

### The Consistency Spectrum

```
Strong ←——————————————————————————————————→ Weak
Linearizable → Sequential → Causal → Eventual
```

### Linearizability (Strongest)

Operations appear to happen atomically at some point between invocation and completion.

```mermaid
sequenceDiagram
    participant C1 as Client 1
    participant C2 as Client 2
    participant S as System
    
    C1->>S: Write X=1
    Note over S: Linearization point
    S->>C1: Ack
    C2->>S: Read X
    S->>C2: X=1 (guaranteed)
```

**Guarantee:** If write completes before read starts, read sees the write.

### Sequential Consistency

Operations appear in some sequential order consistent with program order of each client.

**Difference from linearizable:** Order doesn't have to match real-time order, just per-client order.

### Causal Consistency

Operations that are causally related appear in the same order to all nodes.

```mermaid
sequenceDiagram
    participant A as Alice
    participant B as Bob
    participant S as System
    
    A->>S: Write: "Hello"
    B->>S: Read: "Hello"
    B->>S: Write: "Reply to Hello"
    
    Note over S: "Reply" must appear after "Hello"<br/>on all nodes (causally related)
```

**Use case:** Social media comments must appear after their parent posts.

### Eventual Consistency

If no new updates, all replicas will eventually converge to the same value.

```mermaid
sequenceDiagram
    participant C as Client
    participant N1 as Node 1
    participant N2 as Node 2
    
    C->>N1: Write X=1
    N1->>C: Ack
    
    Note over N1,N2: Async replication
    
    C->>N2: Read X
    N2->>C: X=? (might be old)
    
    Note over N1,N2: Eventually...
    N1-->>N2: Replicate X=1
    
    C->>N2: Read X
    N2->>C: X=1 (consistent now)
```

### Consistency Model Comparison

| Model | Guarantees | Performance | Use Cases |
|-------|------------|-------------|-----------|
| **Linearizable** | Real-time ordering | Slowest | Distributed locks, leader election |
| **Sequential** | Per-client ordering | Fast | Many applications |
| **Causal** | Cause-effect ordering | Fast | Social media, collaboration |
| **Eventual** | Eventual convergence | Fastest | Caching, DNS, analytics |

---

## 4. Replication Strategies

### Single-Leader Replication

One node (leader) accepts all writes; followers replicate from leader.

```mermaid
graph TD
    C[Clients] --> L[Leader]
    L -->|Replicate| F1[Follower 1]
    L -->|Replicate| F2[Follower 2]
    L -->|Replicate| F3[Follower 3]
    
    R[Read Clients] --> F1
    R --> F2
    R --> F3
```

| Aspect | Description |
|--------|-------------|
| **Consistency** | Strong (reads from leader) or eventual (reads from followers) |
| **Write throughput** | Limited by single leader |
| **Failure handling** | Leader failure requires failover |

### Multi-Leader Replication

Multiple nodes accept writes; changes replicated to all leaders.

```mermaid
graph TD
    subgraph "Datacenter A"
        LA[Leader A]
    end
    
    subgraph "Datacenter B"
        LB[Leader B]
    end
    
    subgraph "Datacenter C"
        LC[Leader C]
    end
    
    LA <-->|Async replicate| LB
    LB <-->|Async replicate| LC
    LC <-->|Async replicate| LA
    
    CA[Clients A] --> LA
    CB[Clients B] --> LB
    CC[Clients C] --> LC
```

| Aspect | Description |
|--------|-------------|
| **Consistency** | Eventual (conflicts possible) |
| **Write throughput** | Higher (parallel writes) |
| **Failure handling** | Other leaders continue |

**Challenge:** Write conflicts must be resolved.

### Leaderless Replication

No single leader; clients write to multiple nodes directly.

```mermaid
graph TD
    C[Client] --> N1[Node 1]
    C --> N2[Node 2]
    C --> N3[Node 3]
    
    N1 <--> N2
    N2 <--> N3
    N3 <--> N1
```

| Aspect | Description |
|--------|-------------|
| **Consistency** | Tunable via quorums |
| **Write throughput** | High (parallel to multiple nodes) |
| **Failure handling** | Automatic (no leader to fail) |

### Replication Topology Comparison

| Topology | Conflict Handling | Latency | Complexity | Use Case |
|----------|-------------------|---------|------------|----------|
| **Single-Leader** | None (serialized) | Higher for remote writes | Low | Most OLTP |
| **Multi-Leader** | Required | Lower (local writes) | High | Multi-DC |
| **Leaderless** | Required | Tunable | Medium | High availability |

---

## 5. Conflict Resolution

### Why Conflicts Occur

In multi-leader or leaderless systems, concurrent writes to the same data create conflicts.

```mermaid
sequenceDiagram
    participant C1 as Client 1
    participant L1 as Leader 1
    participant L2 as Leader 2
    participant C2 as Client 2
    
    C1->>L1: X = "A"
    C2->>L2: X = "B"
    
    Note over L1,L2: Both think they succeeded
    
    L1-->>L2: Replicate X="A"
    L2-->>L1: Replicate X="B"
    
    Note over L1,L2: Conflict: X="A" or X="B"?
```

### Conflict Resolution Strategies

| Strategy | Description | Pros | Cons |
|----------|-------------|------|------|
| **Last-Write-Wins (LWW)** | Highest timestamp wins | Simple | Data loss, clock skew issues |
| **Merge** | Combine conflicting values | No data loss | May not make sense |
| **Custom resolution** | Application-specific logic | Correct semantics | Complex |
| **Keep all versions** | Store both, let user resolve | No data loss | Requires user intervention |

### Last-Write-Wins (LWW)

```mermaid
graph TD
    W1["Write X='A' @ t=100"]
    W2["Write X='B' @ t=105"]
    R["Resolve: X='B' (higher timestamp)"]
    
    W1 --> R
    W2 --> R
```

**Problem:** If clocks are skewed, "last" might not be real last.

### Vector Clocks

Track causality to detect true conflicts.

```mermaid
graph TD
    subgraph "Vector Clock Example"
        S1["Server 1: [1,0,0]"]
        S2["Server 2: [0,1,0]"]
        S3["Server 3: [0,0,1]"]
    end
    
    W1["Write @ [2,0,0]"]
    W2["Write @ [0,2,0]"]
    
    Compare["[2,0,0] vs [0,2,0]<br/>Neither dominates → Conflict!"]
```

**Rule:** If all components of V1 ≤ V2, then V1 happened-before V2. Otherwise, concurrent (conflict).

### CRDTs (Conflict-free Replicated Data Types)

Data structures that automatically merge without conflicts.

| CRDT Type | Example | Merge Behavior |
|-----------|---------|----------------|
| **G-Counter** | Page views | Sum of all increments |
| **PN-Counter** | Likes/unlikes | Positive + Negative counters |
| **G-Set** | Tags | Union of sets |
| **LWW-Register** | User profile | Last-write-wins value |

```mermaid
graph TD
    subgraph "G-Counter CRDT"
        N1["Node 1: count=5"]
        N2["Node 2: count=3"]
        N3["Node 3: count=7"]
        
        Merge["Global count = 5+3+7 = 15"]
    end
    
    N1 --> Merge
    N2 --> Merge
    N3 --> Merge
```

---

## 6. Read-Your-Writes Consistency

### The Problem

After writing, user reads from a replica that hasn't received the write yet.

```mermaid
sequenceDiagram
    participant U as User
    participant L as Leader
    participant F as Follower
    
    U->>L: Write: name = "Alice"
    L->>U: Success
    L-->>F: Replicate (async, delayed)
    U->>F: Read name
    F->>U: name = "Bob" (old value!)
```

### Solutions

| Solution | Description | Trade-off |
|----------|-------------|-----------|
| **Read from leader** | After write, read from leader | Leader load |
| **Sticky sessions** | Route user to same replica | Load balancing complexity |
| **Version tracking** | Client tracks write version | Client complexity |
| **Synchronous replication** | Wait for replication | Latency |

### Monotonic Reads

Guarantee: Once you read a value, you won't see an older value.

```mermaid
sequenceDiagram
    participant U as User
    participant F1 as Follower 1
    participant F2 as Follower 2
    
    U->>F1: Read X = 5
    U->>F2: Read X = ?
    
    Note over F2: Must return X ≥ 5<br/>(not older value)
```

**Implementation:** Track read version, route to replicas at or ahead of that version.

---

## 7. Consensus Algorithms

### The Consensus Problem

Multiple nodes must agree on a single value, even if some nodes fail.

**Requirements:**
- **Agreement:** All non-faulty nodes decide the same value
- **Validity:** Decided value was proposed by some node
- **Termination:** All non-faulty nodes eventually decide

### Paxos

Classic consensus algorithm (complex to understand).

**Roles:**
- **Proposers:** Propose values
- **Acceptors:** Vote on proposals
- **Learners:** Learn decided value

### Raft

Understandable consensus algorithm.

```mermaid
stateDiagram-v2
    [*] --> Follower
    Follower --> Candidate: Election timeout
    Candidate --> Leader: Wins election<br/>(majority votes)
    Candidate --> Follower: Discovers higher term
    Leader --> Follower: Discovers higher term
```

**Key mechanisms:**
1. **Leader election:** Candidates request votes, majority wins
2. **Log replication:** Leader appends entries, replicates to followers
3. **Safety:** Only leaders with up-to-date logs can be elected

### Consensus in Practice

| Use Case | Why Consensus |
|----------|---------------|
| **Leader election** | Agree on who is leader |
| **Distributed locks** | Agree on lock holder |
| **Configuration** | Agree on cluster membership |
| **Total ordering** | Agree on operation order |

---

## 8. Chapter Summary

### Key Concepts

| Concept | One-Line Definition |
|---------|---------------------|
| **CAP** | During partitions, choose consistency or availability |
| **PACELC** | Even without partitions, trade latency for consistency |
| **Linearizability** | Operations appear atomic, real-time ordered |
| **Eventual consistency** | All replicas converge given enough time |
| **Quorum** | Majority agreement for consistency |
| **Vector clock** | Logical time to detect concurrent writes |
| **CRDT** | Data type that merges without conflicts |

### Design Decision Framework

```mermaid
flowchart TD
    A[Consistency Requirements?]
    
    A -->|Must be correct| B[CP System]
    A -->|Can tolerate stale| C[AP System]
    A -->|Depends on operation| D[Tunable Consistency]
    
    B --> B1[Single-leader sync replication]
    B --> B2[Consensus protocol]
    
    C --> C1[Multi-leader or leaderless]
    C --> C2[Eventual consistency + conflict resolution]
    
    D --> D1[Quorum tuning W, R, N]
    D --> D2[Per-operation consistency level]
```

### Interview Articulation Patterns

> "Explain the CAP theorem."

"CAP states that during a network partition, a distributed system must choose between consistency (all nodes see the same data) and availability (all requests get responses). Since partitions are inevitable, the practical choice is whether to fail requests or serve potentially stale data during partitions."

> "When would you choose eventual consistency?"

"When availability and latency are more important than immediate consistency. For example, a social media feed can show slightly stale data—users tolerate seeing a post a few seconds late. But a banking system transferring money needs strong consistency to prevent double-spending."

> "How do you handle conflicts in a multi-leader setup?"

"Options include last-write-wins (simple but can lose data), vector clocks (detect conflicts, let application resolve), or CRDTs (data structures that automatically merge). The choice depends on whether data loss is acceptable and whether meaningful automatic merging is possible."

---

## Navigation

**Previous:** [05 — Distributed Patterns](./05-DISTRIBUTED-PATTERNS.md)  
**Next:** [07 — Load Balancing & Scaling](./07-LOAD-BALANCING-SCALING.md)  
**Index:** [00 — Handbook Index](./00-INDEX.md)
