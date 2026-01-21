# 07 — Load Balancing & Scaling

> Distributing work efficiently across resources is fundamental to building systems that handle growth.

**Prerequisites:** [01 — Foundational Concepts](./01-FOUNDATIONAL-CONCEPTS.md), [02 — Networking & Communication](./02-NETWORKING-COMMUNICATION.md)  
**Builds toward:** [09 — Quick Reference](./09-QUICK-REFERENCE.md)  
**Estimated study time:** 3-4 hours

---

## Chapter Overview

This module covers load balancing algorithms, types of load balancers, API gateways, and strategies for scaling systems horizontally and vertically.

```mermaid
graph TD
    subgraph "Traffic Distribution"
        LB[Load Balancers]
        AG[API Gateway]
    end
    
    subgraph "Scaling Strategies"
        HS[Horizontal Scaling]
        VS[Vertical Scaling]
    end
    
    subgraph "High Availability"
        R[Redundancy]
        F[Failover]
    end
    
    LB --> HS
    AG --> LB
    HS --> R
    VS --> F
```

---

## 1. Load Balancing Fundamentals

### What Is a Load Balancer?

A load balancer distributes incoming traffic across multiple servers to:
- **Improve throughput:** Parallelize request handling
- **Increase availability:** No single point of failure
- **Enable scaling:** Add/remove servers dynamically
- **Reduce latency:** Route to fastest/nearest server

```mermaid
graph LR
    C[Clients] --> LB[Load Balancer]
    LB --> S1[Server 1]
    LB --> S2[Server 2]
    LB --> S3[Server 3]
    LB --> S4[Server N]
```

### Load Balancer Placement

```mermaid
graph TD
    Users[Users] --> LB1[LB: Edge]
    LB1 --> Web1[Web Server 1]
    LB1 --> Web2[Web Server 2]
    
    Web1 --> LB2[LB: Application Tier]
    Web2 --> LB2
    
    LB2 --> App1[App Server 1]
    LB2 --> App2[App Server 2]
    
    App1 --> LB3[LB: Database Tier]
    App2 --> LB3
    
    LB3 --> DB1[(DB Primary)]
    LB3 --> DB2[(DB Replica)]
```

**Key positions:**
1. Between users and web servers
2. Between web and application servers
3. Between application and database servers

---

## 2. Load Balancing Algorithms

### Algorithm Overview

| Algorithm | State | Complexity | Best For |
|-----------|-------|------------|----------|
| Round Robin | Stateless | O(1) | Homogeneous servers |
| Weighted Round Robin | Stateless | O(1) | Heterogeneous servers |
| Least Connections | Stateful | O(n) or O(log n) | Variable request duration |
| Weighted Least Connections | Stateful | O(n) | Heterogeneous + variable duration |
| IP Hash | Stateless | O(1) | Session affinity |
| Least Response Time | Stateful | O(n) | Latency-sensitive |
| Random | Stateless | O(1) | Simple, surprisingly effective |

### Round Robin

Distribute requests in circular order: 1 → 2 → 3 → 1 → 2 → 3...

```mermaid
graph LR
    subgraph "Request Sequence"
        R1[Req 1] --> S1[Server 1]
        R2[Req 2] --> S2[Server 2]
        R3[Req 3] --> S3[Server 3]
        R4[Req 4] --> S1
        R5[Req 5] --> S2
    end
```

| Pros | Cons |
|------|------|
| Simple to implement | Ignores server capacity |
| No state needed | Ignores current load |
| Fair distribution | Bad for long-lived connections |

### Weighted Round Robin

Like round robin, but servers get traffic proportional to their weight.

```
Server A (weight=5): Gets 5 requests per cycle
Server B (weight=3): Gets 3 requests per cycle
Server C (weight=2): Gets 2 requests per cycle
```

```mermaid
graph LR
    subgraph "Weighted Distribution"
        A["Server A (w=5)<br/>50% of requests"]
        B["Server B (w=3)<br/>30% of requests"]
        C["Server C (w=2)<br/>20% of requests"]
    end
```

**Use when:** Servers have different capacities (CPU, memory).

### Least Connections

Route to server with fewest active connections.

```mermaid
graph TD
    LB[Load Balancer]
    
    S1["Server 1<br/>Connections: 5"]
    S2["Server 2<br/>Connections: 2 ← Next request"]
    S3["Server 3<br/>Connections: 8"]
    
    LB --> S1
    LB --> S2
    LB --> S3
```

| Pros | Cons |
|------|------|
| Adapts to server load | Requires connection tracking |
| Good for variable request times | Doesn't consider capacity |
| Prevents overload | New servers may be overwhelmed |

### Weighted Least Connections

Combines weights with connection count:

```
Score = Active Connections / Weight
Route to lowest score
```

**Example:**
- Server A: 10 connections, weight 5 → Score = 2.0
- Server B: 4 connections, weight 2 → Score = 2.0
- Server C: 3 connections, weight 3 → Score = 1.0 ← Selected

### IP Hash

Hash client IP to determine server. Same IP always goes to same server.

```
server_index = hash(client_ip) % num_servers
```

```mermaid
graph TD
    C1["Client 192.168.1.1"] -->|"hash % 3 = 1"| S2
    C2["Client 192.168.1.2"] -->|"hash % 3 = 0"| S1
    C3["Client 192.168.1.3"] -->|"hash % 3 = 2"| S3
    
    S1[Server 1]
    S2[Server 2]
    S3[Server 3]
```

| Pros | Cons |
|------|------|
| Session affinity (sticky sessions) | Uneven if IPs not distributed |
| No session state needed at LB | Adding servers breaks mapping |
| Simple | Ignores server load |

**Use when:** Need stateful sessions without session store.

### Least Response Time

Route to server with lowest recent response time.

```mermaid
graph TD
    LB[Load Balancer]
    
    S1["Server 1<br/>Avg Response: 45ms"]
    S2["Server 2<br/>Avg Response: 12ms ← Selected"]
    S3["Server 3<br/>Avg Response: 78ms"]
    
    LB --> S1
    LB --> S2
    LB --> S3
```

| Pros | Cons |
|------|------|
| Optimizes for user latency | Requires response time tracking |
| Adapts to server performance | Can oscillate |
| Good for heterogeneous servers | Doesn't account for throughput |

### Algorithm Selection Guide

```mermaid
flowchart TD
    A[Choose Algorithm] --> B{Server capacity equal?}
    
    B -->|Yes| C{Request duration varies?}
    B -->|No| D[Weighted variant]
    
    C -->|No| E[Round Robin]
    C -->|Yes| F{Need session affinity?}
    
    F -->|Yes| G[IP Hash]
    F -->|No| H{Latency critical?}
    
    H -->|Yes| I[Least Response Time]
    H -->|No| J[Least Connections]
    
    D --> K{Request duration varies?}
    K -->|Yes| L[Weighted Least Connections]
    K -->|No| M[Weighted Round Robin]
```

---

## 3. Layer 4 vs Layer 7 Load Balancing

### OSI Layer Reference

| Layer | Name | Data Unit | LB Visibility |
|-------|------|-----------|---------------|
| 7 | Application | HTTP, gRPC | Headers, content, cookies |
| 4 | Transport | TCP/UDP | IP, port, connection |

### Layer 4 (Transport) Load Balancing

Operates on TCP/UDP connections. Cannot see application content.

```mermaid
graph LR
    C[Client] -->|TCP| L4[L4 LB]
    L4 -->|TCP| S1[Server 1]
    L4 -->|TCP| S2[Server 2]
    
    Note["LB sees: Source IP, Dest IP, Ports"]
```

| Pros | Cons |
|------|------|
| Very fast (no parsing) | No content-based routing |
| Protocol agnostic | Limited health checks |
| Low latency | No URL-based decisions |

### Layer 7 (Application) Load Balancing

Operates on HTTP requests. Can inspect headers, URLs, content.

```mermaid
graph LR
    C[Client] -->|HTTP| L7[L7 LB]
    L7 -->|"/api/*"| API[API Servers]
    L7 -->|"/static/*"| Static[Static Servers]
    L7 -->|"/ws/*"| WS[WebSocket Servers]
    
    Note["LB sees: URL, Headers, Cookies, Body"]
```

| Pros | Cons |
|------|------|
| Content-based routing | Higher latency (parsing) |
| URL/header decisions | More CPU intensive |
| SSL termination | Protocol specific |
| Caching, compression | Complex configuration |

### Comparison Table

| Feature | Layer 4 | Layer 7 |
|---------|---------|---------|
| **Speed** | Faster | Slower |
| **Routing decisions** | IP/Port | URL, headers, cookies |
| **SSL termination** | Pass-through | Yes |
| **Caching** | No | Yes |
| **Health checks** | TCP connect | HTTP status |
| **Use case** | High throughput | Content routing |

---

## 4. Load Balancer Types

### Hardware Load Balancers

Physical appliances (F5, Citrix, A10).

| Pros | Cons |
|------|------|
| Very high performance | Expensive |
| Specialized hardware | Proprietary |
| Enterprise support | Less flexible |

### Software Load Balancers

Software running on commodity hardware.

| Software | Type | Features |
|----------|------|----------|
| **Nginx** | L7 | HTTP, reverse proxy, caching |
| **HAProxy** | L4/L7 | High performance, battle-tested |
| **Envoy** | L7 | Service mesh, observability |
| **Traefik** | L7 | Auto-discovery, Kubernetes native |

### Cloud Load Balancers

Managed services from cloud providers.

| Provider | L4 Service | L7 Service |
|----------|------------|------------|
| **AWS** | NLB (Network) | ALB (Application) |
| **GCP** | Network LB | HTTP(S) LB |
| **Azure** | Load Balancer | Application Gateway |

| Pros | Cons |
|------|------|
| Managed, auto-scaling | Cost |
| Integrated with cloud | Vendor lock-in |
| Global distribution | Less control |

### DNS Load Balancing

DNS returns different IPs to distribute traffic.

```mermaid
sequenceDiagram
    participant C1 as Client 1
    participant C2 as Client 2
    participant DNS
    
    C1->>DNS: Resolve app.example.com
    DNS->>C1: 10.0.0.1
    
    C2->>DNS: Resolve app.example.com
    DNS->>C2: 10.0.0.2
```

| Pros | Cons |
|------|------|
| Simple | Slow propagation (TTL) |
| Geographic routing | No health awareness |
| No infrastructure | Limited algorithms |

---

## 5. Health Checks and Failover

### Health Check Types

| Type | What It Checks | Pros | Cons |
|------|---------------|------|------|
| **TCP** | Port is open | Simple, fast | Server might be unhealthy |
| **HTTP** | Returns 2xx | Application-aware | Slower |
| **Custom script** | Application logic | Most accurate | Complex |

### Health Check Configuration

```yaml
# Example HAProxy configuration
backend servers
    option httpchk GET /health
    http-check expect status 200
    
    server server1 10.0.0.1:8080 check inter 5s fall 3 rise 2
    server server2 10.0.0.2:8080 check inter 5s fall 3 rise 2
```

**Parameters:**
- **interval:** How often to check
- **timeout:** How long to wait for response
- **fall:** Failures before marking unhealthy
- **rise:** Successes before marking healthy

### Failover Patterns

```mermaid
graph TD
    subgraph "Active-Passive"
        LB1A[Active LB]
        LB1P[Passive LB<br/>Standby]
        LB1A -.->|Heartbeat| LB1P
    end
    
    subgraph "Active-Active"
        LB2A[Active LB 1]
        LB2B[Active LB 2]
        DNS[DNS] --> LB2A
        DNS --> LB2B
    end
```

---

## 6. API Gateway

### API Gateway vs Load Balancer

| Feature | Load Balancer | API Gateway |
|---------|---------------|-------------|
| **Primary function** | Distribute traffic | Manage API lifecycle |
| **Awareness** | Connection/request | API semantics |
| **Authentication** | Basic | Full (OAuth, JWT, API keys) |
| **Rate limiting** | Basic | Per-user, per-endpoint |
| **Transformation** | No | Request/response modification |
| **Analytics** | Basic metrics | API-level analytics |

### API Gateway Functions

```mermaid
graph LR
    C[Client] --> AG[API Gateway]
    
    AG --> Auth[Authentication]
    AG --> Rate[Rate Limiting]
    AG --> Trans[Transformation]
    AG --> Route[Routing]
    AG --> Cache[Caching]
    AG --> Log[Logging]
    
    Route --> S1[Service A]
    Route --> S2[Service B]
    Route --> S3[Service C]
```

### Key Capabilities

| Capability | Description | Example |
|------------|-------------|---------|
| **Request routing** | Route based on path, method | `/users/*` → User Service |
| **Authentication** | Verify identity | Validate JWT token |
| **Authorization** | Check permissions | User can access resource? |
| **Rate limiting** | Prevent abuse | 100 req/min per user |
| **Request transformation** | Modify requests | Add headers, change format |
| **Response transformation** | Modify responses | Filter fields, format |
| **Caching** | Cache responses | Cache GET responses |
| **Circuit breaker** | Handle failures | Stop calling failing service |

### API Gateway Patterns

#### Backend for Frontend (BFF)

Different gateways for different clients.

```mermaid
graph TD
    Web[Web App] --> GW1[Web Gateway]
    Mobile[Mobile App] --> GW2[Mobile Gateway]
    
    GW1 --> Services[Backend Services]
    GW2 --> Services
```

**Why:** Different clients need different data shapes, authentication, rate limits.

#### Gateway Aggregation

Gateway combines multiple service calls.

```mermaid
sequenceDiagram
    participant C as Client
    participant G as Gateway
    participant U as User Service
    participant O as Order Service
    participant P as Product Service
    
    C->>G: GET /user/123/dashboard
    
    par Parallel requests
        G->>U: GET /users/123
        G->>O: GET /orders?user=123
        G->>P: GET /recommendations/123
    end
    
    U->>G: User data
    O->>G: Orders
    P->>G: Recommendations
    
    G->>C: Combined dashboard response
```

---

## 7. Scaling Strategies

### Vertical Scaling (Scale Up)

Add more resources to existing machine.

```mermaid
graph LR
    subgraph "Vertical Scaling"
        S1["Server<br/>2 CPU, 4GB RAM"]
        S2["Server<br/>8 CPU, 32GB RAM"]
        S3["Server<br/>32 CPU, 128GB RAM"]
    end
    
    S1 --> S2 --> S3
```

| Pros | Cons |
|------|------|
| Simple | Hardware limits |
| No code changes | Single point of failure |
| No distributed complexity | Expensive at scale |

### Horizontal Scaling (Scale Out)

Add more machines.

```mermaid
graph LR
    subgraph "Horizontal Scaling"
        S1[Server 1]
        S2[Server 2]
        S3[Server 3]
        S4[Server N...]
    end
    
    LB[Load Balancer] --> S1
    LB --> S2
    LB --> S3
    LB --> S4
```

| Pros | Cons |
|------|------|
| Near-infinite scale | Distributed system complexity |
| Fault tolerance | State management challenges |
| Cost-effective | Load balancing needed |

### Stateless vs Stateful Services

```mermaid
graph TD
    subgraph "Stateless Service"
        LB1[Load Balancer]
        S1[Server 1]
        S2[Server 2]
        S3[Server 3]
        
        LB1 --> S1
        LB1 --> S2
        LB1 --> S3
        
        Note1[Any server can handle any request]
    end
    
    subgraph "Stateful Service"
        LB2[Load Balancer]
        SS1[Server 1<br/>State: User A,B]
        SS2[Server 2<br/>State: User C,D]
        
        LB2 -->|"User A,B"| SS1
        LB2 -->|"User C,D"| SS2
        
        Note2[Requests must go to correct server]
    end
```

**Guideline:** Make services stateless. Externalize state to:
- Redis/Memcached for sessions
- Database for persistent state
- Message queues for async state

### Auto-Scaling

Automatically adjust capacity based on demand.

```mermaid
graph TD
    Monitor[Metrics Monitor] --> Decision{CPU > 70%?}
    Decision -->|Yes| ScaleOut[Add instances]
    Decision -->|No| Check2{CPU < 30%?}
    Check2 -->|Yes| ScaleIn[Remove instances]
    Check2 -->|No| NoChange[Maintain]
```

**Scaling triggers:**
- CPU utilization
- Memory utilization
- Request count
- Queue depth
- Custom metrics

---

## 8. High Availability for Load Balancers

### Eliminating the SPOF

Load balancers can become single points of failure. Solutions:

#### Active-Passive (Failover)

```mermaid
graph TD
    VIP[Virtual IP] --> Active[Active LB]
    Active -.->|Heartbeat| Passive[Passive LB]
    
    Active --> S1[Servers]
    Passive -.->|Failover| S1
```

**How it works:**
1. Active LB owns the VIP
2. Passive monitors via heartbeat
3. If Active fails, Passive takes VIP

#### Active-Active

```mermaid
graph TD
    DNS[DNS] --> LB1[LB 1]
    DNS --> LB2[LB 2]
    
    LB1 --> Servers[Server Pool]
    LB2 --> Servers
```

**How it works:**
1. DNS returns both LB IPs
2. Traffic splits between both
3. Either can handle full load

---

## 9. Chapter Summary

### Key Concepts

| Concept | One-Line Definition |
|---------|---------------------|
| **Round Robin** | Distribute sequentially across servers |
| **Least Connections** | Route to server with fewest active connections |
| **Layer 4 LB** | Balance based on IP/port (fast, simple) |
| **Layer 7 LB** | Balance based on HTTP content (flexible, slower) |
| **API Gateway** | Manage API lifecycle: auth, rate limit, transform |
| **Horizontal scaling** | Add more machines |
| **Vertical scaling** | Add resources to existing machines |

### Algorithm Selection Cheat Sheet

| Scenario | Recommended Algorithm |
|----------|----------------------|
| Homogeneous servers, similar requests | Round Robin |
| Heterogeneous servers | Weighted Round Robin |
| Long-lived connections | Least Connections |
| Need session affinity | IP Hash |
| Latency-critical | Least Response Time |

### Interview Articulation Patterns

> "How would you scale this system?"

"First, I'd make the service stateless by externalizing session state to Redis. Then I'd add a load balancer in front and horizontally scale by adding instances. Auto-scaling based on CPU would handle traffic spikes. For the database, I'd add read replicas and consider sharding if write throughput becomes a bottleneck."

> "Why use an API Gateway instead of just a load balancer?"

"An API Gateway provides application-level features that a basic load balancer doesn't: authentication, rate limiting per user, request/response transformation, and API versioning. If I just need to distribute traffic, a load balancer is simpler. But for managing APIs with different access patterns, authentication, and client-specific needs, an API Gateway is more appropriate."

---

## Navigation

**Previous:** [06 — Consistency & Consensus](./06-CONSISTENCY-CONSENSUS.md)  
**Next:** [08 — Messaging & Async](./08-MESSAGING-ASYNC.md)  
**Index:** [00 — Handbook Index](./00-INDEX.md)
