# 09 — Quick Reference Card

> Essential numbers, formulas, and decision frameworks for rapid recall during interviews.

---

## 1. Latency Numbers

### Memory & Storage

| Operation | Latency | Notes |
|-----------|---------|-------|
| L1 cache reference | 0.5 ns | |
| L2 cache reference | 7 ns | 14x L1 |
| Main memory reference | 100 ns | 200x L1 |
| SSD random read | 150 μs | 1,000x memory |
| HDD seek | 10 ms | 100x SSD |

### Network

| Operation | Latency |
|-----------|---------|
| Same datacenter RTT | 0.5 ms |
| Cross-region RTT | 50-150 ms |
| Intercontinental RTT | 100-300 ms |

### Rule of Thumb
Each layer of indirection adds ~10-100x latency.

---

## 2. Throughput Numbers

| System | Throughput |
|--------|------------|
| SSD sequential read | 500 MB/s |
| 1 Gbps network | 125 MB/s |
| HDD sequential read | 100 MB/s |
| SSD random read | 10,000 IOPS |
| HDD random read | 100 IOPS |

---

## 3. Estimation Formulas

### Requests Per Second

```
RPS = (DAU × Actions per User) / 86,400

Example: 10M DAU, 10 actions/user
RPS = (10M × 10) / 86,400 ≈ 1,150 req/s
```

### Storage

```
Storage = Users × Data per User × Retention

Example: 100M users, 1KB/day, 1 year
Storage = 100M × 1KB × 365 = 36.5 TB/year
```

### Bandwidth

```
Bandwidth = RPS × Request Size

Example: 1,000 RPS, 100KB response
Bandwidth = 1,000 × 100KB = 100 MB/s
```

### Servers Needed

```
Servers = RPS / RPS per Server

Example: 10,000 RPS, server handles 1,000 RPS
Servers = 10,000 / 1,000 = 10 servers
```

---

## 4. Powers of 2

| Power | Value | Approx |
|-------|-------|--------|
| 2^10 | 1,024 | 1 Thousand (KB) |
| 2^20 | 1,048,576 | 1 Million (MB) |
| 2^30 | ~1 Billion | 1 GB |
| 2^40 | ~1 Trillion | 1 TB |

### Time

| Period | Seconds |
|--------|---------|
| 1 minute | 60 |
| 1 hour | 3,600 |
| 1 day | 86,400 (~10^5) |
| 1 month | 2.6M |
| 1 year | 31.5M (~3×10^7) |

---

## 5. Availability Table

| Availability | Downtime/Year | Downtime/Month |
|--------------|---------------|----------------|
| 99% | 3.65 days | 7.3 hours |
| 99.9% | 8.76 hours | 43.8 min |
| 99.99% | 52.6 min | 4.38 min |
| 99.999% | 5.26 min | 26.3 sec |

---

## 6. Database Selection

```
Structured + Transactions + Complex Queries → SQL
Flexible Schema + Scale + Simple Queries → NoSQL

NoSQL Type Selection:
- Key lookups → Key-Value (Redis, DynamoDB)
- Documents/JSON → Document (MongoDB)
- Time-series/Logs → Column (Cassandra)
- Relationships → Graph (Neo4j)
```

---

## 7. Caching Decision

```
Cache When:
✓ Read-heavy (read:write > 10:1)
✓ Data changes infrequently
✓ Can tolerate staleness
✓ Expensive to compute/fetch

Cache Strategy:
- Read-heavy → Cache-aside
- Write-heavy + consistency → Write-through
- Write-heavy + speed → Write-behind
```

---

## 8. Load Balancer Algorithm

| Scenario | Algorithm |
|----------|-----------|
| Equal servers, similar requests | Round Robin |
| Unequal server capacity | Weighted Round Robin |
| Variable request duration | Least Connections |
| Need session stickiness | IP Hash |
| Latency-critical | Least Response Time |

---

## 9. Consistency vs Availability

```
Need correctness (banking, inventory) → CP system
Need always-on (social, content) → AP system

During Partition:
- CP: Reject requests, maintain consistency
- AP: Serve requests, accept inconsistency
```

---

## 10. Replication Strategy

| Need | Strategy |
|------|----------|
| Strong consistency | Single-leader, sync replication |
| High availability | Multi-leader or leaderless |
| Read scaling | Single-leader + read replicas |
| Geographic distribution | Multi-leader |

---

## 11. Partitioning Strategy

```
Partition Key Selection:
✓ High cardinality (many unique values)
✓ Even distribution
✓ Matches query patterns

Sharding Methods:
- Range: Good for range queries, risk of hotspots
- Hash: Even distribution, no range queries
- Directory: Flexible, adds lookup overhead
```

---

## 12. Message Queue vs Pub/Sub

| Need | Pattern |
|------|---------|
| Work distribution | Message Queue |
| Event broadcast | Pub/Sub |
| Task processing | Message Queue |
| Notifications | Pub/Sub |

---

## 13. Real-Time Communication

| Need | Technology |
|------|------------|
| Server → Client only | SSE |
| Bidirectional | WebSocket |
| Works everywhere | Long Polling |

---

## 14. Common Capacity Numbers

| Service | Typical Capacity |
|---------|------------------|
| Web server | 1,000-10,000 req/s |
| Database (single) | 10,000-50,000 QPS |
| Redis | 100,000+ ops/s |
| Kafka partition | 10,000+ msg/s |

---

## 15. Interview Framework

### Opening (5 min)
1. Clarify requirements (functional + non-functional)
2. Establish constraints (scale, latency, consistency)
3. State assumptions

### Design (20-30 min)
1. High-level architecture
2. Data model
3. API design
4. Deep dive on components

### Closing (5 min)
1. Trade-offs acknowledged
2. Scaling considerations
3. Failure handling

### Key Phrases

```
"The trade-off here is..."
"This optimizes for X at the cost of Y..."
"If [constraint] changes, we'd revisit..."
"In production, we'd also consider..."
```

---

## Navigation

**Previous:** [08 — Messaging & Async](./08-MESSAGING-ASYNC.md)  
**Index:** [00 — Handbook Index](./00-INDEX.md)
