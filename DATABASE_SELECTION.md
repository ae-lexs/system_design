# 05 - Database Selection: SQL vs NoSQL

## Overview

Database selection is one of the most consequential decisions in system design. The choice affects scalability, consistency guarantees, query flexibility, and operational complexity. This document provides a systematic framework for evaluating SQL and NoSQL databases, with emphasis on trade-offs that interviewers commonly probe.

---

## Core Mental Model

```mermaid
flowchart TD
    subgraph Decision["Decision Framework"]
        Q1{Data Structure<br/>Well-Defined?}
        Q2{Need ACID<br/>Transactions?}
        Q3{Read/Write<br/>Pattern?}
        Q4{Scale<br/>Requirements?}
    end
    
    Q1 -->|Yes, Relational| Q2
    Q1 -->|No, Flexible| NOSQL1[Document Store]
    Q1 -->|Graph Relations| NOSQL2[Graph DB]
    
    Q2 -->|Yes, Strong| SQL1[PostgreSQL/MySQL]
    Q2 -->|Eventual OK| Q3
    
    Q3 -->|Read Heavy| NOSQL3[Read Replicas + Cache]
    Q3 -->|Write Heavy| Q4
    
    Q4 -->|Horizontal Scale| NOSQL4[Wide Column/KV]
    Q4 -->|Moderate| SQL2[Sharded SQL]
```

**Key Insight**: The question is rarely "SQL or NoSQL" but rather "which consistency, availability, and partition-tolerance trade-offs match my requirements?"

---

## SQL Databases

### Characteristics

| Property | Description |
|----------|-------------|
| **Schema** | Fixed, predefined structure with strong typing |
| **Relationships** | First-class support via foreign keys, JOINs |
| **Transactions** | ACID guarantees (Atomicity, Consistency, Isolation, Durability) |
| **Query Language** | Declarative SQL with optimization |
| **Scaling** | Primarily vertical; horizontal via read replicas or sharding |

### When to Choose SQL

1. **Complex queries with JOINs**: Financial reporting, analytics dashboards
2. **Strong consistency requirements**: Banking, inventory management
3. **Well-defined schema**: User accounts, orders, products
4. **Referential integrity**: Systems where data relationships must be enforced

### ACID Deep Dive

```mermaid
sequenceDiagram
    participant C as Client
    participant DB as Database
    participant Log as WAL/Redo Log
    
    C->>DB: BEGIN TRANSACTION
    DB->>Log: Write intent
    C->>DB: UPDATE accounts SET balance = balance - 100 WHERE id = 1
    DB->>Log: Log change (uncommitted)
    C->>DB: UPDATE accounts SET balance = balance + 100 WHERE id = 2
    DB->>Log: Log change (uncommitted)
    C->>DB: COMMIT
    DB->>Log: Mark committed
    Log-->>DB: Flush to disk
    DB->>C: Success
    
    Note over DB,Log: If crash before COMMIT:<br/>Changes rolled back from log
```

**Interview Point**: Explain how Write-Ahead Logging (WAL) ensures durability. Changes are written to the log before being applied to data pages.

### Isolation Levels

```mermaid
graph LR
    subgraph Isolation["Isolation Levels (Weakest → Strongest)"]
        RU[Read Uncommitted] --> RC[Read Committed]
        RC --> RR[Repeatable Read]
        RR --> S[Serializable]
    end
    
    subgraph Anomalies["Prevented Anomalies"]
        RU -.->|Allows| DirtyRead[Dirty Reads]
        RC -.->|Allows| NonRepeat[Non-Repeatable Reads]
        RR -.->|Allows| Phantom[Phantom Reads]
        S -.->|Prevents| All[All Anomalies]
    end
```

| Level | Dirty Read | Non-Repeatable Read | Phantom Read | Performance |
|-------|------------|---------------------|--------------|-------------|
| Read Uncommitted | ✓ | ✓ | ✓ | Fastest |
| Read Committed | ✗ | ✓ | ✓ | Fast |
| Repeatable Read | ✗ | ✗ | ✓ | Moderate |
| Serializable | ✗ | ✗ | ✗ | Slowest |

---

## NoSQL Categories

### Taxonomy

```mermaid
flowchart TB
    NoSQL[NoSQL Databases]
    
    NoSQL --> KV[Key-Value Stores]
    NoSQL --> Doc[Document Stores]
    NoSQL --> WC[Wide-Column Stores]
    NoSQL --> Graph[Graph Databases]
    
    KV --> KV1[Redis]
    KV --> KV2[DynamoDB]
    KV --> KV3[Memcached]
    
    Doc --> Doc1[MongoDB]
    Doc --> Doc2[CouchDB]
    Doc --> Doc3[Firestore]
    
    WC --> WC1[Cassandra]
    WC --> WC2[HBase]
    WC --> WC3[ScyllaDB]
    
    Graph --> Graph1[Neo4j]
    Graph --> Graph2[Amazon Neptune]
    Graph --> Graph3[JanusGraph]
```

### Category Comparison

| Category | Data Model | Use Case | Trade-off |
|----------|------------|----------|-----------|
| **Key-Value** | Opaque blobs by key | Session, cache, config | No complex queries |
| **Document** | JSON/BSON documents | Content management, catalogs | Schema flexibility vs. JOIN complexity |
| **Wide-Column** | Rows with dynamic columns | Time-series, IoT, analytics | Write-optimized, read patterns must match |
| **Graph** | Nodes + edges | Social networks, recommendations | Traversal fast, aggregation slow |

---

## Document Stores Deep Dive (MongoDB)

### Data Model

```mermaid
erDiagram
    USER ||--o{ ORDER : places
    ORDER ||--|{ ORDER_ITEM : contains
    ORDER_ITEM }o--|| PRODUCT : references
    
    USER {
        ObjectId _id
        string name
        string email
        object address
        array preferences
    }
    
    ORDER {
        ObjectId _id
        ObjectId user_id
        date created_at
        string status
        array items_embedded
    }
```

### Embedding vs. Referencing

```mermaid
flowchart LR
    subgraph Embedding["Embedding (Denormalized)"]
        E1[User Document]
        E2["{ orders: [...] }"]
        E1 --> E2
    end
    
    subgraph Referencing["Referencing (Normalized)"]
        R1[User Document]
        R2[Order Documents]
        R1 -.->|user_id| R2
    end
    
    Embedding -->|"Pro: Single read<br/>Con: Document bloat"| Choice
    Referencing -->|"Pro: No duplication<br/>Con: Multiple reads"| Choice
    
    Choice{Choose Based On}
    Choice -->|"1:Few, Read Together"| Embed[Embed]
    Choice -->|"1:Many, Independent Access"| Ref[Reference]
```

**Decision Heuristic**:
- Embed when data is always accessed together and bounded in size
- Reference when data grows unbounded or is accessed independently

---

## Wide-Column Stores Deep Dive (Cassandra)

### Data Model

```mermaid
flowchart TB
    subgraph Cassandra["Cassandra Data Model"]
        KS[Keyspace] --> CF[Column Family/Table]
        CF --> PK[Partition Key]
        PK --> CK[Clustering Key]
        CK --> Cols[Columns]
    end
    
    subgraph Example["Example: Time-Series Sensor Data"]
        Table["sensor_readings"]
        Table --> Part["Partition: sensor_id"]
        Part --> Clust["Clustering: timestamp DESC"]
        Clust --> Data["temperature, humidity, pressure"]
    end
```

### Query Pattern Matching

```sql
-- Schema designed for specific access pattern
CREATE TABLE sensor_readings (
    sensor_id UUID,
    reading_time TIMESTAMP,
    temperature FLOAT,
    humidity FLOAT,
    PRIMARY KEY (sensor_id, reading_time)
) WITH CLUSTERING ORDER BY (reading_time DESC);

-- ✅ Supported: Query by partition key + clustering range
SELECT * FROM sensor_readings 
WHERE sensor_id = ? AND reading_time > ?;

-- ❌ Anti-pattern: Query without partition key (full scan)
SELECT * FROM sensor_readings 
WHERE temperature > 30;
```

**Interview Point**: Cassandra requires you to model data around query patterns, not around entities. This is the opposite of SQL normalization.

---

## CAP Theorem Application

```mermaid
graph TD
    subgraph CAP["CAP Theorem"]
        C[Consistency]
        A[Availability]
        P[Partition Tolerance]
    end
    
    subgraph Systems["System Classification"]
        CA[CA Systems<br/>Single-node RDBMS]
        CP[CP Systems<br/>MongoDB, HBase, Spanner]
        AP[AP Systems<br/>Cassandra, DynamoDB, CouchDB]
    end
    
    C --- CA
    A --- CA
    C --- CP
    P --- CP
    A --- AP
    P --- AP
    
    Note[In distributed systems,<br/>P is mandatory.<br/>Choose between C and A.]
```

### Practical Interpretation

| Scenario | Choose CP | Choose AP |
|----------|-----------|-----------|
| Network partition occurs | Reject writes to maintain consistency | Accept writes, resolve conflicts later |
| Banking transactions | ✓ | |
| Social media posts | | ✓ |
| Inventory (low stock) | ✓ | |
| User preferences | | ✓ |

---

## Decision Framework

### Systematic Evaluation Checklist

```mermaid
flowchart TD
    Start[New System Design] --> Q1{Data model<br/>complexity?}
    
    Q1 -->|"Many relationships,<br/>complex queries"| SQL_PATH
    Q1 -->|"Flexible schema,<br/>document-centric"| DOC_PATH
    Q1 -->|"Simple key-value,<br/>high throughput"| KV_PATH
    Q1 -->|"Time-series,<br/>append-heavy"| WC_PATH
    Q1 -->|"Graph traversals,<br/>relationships ARE data"| GRAPH_PATH
    
    SQL_PATH --> SQL_Q{Scale<br/>requirements?}
    SQL_Q -->|"<10TB, <100K QPS"| PG[PostgreSQL]
    SQL_Q -->|"Global scale,<br/>strong consistency"| Spanner[Cloud Spanner]
    SQL_Q -->|"Read-heavy,<br/>eventual OK"| Aurora[Aurora + Replicas]
    
    DOC_PATH --> MongoDB[MongoDB / DocumentDB]
    KV_PATH --> KV_Q{Persistence<br/>needed?}
    KV_Q -->|"Yes"| DDB[DynamoDB / Redis Cluster]
    KV_Q -->|"Cache only"| Memcached[Memcached / Redis]
    
    WC_PATH --> Cassandra[Cassandra / ScyllaDB]
    GRAPH_PATH --> Neo4j[Neo4j / Neptune]
```

### Trade-off Matrix

| Requirement | SQL | Document | Wide-Column | Key-Value | Graph |
|-------------|-----|----------|-------------|-----------|-------|
| Complex JOINs | ★★★★★ | ★★☆☆☆ | ★☆☆☆☆ | ☆☆☆☆☆ | ★★★★☆ |
| Schema flexibility | ★★☆☆☆ | ★★★★★ | ★★★★☆ | ★★★★★ | ★★★☆☆ |
| Write throughput | ★★★☆☆ | ★★★★☆ | ★★★★★ | ★★★★★ | ★★★☆☆ |
| Horizontal scale | ★★☆☆☆ | ★★★★☆ | ★★★★★ | ★★★★★ | ★★★☆☆ |
| ACID transactions | ★★★★★ | ★★★☆☆ | ★★☆☆☆ | ★☆☆☆☆ | ★★★☆☆ |
| Operational simplicity | ★★★★☆ | ★★★☆☆ | ★★☆☆☆ | ★★★★★ | ★★★☆☆ |

---

## Common Interview Scenarios

### Scenario 1: E-Commerce Platform

**Requirements**: Product catalog, user accounts, orders, inventory

```mermaid
flowchart LR
    subgraph Hybrid["Recommended: Polyglot Persistence"]
        PG[(PostgreSQL)]
        Mongo[(MongoDB)]
        Redis[(Redis)]
        ES[(Elasticsearch)]
    end
    
    PG -->|"Users, Orders,<br/>Inventory (ACID)"| Core[Core Transactions]
    Mongo -->|"Product Catalog,<br/>Reviews"| Flexible[Flexible Content]
    Redis -->|"Sessions, Cart,<br/>Rate Limits"| Fast[Fast Access]
    ES -->|"Product Search,<br/>Facets"| Search[Search/Discovery]
```

**Talking Points**:
- Orders require ACID → SQL
- Product attributes vary by category → Document store
- Session/cart data is ephemeral → Redis
- Full-text search → Elasticsearch

### Scenario 2: Social Network

**Requirements**: User profiles, posts, follows, feed generation

```mermaid
flowchart TB
    subgraph Storage["Storage Layer"]
        Graph[(Neo4j<br/>Social Graph)]
        Cassandra[(Cassandra<br/>Feed Storage)]
        PG[(PostgreSQL<br/>User Profiles)]
        Redis[(Redis<br/>Active Feeds Cache)]
    end
    
    Graph -->|"Who follows whom,<br/>Friend suggestions"| Social[Social Features]
    Cassandra -->|"Pre-computed feeds,<br/>Time-series posts"| Feed[Feed Generation]
    PG -->|"User accounts,<br/>Settings"| Profile[Profile Management]
    Redis -->|"Hot user feeds,<br/>Online status"| Realtime[Real-time Data]
```

### Scenario 3: IoT Analytics

**Requirements**: Millions of sensors, time-series data, real-time dashboards

**Recommended**: Wide-column store (Cassandra/TimescaleDB)

**Why**:
- Append-only writes (perfect for LSM trees)
- Queries always include time range (partition by sensor + time)
- No complex JOINs needed
- Linear horizontal scaling

---

## Interview Tips

### What Interviewers Look For

1. **Systematic reasoning**: Don't jump to "use Postgres" without explaining why
2. **Trade-off awareness**: Every choice has costs; articulate them
3. **Real-world experience**: Mention operational concerns (backups, failover, monitoring)
4. **Polyglot thinking**: Best designs often use multiple databases

### Common Mistakes

| Mistake | Better Approach |
|---------|-----------------|
| "NoSQL scales better" | "NoSQL trades consistency for horizontal scale" |
| "MongoDB for everything" | "Document stores excel when schema flexibility matters" |
| "SQL can't scale" | "SQL scales with read replicas, sharding, or NewSQL" |
| Ignoring operational cost | Consider team expertise, managed services, vendor lock-in |

### Key Phrases to Use

- "Given the read/write ratio of X:Y..."
- "Since we need strong consistency for..."
- "The query pattern suggests..."
- "To optimize for the hot path..."
- "The trade-off here is..."

---

## Quick Reference Card

```
┌─────────────────────────────────────────────────────────────────┐
│                    DATABASE SELECTION CHEAT SHEET               │
├─────────────────────────────────────────────────────────────────┤
│ ACID Required?           → Start with SQL (PostgreSQL)          │
│ Flexible Schema?         → Document Store (MongoDB)             │
│ Simple K/V, High Speed?  → Redis / DynamoDB                     │
│ Time-Series, Write-Heavy? → Wide-Column (Cassandra)             │
│ Relationship Traversal?  → Graph DB (Neo4j)                     │
│ Global Strong Consistency? → NewSQL (Spanner, CockroachDB)      │
│ Full-Text Search?        → Elasticsearch (+ primary DB)         │
├─────────────────────────────────────────────────────────────────┤
│ REMEMBER: Most systems benefit from polyglot persistence        │
│ Use the right tool for each data type / access pattern          │
└─────────────────────────────────────────────────────────────────┘
```

---

## Practice Questions

1. Design a database schema for a ride-sharing app. Which databases would you use and why?
2. Your e-commerce site has 10M products with varying attributes. How do you store them?
3. Explain how you'd migrate from a monolithic SQL database to a microservices architecture with multiple databases.
4. A social network needs to store and query follower relationships. Compare SQL JOINs vs. graph database for this use case.
5. Your write volume is 1M events/second. Which database technology fits best?
