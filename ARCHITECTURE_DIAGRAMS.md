# 11 - Architecture Paradigms: Serverless vs Traditional, Stateful vs Stateless

## Overview

Architecture paradigms define how systems are structured, deployed, and scaled. The choice between serverless and traditional architectures, and between stateful and stateless designs, fundamentally affects operational complexity, cost, scalability, and developer experience. This document provides a systematic framework for evaluating these architectural decisions.

---

## Core Mental Model

```mermaid
flowchart TD
    subgraph Paradigms["Architecture Paradigms"]
        subgraph Deployment["Deployment Model"]
            Trad[Traditional<br/>Long-running servers]
            SL[Serverless<br/>Event-driven functions]
        end
        
        subgraph State["State Management"]
            Stateful[Stateful<br/>Server holds state]
            Stateless[Stateless<br/>External state store]
        end
    end
    
    subgraph Combinations["Common Combinations"]
        C1[Traditional + Stateful<br/>Monolith with sessions]
        C2[Traditional + Stateless<br/>Stateless microservices]
        C3[Serverless + Stateless<br/>Functions + DynamoDB]
    end
```

**Key Insight**: These paradigms are orthogonal. Serverless can be stateful (with external state), and traditional servers can be stateless. The combination you choose depends on your specific requirements.

---

## Stateless vs Stateful Services

### Fundamental Difference

```mermaid
flowchart LR
    subgraph Stateful["Stateful Service"]
        Client1[Client] -->|"Request 1"| Server1[Server]
        Server1 -->|"Store session"| Server1
        Client1 -->|"Request 2"| Server1
        Server1 -->|"Use stored session"| Response1[Response]
    end
    
    subgraph Stateless["Stateless Service"]
        Client2[Client] -->|"Request (with context)"| LB[Load Balancer]
        LB --> S1[Server 1]
        LB --> S2[Server 2]
        S1 & S2 --> Store[(External State)]
    end
```

### Comparison

| Aspect | Stateful | Stateless |
|--------|----------|-----------|
| **Scaling** | Complex (session affinity) | Simple (any instance) |
| **Failure handling** | State lost on crash | No state to lose |
| **Load balancing** | Sticky sessions required | Round-robin works |
| **Deployment** | Drain sessions first | Replace anytime |
| **Memory usage** | Higher (holds state) | Lower |
| **Latency** | Lower (local state) | Higher (external lookup) |

### Stateless Design Pattern

```mermaid
sequenceDiagram
    participant Client
    participant LB as Load Balancer
    participant S1 as Server 1
    participant S2 as Server 2
    participant Redis as Redis/Cache
    
    Client->>LB: Request 1 (token: abc123)
    LB->>S1: Forward request
    S1->>Redis: GET session:abc123
    Redis->>S1: { user_id: 42, cart: [...] }
    S1->>Client: Response
    
    Note over LB: Next request goes to different server
    
    Client->>LB: Request 2 (token: abc123)
    LB->>S2: Forward request
    S2->>Redis: GET session:abc123
    Redis->>S2: { user_id: 42, cart: [...] }
    S2->>Client: Response
```

### Making Services Stateless

```mermaid
flowchart TB
    subgraph Before["Stateful Design ❌"]
        B1["User sessions in server memory"]
        B2["File uploads stored locally"]
        B3["Background jobs tracked in memory"]
    end
    
    subgraph After["Stateless Design ✓"]
        A1["Sessions in Redis/DynamoDB"]
        A2["Files in S3/GCS"]
        A3["Jobs in SQS/Celery with Redis"]
    end
    
    B1 -->|"Externalize"| A1
    B2 -->|"Externalize"| A2
    B3 -->|"Externalize"| A3
```

### State Externalization Strategies

| State Type | Solution |
|------------|----------|
| **User sessions** | Redis, Memcached, DynamoDB |
| **Shopping carts** | Redis with persistence, database |
| **File uploads** | S3, GCS, Azure Blob |
| **Background job state** | Message queue + database |
| **In-memory cache** | Distributed cache (Redis) |
| **Locks/coordination** | Redis, ZooKeeper, etcd |

---

## Traditional Architecture

### Characteristics

```mermaid
flowchart TB
    subgraph Traditional["Traditional Server Architecture"]
        LB[Load Balancer]
        
        subgraph Servers["Server Fleet"]
            S1[Server 1<br/>Always Running]
            S2[Server 2<br/>Always Running]
            S3[Server 3<br/>Always Running]
        end
        
        DB[(Database)]
        Cache[(Cache)]
    end
    
    LB --> Servers
    Servers --> DB
    Servers --> Cache
```

### Deployment Models

```mermaid
flowchart LR
    subgraph Models["Traditional Deployment"]
        VM[VMs<br/>EC2, GCE]
        Container[Containers<br/>ECS, Kubernetes]
        PaaS[PaaS<br/>Heroku, App Engine]
    end
```

| Model | Control | Complexity | Use Case |
|-------|---------|------------|----------|
| **VMs** | Full | High | Legacy, specific OS needs |
| **Containers** | Application-level | Medium | Microservices, portability |
| **PaaS** | Minimal | Low | Rapid development, small teams |

### Scaling Traditional Services

```mermaid
flowchart TB
    subgraph Scaling["Scaling Strategies"]
        Vertical["Vertical Scaling<br/>Bigger machines"]
        Horizontal["Horizontal Scaling<br/>More machines"]
    end
    
    subgraph Auto["Auto-scaling"]
        CPU["CPU-based<br/>Scale at 70% CPU"]
        Memory["Memory-based<br/>Scale at 80% memory"]
        Custom["Custom metrics<br/>Queue depth, latency"]
        Schedule["Schedule-based<br/>Known traffic patterns"]
    end
```

```yaml
# Kubernetes HPA example
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-server
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api-server
  minReplicas: 3
  maxReplicas: 100
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Pods
    pods:
      metric:
        name: requests_per_second
      target:
        type: AverageValue
        averageValue: 1000
```

---

## Serverless Architecture

### Characteristics

```mermaid
flowchart TB
    subgraph Serverless["Serverless Architecture"]
        Trigger[Event Triggers]
        
        subgraph Functions["Functions (ephemeral)"]
            F1["Function Instance<br/>(created on demand)"]
            F2["Function Instance<br/>(created on demand)"]
            F3["Function Instance<br/>(created on demand)"]
        end
        
        Services[Managed Services]
    end
    
    Trigger -->|"HTTP, Queue,<br/>Schedule, etc."| Functions
    Functions --> Services
    
    Services --> DB[(DynamoDB)]
    Services --> S3[(S3)]
    Services --> SQS[(SQS)]
```

### Function Lifecycle

```mermaid
sequenceDiagram
    participant Event
    participant Platform as Serverless Platform
    participant Container as Function Container
    participant Code as Your Code
    
    Event->>Platform: Trigger (HTTP request)
    
    alt Cold Start (no warm container)
        Platform->>Container: Provision container
        Container->>Container: Download code
        Container->>Code: Initialize runtime
        Code->>Code: Run initialization code
        Note over Container,Code: Cold start latency<br/>(100ms - 10s)
    else Warm Start (reused container)
        Note over Container: Container already warm
    end
    
    Code->>Code: Execute handler
    Code->>Platform: Return response
    Platform->>Event: Response
    
    Note over Container: Container may be<br/>kept warm for reuse
```

### Cold Start Mitigation

```mermaid
flowchart TB
    subgraph Strategies["Cold Start Mitigation"]
        Warm[Provisioned Concurrency]
        Small[Smaller Packages]
        Lang[Language Choice]
        Init[Minimize Init Code]
    end
    
    Warm -->|"Pre-warm containers"| W1["AWS: Provisioned Concurrency<br/>GCP: Min Instances"]
    Small -->|"Reduce deployment size"| S1["Exclude dev dependencies<br/>Use layers/shared code"]
    Lang -->|"Faster languages"| L1["Go, Rust > Python > Java"]
    Init -->|"Lazy initialization"| I1["DB connections on first use<br/>Avoid heavy imports"]
```

### Serverless Patterns

```mermaid
flowchart TB
    subgraph Patterns["Common Serverless Patterns"]
        API[API Backend]
        Queue[Queue Processor]
        Schedule[Scheduled Task]
        Stream[Stream Processor]
        Webhook[Webhook Handler]
    end
    
    API -->|"API Gateway → Lambda"| API_Ex[REST/GraphQL APIs]
    Queue -->|"SQS → Lambda"| Queue_Ex[Async processing]
    Schedule -->|"CloudWatch Events → Lambda"| Schedule_Ex[Cron jobs, cleanup]
    Stream -->|"Kinesis → Lambda"| Stream_Ex[Real-time analytics]
    Webhook -->|"HTTP → Lambda"| Webhook_Ex[Third-party integrations]
```

---

## Serverless vs Traditional: Detailed Comparison

### Cost Model

```mermaid
flowchart LR
    subgraph Traditional["Traditional Cost"]
        T1["Fixed: Servers always running"]
        T2["Pay for idle time"]
        T3["Predictable monthly bill"]
    end
    
    subgraph Serverless["Serverless Cost"]
        S1["Variable: Pay per invocation"]
        S2["Zero cost when idle"]
        S3["Can spike with traffic"]
    end
```

```
Traditional Cost Example:
- 3 x t3.medium EC2 instances
- Running 24/7 for 30 days
- Cost: 3 × $0.0416/hr × 720 hrs = ~$90/month
- Even if handling 0 requests

Serverless Cost Example (AWS Lambda):
- 1 million requests/month
- 200ms average duration, 256MB memory
- Compute: 1M × 0.2s × 256MB = 50,000 GB-seconds
- Cost: $0.0000166667 × 50,000 + $0.20 = ~$1.03/month

Crossover point: Where serverless becomes more expensive
- Depends on traffic patterns and execution time
- Typically: High, consistent traffic favors traditional
```

### Comparison Matrix

| Aspect | Traditional | Serverless |
|--------|-------------|------------|
| **Scaling** | Manual/auto-scaling config | Automatic, instant |
| **Cold start** | None (always running) | Can be 100ms-10s |
| **Max execution** | Unlimited | Limited (15 min Lambda) |
| **State** | Can be stateful | Must be stateless |
| **Debugging** | SSH, local reproduction | CloudWatch, harder to debug |
| **Vendor lock-in** | Lower (portable containers) | Higher (proprietary triggers) |
| **Cost at scale** | More predictable | Can be surprising |
| **Ops overhead** | Higher (patching, scaling) | Lower (managed) |
| **Development speed** | Slower setup | Faster prototyping |

### Decision Framework

```mermaid
flowchart TD
    Start[New Service] --> Q1{Execution<br/>duration?}
    
    Q1 -->|"> 15 minutes"| Trad1[Traditional]
    Q1 -->|"< 15 minutes"| Q2{Traffic<br/>pattern?}
    
    Q2 -->|"Spiky, unpredictable"| SL1[Serverless]
    Q2 -->|"Steady, high volume"| Q3{Latency<br/>requirements?}
    
    Q3 -->|"Cold start OK"| SL2[Serverless]
    Q3 -->|"Sub-100ms required"| Q4{Team<br/>expertise?}
    
    Q4 -->|"Ops capability"| Trad2[Traditional]
    Q4 -->|"Prefer managed"| SL3["Serverless +<br/>Provisioned Concurrency"]
```

---

## Hybrid Architectures

### Real-World Pattern

```mermaid
flowchart TB
    subgraph Hybrid["Hybrid Architecture"]
        subgraph Serverless["Serverless Components"]
            Lambda1[Image Processing<br/>Lambda]
            Lambda2[Webhook Handler<br/>Lambda]
            Lambda3[Scheduled Reports<br/>Lambda]
        end
        
        subgraph Traditional["Traditional Components"]
            API[API Servers<br/>ECS/Kubernetes]
            WS[WebSocket Server<br/>EC2]
            ML[ML Inference<br/>GPU instances]
        end
        
        subgraph Shared["Shared Infrastructure"]
            DB[(PostgreSQL RDS)]
            Cache[(ElastiCache)]
            Queue[(SQS)]
            S3[(S3)]
        end
    end
    
    Lambda1 --> S3
    Lambda2 --> Queue
    Lambda3 --> DB
    
    API --> DB
    API --> Cache
    WS --> Cache
    ML --> S3
```

### When to Mix

| Use Serverless For | Use Traditional For |
|--------------------|--------------------|
| Event-driven processing | Long-running processes |
| Variable/spiky workloads | Steady high throughput |
| Cron jobs, scheduled tasks | WebSocket connections |
| Webhooks, integrations | GPU/specialized hardware |
| Quick prototypes | Latency-critical paths |
| Infrequent batch jobs | Stateful applications |

---

## Stateful Serverless Patterns

### Durable Functions / Step Functions

```mermaid
stateDiagram-v2
    [*] --> ReceiveOrder
    ReceiveOrder --> ValidatePayment
    ValidatePayment --> ProcessPayment: Valid
    ValidatePayment --> RejectOrder: Invalid
    ProcessPayment --> ReserveInventory
    ReserveInventory --> ShipOrder: Available
    ReserveInventory --> Backorder: Unavailable
    ShipOrder --> SendConfirmation
    Backorder --> NotifyCustomer
    SendConfirmation --> [*]
    NotifyCustomer --> [*]
    RejectOrder --> [*]
```

```python
# AWS Step Functions definition
{
    "Comment": "Order Processing Workflow",
    "StartAt": "ValidatePayment",
    "States": {
        "ValidatePayment": {
            "Type": "Task",
            "Resource": "arn:aws:lambda:...:validate-payment",
            "Next": "ProcessPayment",
            "Catch": [{
                "ErrorEquals": ["PaymentError"],
                "Next": "RejectOrder"
            }]
        },
        "ProcessPayment": {
            "Type": "Task",
            "Resource": "arn:aws:lambda:...:process-payment",
            "Next": "ReserveInventory"
        },
        "ReserveInventory": {
            "Type": "Task",
            "Resource": "arn:aws:lambda:...:reserve-inventory",
            "Next": "ShipOrder",
            "Catch": [{
                "ErrorEquals": ["OutOfStockError"],
                "Next": "Backorder"
            }]
        }
        // ... more states
    }
}
```

### Actor Model (Durable Entities)

```mermaid
flowchart TB
    subgraph Actors["Actor Pattern"]
        A1["Shopping Cart Actor<br/>user-123"]
        A2["Shopping Cart Actor<br/>user-456"]
        A3["Inventory Actor<br/>product-789"]
    end
    
    subgraph Operations
        Op1["AddItem(item)"]
        Op2["RemoveItem(item)"]
        Op3["Checkout()"]
    end
    
    Op1 --> A1
    Op2 --> A1
    Op3 --> A1
    
    A1 -->|"Reserve"| A3
```

---

## Interview Scenarios

### Scenario 1: Image Processing Pipeline

**Requirements**: Users upload images, need thumbnails and multiple sizes

```mermaid
flowchart LR
    subgraph Recommendation["Recommended: Serverless"]
        Upload[S3 Upload] -->|"Trigger"| Lambda[Resize Lambda]
        Lambda --> S3Out[S3 Output]
    end
```

**Talking Points**:
- Event-driven (S3 trigger)
- Variable workload (uploads are spiky)
- No state needed between invocations
- Pay only for processing time
- Easy to parallelize

### Scenario 2: Real-Time Gaming Server

**Requirements**: Multiplayer game, 60 FPS updates, persistent connections

```mermaid
flowchart TB
    subgraph Recommendation["Recommended: Traditional (Stateful)"]
        LB[Load Balancer<br/>Sticky sessions]
        
        subgraph Servers["Game Servers"]
            GS1[Game Server 1<br/>Holds game state]
            GS2[Game Server 2<br/>Holds game state]
        end
        
        Match[Matchmaking Service]
    end
    
    LB --> Servers
    Match --> Servers
```

**Talking Points**:
- Persistent WebSocket connections
- Sub-millisecond latency required
- In-memory game state for performance
- Serverless cold starts unacceptable
- Long-running connections

### Scenario 3: E-Commerce API

**Requirements**: Product catalog, cart, checkout

```mermaid
flowchart TB
    subgraph Recommendation["Recommended: Hybrid"]
        subgraph Serverless
            Webhook[Payment Webhooks<br/>Lambda]
            Email[Email Sender<br/>Lambda]
            Report[Daily Reports<br/>Lambda]
        end
        
        subgraph Traditional
            API[API Servers<br/>ECS Fargate]
        end
        
        subgraph State["Stateless Design"]
            Redis[(Redis<br/>Sessions, Cart)]
            PG[(PostgreSQL<br/>Products, Orders)]
        end
    end
    
    API --> Redis
    API --> PG
    Webhook --> PG
```

**Talking Points**:
- Core API: Traditional for consistent latency
- Webhooks: Serverless (event-driven, variable load)
- Background jobs: Serverless (pay per use)
- Stateless API servers with external state
- Can scale API horizontally

---

## Migration Strategies

### Monolith to Stateless Microservices

```mermaid
flowchart TB
    subgraph Before["Before: Stateful Monolith"]
        Mono[Monolith<br/>Sessions in memory<br/>Files on disk]
    end
    
    subgraph During["Strangler Fig Pattern"]
        LB[Load Balancer]
        Mono2[Legacy Monolith]
        MS1[New Service 1]
        MS2[New Service 2]
        ExtState[(External State)]
    end
    
    subgraph After["After: Stateless Services"]
        MS_Final1[Service 1]
        MS_Final2[Service 2]
        MS_Final3[Service 3]
        State[(Shared State Layer)]
    end
    
    Before --> During
    During --> After
```

### Traditional to Serverless

```mermaid
flowchart LR
    subgraph Steps["Migration Steps"]
        S1["1. Identify stateless<br/>components"]
        S2["2. Extract to<br/>functions"]
        S3["3. Update triggers<br/>(API GW, events)"]
        S4["4. Migrate state<br/>to managed services"]
        S5["5. Decommission<br/>servers"]
    end
    
    S1 --> S2 --> S3 --> S4 --> S5
```

---

## Quick Reference Card

```
┌─────────────────────────────────────────────────────────────────┐
│              ARCHITECTURE PARADIGMS CHEAT SHEET                  │
├─────────────────────────────────────────────────────────────────┤
│ STATELESS DESIGN:                                               │
│   • No server-side session state                                │
│   • Any instance can handle any request                         │
│   • Externalize state: Redis, S3, DynamoDB                      │
│   • Benefits: Easy scaling, deployment, failure recovery        │
├─────────────────────────────────────────────────────────────────┤
│ STATEFUL DESIGN:                                                │
│   • Server maintains state between requests                     │
│   • Requires sticky sessions or session replication             │
│   • Use for: Gaming, real-time collaboration                    │
│   • Trade-off: Performance vs. operational complexity           │
├─────────────────────────────────────────────────────────────────┤
│ SERVERLESS:                                                     │
│   • Event-driven, pay-per-invocation                            │
│   • Auto-scaling, zero ops                                      │
│   • Watch for: Cold starts, execution limits, vendor lock-in    │
│   • Best for: Variable workloads, event processing              │
├─────────────────────────────────────────────────────────────────┤
│ TRADITIONAL:                                                    │
│   • Long-running servers, predictable costs                     │
│   • Full control, any runtime/language                          │
│   • More ops: Patching, scaling, monitoring                     │
│   • Best for: Steady traffic, low-latency, long processes       │
├─────────────────────────────────────────────────────────────────┤
│ DECISION HEURISTIC:                                             │
│   • Spiky traffic? → Serverless                                 │
│   • Long-running? → Traditional                                 │
│   • Real-time? → Traditional (often stateful)                   │
│   • Event-driven? → Serverless                                  │
│   • Need easy scaling? → Stateless                              │
│   • Performance-critical state? → Stateful (carefully)          │
└─────────────────────────────────────────────────────────────────┘
```

---

## Practice Questions

1. How would you migrate a monolithic application with in-memory sessions to a stateless microservices architecture?
2. Design a notification system. Would you use serverless or traditional architecture? Justify your choice.
3. Explain the trade-offs of using provisioned concurrency in AWS Lambda vs. running containers in ECS.
4. Your serverless function has cold starts of 3 seconds. How would you mitigate this?
5. Design a system that needs both real-time WebSocket communication and batch processing. How would you architect it?
