# 05 — Distributed System Patterns

> Recurring solutions to common problems in distributed systems.

**Prerequisites:** [01 — Foundational Concepts](./01-FOUNDATIONAL-CONCEPTS.md), [03 — Data Management](./03-DATA-MANAGEMENT.md)  
**Builds toward:** [06 — Consistency & Consensus](./06-CONSISTENCY-CONSENSUS.md)  
**Estimated study time:** 3-4 hours

---

## Chapter Overview

This module covers fundamental patterns that appear repeatedly in distributed systems: how nodes find each other, agree on leaders, detect failures, and distribute data.

```mermaid
graph TD
    subgraph "Coordination"
        LE[Leader Election]
        Q[Quorum]
    end
    
    subgraph "Failure Detection"
        HB[Heartbeat]
        CS[Checksum]
    end
    
    subgraph "Data Distribution"
        CH[Consistent Hashing]
        G[Gossip Protocol]
    end
    
    LE --> Q
    HB --> LE
    CS --> HB
    CH --> G
```

---

## 1. Consistent Hashing

### The Problem

Standard hash-based distribution breaks when nodes are added or removed.

**Naive approach:** `server = hash(key) % N`

```
N = 3 servers
hash("user:123") % 3 = 1 → Server 1
hash("user:456") % 3 = 2 → Server 2

# Add a 4th server (N = 4)
hash("user:123") % 4 = 1 → Server 1 (same)
hash("user:456") % 4 = 0 → Server 0 (MOVED!)
```

**Problem:** When N changes, most keys remap to different servers → massive data movement.

### Consistent Hashing Solution

Arrange hash space as a ring. Both servers and keys are hashed onto this ring.

```mermaid
graph TD
    subgraph "Hash Ring"
        direction TB
        A["0°"]
        B["90°"]
        C["180°"]
        D["270°"]
    end
    
    S1["Server A @ 45°"]
    S2["Server B @ 150°"]
    S3["Server C @ 280°"]
    
    K1["Key1 @ 30° → A"]
    K2["Key2 @ 100° → B"]
    K3["Key3 @ 200° → C"]
```

**Rule:** Key is assigned to the first server encountered moving clockwise from the key's position.

### Adding/Removing Nodes

```mermaid
graph LR
    subgraph "Before: 3 Servers"
        R1["A handles: 270°-45°"]
        R2["B handles: 45°-150°"]
        R3["C handles: 150°-270°"]
    end
    
    subgraph "After: Add D @ 100°"
        R4["A handles: 270°-45°"]
        R5["B handles: 100°-150°"]
        R6["C handles: 150°-270°"]
        R7["D handles: 45°-100°"]
    end
```

**Key insight:** Only keys between the new node and its predecessor move. All other keys stay put.

| Operation | Keys Moved | Naive Hashing |
|-----------|------------|---------------|
| Add 1 node to N | ~1/N of keys | ~(N-1)/N of keys |
| Remove 1 node from N | ~1/N of keys | ~(N-1)/N of keys |

### Virtual Nodes (Vnodes)

**Problem:** Physical nodes may be unevenly distributed on the ring, causing load imbalance.

**Solution:** Each physical node owns multiple virtual nodes spread across the ring.

```mermaid
graph TD
    subgraph "Hash Ring with Virtual Nodes"
        V1["A-1"]
        V2["B-1"]
        V3["A-2"]
        V4["C-1"]
        V5["B-2"]
        V6["A-3"]
        V7["C-2"]
        V8["B-3"]
    end
    
    PA["Physical A → A-1, A-2, A-3"]
    PB["Physical B → B-1, B-2, B-3"]
    PC["Physical C → C-1, C-2, C-3"]
```

**Benefits:**
- More even key distribution
- Heterogeneous nodes: powerful nodes get more vnodes
- Smoother rebalancing when nodes join/leave

### Consistent Hashing in Practice

| System | Usage |
|--------|-------|
| **Cassandra** | Partitioning data across nodes |
| **DynamoDB** | Key distribution |
| **Memcached** | Client-side distribution |
| **Nginx** | Upstream server selection |

---

## 2. Leader Election

### Why Leaders?

Distributed systems often need a single coordinator to:
- Serialize writes to avoid conflicts
- Coordinate distributed transactions
- Make authoritative decisions

### The Challenge

How do N nodes agree on exactly one leader without central authority?

**Requirements:**
- **Safety:** At most one leader at any time
- **Liveness:** Eventually elect a leader
- **Fault tolerance:** Survive node/network failures

### Bully Algorithm

Simple algorithm where highest-ranked alive node becomes leader.

```mermaid
sequenceDiagram
    participant N1 as Node 1
    participant N2 as Node 2
    participant N3 as Node 3 (Highest)
    
    Note over N1,N3: N3 was leader, crashes
    N1->>N2: Election (I want to be leader)
    N1->>N3: Election
    N2->>N1: OK (I'm higher, back off)
    Note over N3: No response (dead)
    N2->>N3: Election
    Note over N3: No response
    N2->>N1: Coordinator (I'm the new leader)
    N2->>N3: Coordinator
```

**Pros:** Simple to understand  
**Cons:** Not partition tolerant, high message complexity

### Raft Consensus

Modern, understandable consensus algorithm. Nodes in three states: Follower, Candidate, Leader.

```mermaid
stateDiagram-v2
    [*] --> Follower
    Follower --> Candidate: Election timeout
    Candidate --> Leader: Wins election
    Candidate --> Follower: Discovers leader
    Candidate --> Candidate: Split vote, new election
    Leader --> Follower: Discovers higher term
```

**Election process:**

```mermaid
sequenceDiagram
    participant F1 as Follower 1
    participant C as Candidate
    participant F2 as Follower 2
    
    Note over C: Election timeout, become candidate
    C->>C: Increment term, vote for self
    C->>F1: RequestVote(term=2)
    C->>F2: RequestVote(term=2)
    F1->>C: VoteGranted
    F2->>C: VoteGranted
    Note over C: Majority votes → become Leader
    C->>F1: Heartbeat (I'm leader)
    C->>F2: Heartbeat (I'm leader)
```

**Key mechanisms:**
- **Terms:** Logical time, incremented on elections
- **Heartbeats:** Leader sends periodic heartbeats to maintain authority
- **Randomized timeouts:** Prevent split votes

### Leader Election via External Service

Use ZooKeeper, etcd, or Consul for leader election:

```mermaid
sequenceDiagram
    participant N1 as Node 1
    participant N2 as Node 2
    participant ZK as ZooKeeper
    
    N1->>ZK: Create /leader (ephemeral)
    ZK->>N1: Success (you're leader)
    N2->>ZK: Create /leader (ephemeral)
    ZK->>N2: Fail (already exists)
    N2->>ZK: Watch /leader
    Note over N1: Crashes
    ZK->>ZK: Ephemeral node deleted
    ZK->>N2: Watch triggered
    N2->>ZK: Create /leader (ephemeral)
    ZK->>N2: Success (you're leader)
```

**Pros:** Battle-tested, handles edge cases  
**Cons:** Additional infrastructure dependency

---

## 3. Quorum

### The Concept

A quorum is the minimum number of nodes that must participate in an operation for it to be valid.

**Purpose:** Ensure any two operations have overlapping participants, guaranteeing they see each other's effects.

### Read/Write Quorums

For a system with N replicas:
- **W:** Write quorum (nodes that must acknowledge a write)
- **R:** Read quorum (nodes that must respond to a read)

**Consistency guarantee:** If W + R > N, reads and writes overlap → strong consistency possible

```mermaid
graph TD
    subgraph "N=5 Replicas"
        N1[Node 1]
        N2[Node 2]
        N3[Node 3]
        N4[Node 4]
        N5[Node 5]
    end
    
    W["Write to W=3"]
    R["Read from R=3"]
    
    W --> N1
    W --> N2
    W --> N3
    
    R --> N3
    R --> N4
    R --> N5
    
    Note["N3 participates in both → overlap guaranteed"]
```

### Quorum Configurations

| W | R | Constraint | Trade-off |
|---|---|------------|-----------|
| N | 1 | W + R > N ✓ | Fast reads, slow writes |
| 1 | N | W + R > N ✓ | Fast writes, slow reads |
| (N+1)/2 | (N+1)/2 | W + R > N ✓ | Balanced (majority quorum) |

### Quorum Math

For N replicas:
- **Majority quorum:** ⌊N/2⌋ + 1
- **Tolerate F failures:** Need N ≥ 2F + 1 nodes

| Nodes (N) | Majority | Failures Tolerated |
|-----------|----------|-------------------|
| 3 | 2 | 1 |
| 5 | 3 | 2 |
| 7 | 4 | 3 |

### Sloppy Quorum

When strict quorum nodes unavailable, use any available nodes temporarily.

```mermaid
graph TD
    subgraph "Normal Quorum"
        A[Preferred Node A]
        B[Preferred Node B]
        C[Preferred Node C]
    end
    
    subgraph "Sloppy Quorum (A unavailable)"
        B2[Preferred Node B]
        C2[Preferred Node C]
        D[Substitute Node D]
    end
```

**Hinted handoff:** When A recovers, D sends data to A and deletes its copy.

**Trade-off:** Higher availability, weaker consistency guarantee.

---

## 4. Heartbeat and Failure Detection

### The Problem

In distributed systems, we can't distinguish between:
- Node crashed
- Network partitioned
- Node slow (high load)

### Heartbeat Pattern

Nodes periodically send "I'm alive" messages.

```mermaid
sequenceDiagram
    participant M as Monitor
    participant N1 as Node 1
    participant N2 as Node 2
    
    loop Every T seconds
        N1->>M: Heartbeat
        N2->>M: Heartbeat
    end
    
    Note over N1: Crashes
    Note over M: No heartbeat from N1 for 3T
    M->>M: Mark N1 as failed
```

### Heartbeat Topologies

| Topology | Description | Pros | Cons |
|----------|-------------|------|------|
| **Centralized** | All nodes → central monitor | Simple | Monitor is SPOF |
| **Ring** | Each node monitors next | No SPOF | Slow detection |
| **All-to-all** | Every node monitors every other | Fast, robust | O(N²) messages |
| **Gossip** | Random subset monitoring | Scalable | Probabilistic |

### Failure Detection Parameters

| Parameter | Effect of Increase |
|-----------|-------------------|
| **Heartbeat interval** | Less network traffic, slower detection |
| **Timeout threshold** | Fewer false positives, slower detection |
| **Retry count** | More accurate, slower detection |

**Trade-off:** Speed of detection vs. false positive rate

### Phi Accrual Failure Detector

Instead of binary (alive/dead), output a suspicion level (φ).

```
φ = 0-1: Probably alive
φ = 1-2: Suspicious
φ > 2: Probably dead
```

**Adaptive:** Adjusts thresholds based on observed heartbeat arrival times.

---

## 5. Checksum and Data Integrity

### The Problem

Data can be corrupted during:
- Storage (disk errors)
- Transmission (network errors)
- Processing (software bugs)

### Checksum Basics

A checksum is a fixed-size digest computed from data. If data changes, checksum changes.

```mermaid
graph LR
    D[Data] --> H[Hash Function]
    H --> C[Checksum]
    
    D2[Data + Checksum] --> V{Verify}
    V -->|Match| OK[Data intact]
    V -->|Mismatch| ERR[Data corrupted]
```

### Common Checksum Algorithms

| Algorithm | Output Size | Speed | Collision Resistance |
|-----------|-------------|-------|---------------------|
| CRC32 | 32 bits | Very fast | Low |
| MD5 | 128 bits | Fast | Broken (crypto) |
| SHA-1 | 160 bits | Fast | Weak (crypto) |
| SHA-256 | 256 bits | Moderate | Strong |
| xxHash | 64/128 bits | Very fast | Non-cryptographic |

**Guidelines:**
- **Data integrity:** CRC32 or xxHash (fast, good error detection)
- **Security/deduplication:** SHA-256 (collision resistant)

### Checksum in Distributed Systems

```mermaid
sequenceDiagram
    participant W as Writer
    participant S as Storage
    participant R as Reader
    
    W->>W: Compute checksum(data)
    W->>S: Store(data, checksum)
    
    R->>S: Read()
    S->>R: data, checksum
    R->>R: Verify checksum(data) == checksum
    alt Match
        R->>R: Use data
    else Mismatch
        R->>S: Request from replica
    end
```

### Merkle Trees

Hierarchical checksums for efficient verification of large datasets.

```mermaid
graph TD
    Root["Root Hash<br/>H(H12 + H34)"]
    
    H12["H12<br/>H(H1 + H2)"]
    H34["H34<br/>H(H3 + H4)"]
    
    H1["H1<br/>H(Block 1)"]
    H2["H2<br/>H(Block 2)"]
    H3["H3<br/>H(Block 3)"]
    H4["H4<br/>H(Block 4)"]
    
    Root --> H12
    Root --> H34
    H12 --> H1
    H12 --> H2
    H34 --> H3
    H34 --> H4
```

**Benefits:**
- Compare large datasets by comparing root hashes
- Identify differing blocks in O(log N) comparisons
- Used in: Git, BitTorrent, Cassandra anti-entropy

---

## 6. Gossip Protocol

### The Concept

Nodes periodically exchange information with random peers, spreading updates like rumors.

```mermaid
sequenceDiagram
    participant A
    participant B
    participant C
    participant D
    
    Note over A: Has update X
    A->>B: Gossip: {X}
    Note over B: Learns X
    B->>D: Gossip: {X}
    A->>C: Gossip: {X}
    Note over C,D: All nodes have X
```

### Gossip Properties

| Property | Description |
|----------|-------------|
| **Scalable** | O(log N) rounds to reach all nodes |
| **Fault tolerant** | Works despite node failures |
| **Decentralized** | No single point of failure |
| **Eventually consistent** | All nodes converge to same state |

### Gossip Protocol Types

| Type | What's Shared | Use Case |
|------|---------------|----------|
| **Anti-entropy** | Full state comparison | Replica synchronization |
| **Rumor mongering** | Recent updates only | Event dissemination |
| **Aggregation** | Computed values (count, sum) | Cluster monitoring |

### Gossip in Practice

| System | Gossip Usage |
|--------|--------------|
| **Cassandra** | Cluster membership, failure detection |
| **Consul** | Service discovery, health checking |
| **Amazon S3** | Replica synchronization |
| **Bitcoin** | Transaction propagation |

---

## 7. The Leader-Follower Pattern

### Architecture

One leader handles writes; followers replicate and serve reads.

```mermaid
graph TD
    C[Client Writes] --> L[Leader]
    L --> F1[Follower 1]
    L --> F2[Follower 2]
    L --> F3[Follower 3]
    
    R[Client Reads] --> F1
    R --> F2
    R --> F3
```

### Replication Modes

| Mode | Behavior | Consistency | Availability |
|------|----------|-------------|--------------|
| **Synchronous** | Wait for all followers | Strong | Lower |
| **Semi-synchronous** | Wait for one follower | Strong | Medium |
| **Asynchronous** | Don't wait | Eventual | Higher |

### Failover Process

```mermaid
sequenceDiagram
    participant L as Leader
    participant F1 as Follower 1
    participant F2 as Follower 2
    participant M as Monitor
    
    Note over L: Leader fails
    M->>M: Detect leader failure
    M->>F1: Check replication lag
    M->>F2: Check replication lag
    Note over M: F1 has least lag
    M->>F1: Promote to leader
    F1->>F1: Become leader
    M->>F2: New leader is F1
    F2->>F1: Start replicating from F1
```

### Challenges

| Challenge | Description | Mitigation |
|-----------|-------------|------------|
| **Split brain** | Two nodes think they're leader | Fencing, quorum |
| **Replication lag** | Followers behind leader | Sync replication, read-your-writes |
| **Failover data loss** | Async writes not replicated | Semi-sync, accept loss |

---

## 8. Chapter Summary

### Key Patterns

| Pattern | Purpose | Key Trade-off |
|---------|---------|---------------|
| **Consistent Hashing** | Distribute data, minimize movement | Complexity vs stability |
| **Leader Election** | Coordinate distributed operations | Consistency vs availability |
| **Quorum** | Ensure overlap between operations | Consistency vs latency |
| **Heartbeat** | Detect node failures | Detection speed vs false positives |
| **Checksum** | Verify data integrity | CPU cost vs corruption detection |
| **Gossip** | Disseminate information | Consistency vs scalability |

### Pattern Selection Guide

```mermaid
flowchart TD
    A[Problem Type?]
    
    A -->|Data distribution| B[Consistent Hashing]
    A -->|Need coordinator| C[Leader Election]
    A -->|Ensure consistency| D[Quorum]
    A -->|Monitor health| E[Heartbeat]
    A -->|Verify integrity| F[Checksum]
    A -->|Spread information| G[Gossip]
```

### Interview Articulation Patterns

> "How does consistent hashing help with scaling?"

"Consistent hashing maps both keys and servers to a ring. When servers are added or removed, only keys adjacent to the change point need to move—roughly K/N keys instead of most keys. Virtual nodes further improve load distribution."

> "How do you detect failures in a distributed system?"

"Typically through heartbeats with timeouts. The trade-off is between detection speed and false positives. Shorter timeouts detect failures faster but may incorrectly mark slow nodes as failed. Production systems often use adaptive failure detectors like Phi Accrual."

> "Why use quorums?"

"Quorums ensure any two operations (like a write and subsequent read) have overlapping participants. With W + R > N, at least one node participates in both, guaranteeing the read sees the write. We tune W and R based on whether we optimize for read or write latency."

---

## Navigation

**Previous:** [04 — Caching & CDN](./04-CACHING-CDN.md)  
**Next:** [06 — Consistency & Consensus](./06-CONSISTENCY-CONSENSUS.md)  
**Index:** [00 — Handbook Index](./00-INDEX.md)
