# 10 - Service Exposure: API Gateway and Proxies

## Overview

Service exposure determines how clients access backend services. API Gateways and proxies provide a unified entry point, handling cross-cutting concerns like authentication, rate limiting, routing, and protocol translation. This document covers the patterns, trade-offs, and implementation considerations for exposing services in modern architectures.

---

## Core Mental Model

```mermaid
flowchart LR
    subgraph Clients["Clients"]
        Web[Web App]
        Mobile[Mobile App]
        Partner[Partner API]
    end
    
    subgraph Edge["Edge Layer"]
        Gateway[API Gateway]
    end
    
    subgraph Services["Backend Services"]
        Users[User Service]
        Orders[Order Service]
        Products[Product Service]
    end
    
    Clients --> Gateway
    Gateway --> Users
    Gateway --> Orders
    Gateway --> Products
```

**Key Insight**: The API Gateway is the "front door" to your microservices. It centralizes cross-cutting concerns and provides a stable interface even as backend services evolve.

---

## Proxy Types

### Taxonomy

```mermaid
flowchart TB
    subgraph Proxies["Proxy Classification"]
        FP[Forward Proxy]
        RP[Reverse Proxy]
        LB[Load Balancer]
        GW[API Gateway]
        SM[Service Mesh Sidecar]
    end
    
    FP -->|"Client → Proxy → Internet"| FPUse["Hide client identity,<br/>content filtering"]
    RP -->|"Internet → Proxy → Servers"| RPUse["Hide server identity,<br/>load balancing"]
    LB -->|"Distribute traffic"| LBUse["Round-robin, least-conn,<br/>health checks"]
    GW -->|"API management"| GWUse["Auth, rate limit,<br/>routing, transformation"]
    SM -->|"Service-to-service"| SMUse["mTLS, observability,<br/>traffic control"]
```

### Forward vs Reverse Proxy

```mermaid
flowchart LR
    subgraph Forward["Forward Proxy"]
        C1[Client] --> FP[Forward Proxy]
        FP --> Internet1[Internet/Servers]
    end
    
    subgraph Reverse["Reverse Proxy"]
        Internet2[Internet/Clients] --> RP[Reverse Proxy]
        RP --> S1[Server 1]
        RP --> S2[Server 2]
    end
```

| Type | Direction | Purpose | Examples |
|------|-----------|---------|----------|
| **Forward Proxy** | Client → Proxy → Server | Anonymity, caching, filtering | Squid, corporate proxies |
| **Reverse Proxy** | Client → Proxy → Backend | Load balancing, SSL termination | Nginx, HAProxy |

---

## API Gateway

### Core Responsibilities

```mermaid
flowchart TB
    subgraph Gateway["API Gateway Responsibilities"]
        subgraph Security["Security"]
            Auth[Authentication]
            AuthZ[Authorization]
            WAF[WAF/DDoS Protection]
        end
        
        subgraph Traffic["Traffic Management"]
            Route[Routing]
            LB2[Load Balancing]
            Rate[Rate Limiting]
            CB[Circuit Breaking]
        end
        
        subgraph Transform["Transformation"]
            Proto[Protocol Translation]
            Agg[Request Aggregation]
            Cache2[Caching]
            Format[Response Formatting]
        end
        
        subgraph Observe["Observability"]
            Log[Logging]
            Metrics[Metrics]
            Trace[Distributed Tracing]
        end
    end
```

### Request Flow

```mermaid
sequenceDiagram
    participant Client
    participant Gateway as API Gateway
    participant Auth as Auth Service
    participant Rate as Rate Limiter
    participant Backend as Backend Service
    
    Client->>Gateway: GET /api/v1/users/123
    
    Gateway->>Rate: Check rate limit
    Rate->>Gateway: OK (within limits)
    
    Gateway->>Auth: Validate token
    Auth->>Gateway: Valid, user_id=456
    
    Gateway->>Gateway: Route to user-service
    Gateway->>Backend: GET /users/123 (X-User-Id: 456)
    Backend->>Gateway: { user data }
    
    Gateway->>Gateway: Transform response
    Gateway->>Client: { formatted user data }
    
    Note over Gateway: Log request metrics
```

### Routing Strategies

```mermaid
flowchart TB
    subgraph Routing["Routing Strategies"]
        Path[Path-Based]
        Header[Header-Based]
        Query[Query-Based]
        Version[Version-Based]
    end
    
    Path --> PathEx["/users/* → user-service<br/>/orders/* → order-service"]
    Header --> HeaderEx["X-Version: 2 → v2-service<br/>Accept-Language → regional service"]
    Query --> QueryEx["?partner=acme → acme-integration"]
    Version --> VersionEx["/v1/* → legacy<br/>/v2/* → new-service"]
```

```yaml
# Kong declarative config example
services:
  - name: user-service
    url: http://user-svc:8080
    routes:
      - name: user-routes
        paths:
          - /api/v1/users
        methods:
          - GET
          - POST
        
  - name: order-service
    url: http://order-svc:8080
    routes:
      - name: order-routes
        paths:
          - /api/v1/orders
        headers:
          X-API-Version:
            - "2"
```

---

## Load Balancing

### Algorithms

```mermaid
flowchart TB
    subgraph Algorithms["Load Balancing Algorithms"]
        RR[Round Robin]
        WRR[Weighted Round Robin]
        LC[Least Connections]
        WLC[Weighted Least Connections]
        IP[IP Hash]
        RND[Random]
        Latency[Least Latency]
    end
```

| Algorithm | Description | Best For |
|-----------|-------------|----------|
| **Round Robin** | Cycle through servers sequentially | Equal capacity servers |
| **Weighted Round Robin** | RR with server weights | Mixed capacity servers |
| **Least Connections** | Route to server with fewest active connections | Long-lived connections |
| **IP Hash** | Hash client IP to server | Session affinity needs |
| **Least Latency** | Route to fastest responding server | Latency-sensitive apps |
| **Random** | Random selection | Simple, stateless |

### Health Checking

```mermaid
sequenceDiagram
    participant LB as Load Balancer
    participant S1 as Server 1 (Healthy)
    participant S2 as Server 2 (Failing)
    participant S3 as Server 3 (Healthy)
    
    loop Every 10 seconds
        LB->>S1: GET /health
        S1->>LB: 200 OK
        LB->>S2: GET /health
        S2--xLB: Timeout / 500
        LB->>S3: GET /health
        S3->>LB: 200 OK
    end
    
    Note over LB,S2: After 3 failures:<br/>Remove S2 from rotation
    
    LB->>LB: Route traffic to S1, S3 only
    
    Note over S2: Server recovers
    LB->>S2: GET /health
    S2->>LB: 200 OK
    
    Note over LB,S2: After 2 successes:<br/>Add S2 back
```

### Health Check Types

| Type | Description | Use Case |
|------|-------------|----------|
| **TCP** | Can establish connection | Basic availability |
| **HTTP** | Returns 2xx status | Application-level health |
| **gRPC** | gRPC health protocol | gRPC services |
| **Script** | Custom check script | Complex conditions |

```nginx
# Nginx upstream health check
upstream backend {
    server backend1.example.com:8080;
    server backend2.example.com:8080;
    server backend3.example.com:8080 backup;
    
    health_check interval=10s fails=3 passes=2;
}
```

---

## Authentication at the Gateway

### Patterns

```mermaid
flowchart TB
    subgraph Patterns["Auth Patterns"]
        Terminate[Auth Termination]
        Passthrough[Auth Passthrough]
        Hybrid[Hybrid]
    end
    
    Terminate -->|"Gateway validates token,<br/>passes user context"| T1["+ Centralized auth<br/>+ Backend simplicity<br/>- Gateway complexity"]
    
    Passthrough -->|"Gateway forwards token,<br/>backend validates"| P1["+ Backend control<br/>- Duplicate logic<br/>- Network calls"]
    
    Hybrid -->|"Gateway validates format,<br/>backend validates claims"| H1["+ Balance of concerns<br/>+ Flexible"]
```

### JWT Validation Flow

```mermaid
sequenceDiagram
    participant Client
    participant Gateway
    participant JWKS as JWKS Endpoint
    participant Backend
    
    Note over Gateway,JWKS: On startup / periodically
    Gateway->>JWKS: GET /.well-known/jwks.json
    JWKS->>Gateway: { keys: [...] }
    Gateway->>Gateway: Cache public keys
    
    Client->>Gateway: GET /api/resource<br/>Authorization: Bearer <JWT>
    Gateway->>Gateway: Validate JWT signature (cached key)
    Gateway->>Gateway: Validate exp, iss, aud claims
    Gateway->>Backend: Request + X-User-Id + X-Roles headers
    Backend->>Gateway: Response
    Gateway->>Client: Response
```

```javascript
// Gateway JWT validation middleware (Node.js)
const jwt = require('jsonwebtoken');
const jwksClient = require('jwks-rsa');

const client = jwksClient({
    jwksUri: 'https://auth.example.com/.well-known/jwks.json',
    cache: true,
    rateLimit: true
});

async function validateJWT(req, res, next) {
    const token = req.headers.authorization?.split(' ')[1];
    if (!token) return res.status(401).json({ error: 'No token' });
    
    try {
        const decoded = jwt.decode(token, { complete: true });
        const key = await client.getSigningKey(decoded.header.kid);
        const publicKey = key.getPublicKey();
        
        const payload = jwt.verify(token, publicKey, {
            issuer: 'https://auth.example.com',
            audience: 'my-api'
        });
        
        // Pass user context to backend
        req.headers['X-User-Id'] = payload.sub;
        req.headers['X-Roles'] = payload.roles.join(',');
        next();
    } catch (err) {
        res.status(401).json({ error: 'Invalid token' });
    }
}
```

---

## Rate Limiting at the Gateway

### Algorithms

```mermaid
flowchart TB
    subgraph Algorithms["Rate Limiting Algorithms"]
        Fixed[Fixed Window]
        Sliding[Sliding Window]
        Token[Token Bucket]
        Leaky[Leaky Bucket]
    end
```

*Note: See Document 12 (Traffic Management) for detailed algorithm explanations.*

### Implementation Pattern

```mermaid
flowchart LR
    subgraph Gateway["API Gateway"]
        Check[Check Limit]
        Redis[(Redis)]
    end
    
    Request --> Check
    Check <-->|"GET/INCR"| Redis
    Check -->|"Under limit"| Allow[Allow Request]
    Check -->|"Over limit"| Reject["429 Too Many Requests"]
```

```lua
-- Nginx + Lua rate limiting
local limit = require "resty.limit.req"
local lim, err = limit.new("rate_limit_store", 100, 50)  -- 100 req/s, burst 50

if not lim then
    ngx.log(ngx.ERR, "failed to instantiate limiter: ", err)
    return ngx.exit(500)
end

local key = ngx.var.remote_addr  -- or API key, user ID
local delay, err = lim:incoming(key, true)

if not delay then
    if err == "rejected" then
        ngx.header["Retry-After"] = 1
        return ngx.exit(429)
    end
    return ngx.exit(500)
end

if delay >= 0.001 then
    ngx.sleep(delay)  -- Throttle (leaky bucket behavior)
end
```

---

## Backend for Frontend (BFF)

### Pattern

```mermaid
flowchart TB
    subgraph Clients
        Web[Web App]
        Mobile[Mobile App]
        Partner[Partner API]
    end
    
    subgraph BFFs["Backend for Frontend Layer"]
        WebBFF[Web BFF]
        MobileBFF[Mobile BFF]
        PartnerGW[Partner Gateway]
    end
    
    subgraph Services["Microservices"]
        Users[User Service]
        Orders[Order Service]
        Products[Product Service]
    end
    
    Web --> WebBFF
    Mobile --> MobileBFF
    Partner --> PartnerGW
    
    WebBFF --> Users
    WebBFF --> Orders
    WebBFF --> Products
    
    MobileBFF --> Users
    MobileBFF --> Orders
    
    PartnerGW --> Orders
    PartnerGW --> Products
```

### Why BFF?

| Concern | Single Gateway | BFF per Client |
|---------|----------------|----------------|
| **Response format** | One size fits all | Tailored per client |
| **Aggregation** | Complex, generic | Optimized for UI |
| **Versioning** | Hard to evolve | Independent evolution |
| **Team ownership** | Central team bottleneck | Frontend teams own |
| **Performance** | May overfetch | Exactly what's needed |

### BFF Implementation

```javascript
// Mobile BFF - optimized for mobile screens
app.get('/api/home-feed', async (req, res) => {
    // Aggregate multiple services into one response
    const [user, recentOrders, recommendations] = await Promise.all([
        userService.getBasicProfile(req.userId),  // Minimal fields
        orderService.getRecent(req.userId, 3),    // Only last 3
        productService.getRecommendations(req.userId, 5)  // Top 5
    ]);
    
    // Mobile-optimized response
    res.json({
        user: { name: user.name, avatarUrl: user.avatarUrl },
        orders: recentOrders.map(o => ({
            id: o.id,
            status: o.status,
            thumbnail: o.items[0].thumbnailUrl
        })),
        recommendations: recommendations.map(p => ({
            id: p.id,
            name: p.name,
            price: p.price,
            imageUrl: p.mobileImageUrl  // Mobile-sized image
        }))
    });
});
```

---

## Service Mesh

### Architecture

```mermaid
flowchart TB
    subgraph Pod1["Pod: User Service"]
        US[User Service]
        Proxy1[Envoy Sidecar]
        US <--> Proxy1
    end
    
    subgraph Pod2["Pod: Order Service"]
        OS[Order Service]
        Proxy2[Envoy Sidecar]
        OS <--> Proxy2
    end
    
    subgraph ControlPlane["Control Plane (Istio)"]
        Pilot[Pilot<br/>Config]
        Citadel[Citadel<br/>Certs]
        Galley[Galley<br/>Validation]
    end
    
    Proxy1 <-->|"mTLS"| Proxy2
    
    ControlPlane -->|"Push config"| Proxy1
    ControlPlane -->|"Push config"| Proxy2
```

### Service Mesh vs API Gateway

| Aspect | API Gateway | Service Mesh |
|--------|-------------|--------------|
| **Position** | Edge (north-south traffic) | Internal (east-west traffic) |
| **Focus** | External API management | Service-to-service communication |
| **Auth** | External client auth | mTLS between services |
| **Features** | Rate limiting, API versioning | Observability, traffic shifting |
| **Implementation** | Centralized | Distributed (sidecars) |
| **Examples** | Kong, AWS API Gateway | Istio, Linkerd |

### When to Use Both

```mermaid
flowchart LR
    subgraph External["External Traffic"]
        Client[External Clients]
        Gateway[API Gateway]
    end
    
    subgraph Internal["Internal Traffic (Service Mesh)"]
        S1[Service A]
        S2[Service B]
        S3[Service C]
    end
    
    Client -->|"North-South"| Gateway
    Gateway --> S1
    S1 <-->|"East-West<br/>mTLS via mesh"| S2
    S2 <-->|"East-West<br/>mTLS via mesh"| S3
```

---

## SSL/TLS Termination

### Termination Points

```mermaid
flowchart LR
    subgraph Options["TLS Termination Options"]
        Edge[Edge Termination]
        Passthrough[TLS Passthrough]
        ReEncrypt[Re-encryption]
    end
```

```mermaid
flowchart LR
    subgraph Edge["Edge Termination"]
        C1[Client] -->|"HTTPS"| LB1[Load Balancer]
        LB1 -->|"HTTP"| B1[Backend]
    end
    
    subgraph Passthrough["TLS Passthrough"]
        C2[Client] -->|"HTTPS"| LB2[Load Balancer]
        LB2 -->|"HTTPS"| B2[Backend]
    end
    
    subgraph ReEncrypt["Re-encryption"]
        C3[Client] -->|"HTTPS<br/>(External cert)"| LB3[Load Balancer]
        LB3 -->|"HTTPS<br/>(Internal cert)"| B3[Backend]
    end
```

| Strategy | Performance | Security | Complexity |
|----------|-------------|----------|------------|
| **Edge Termination** | Best (single TLS handshake) | Internal traffic unencrypted | Low |
| **TLS Passthrough** | Moderate (backend handles TLS) | End-to-end encryption | Medium |
| **Re-encryption** | Worst (two TLS handshakes) | End-to-end + gateway inspection | High |

---

## Gateway Patterns

### Request Aggregation (API Composition)

```mermaid
sequenceDiagram
    participant Client
    participant Gateway
    participant UserSvc as User Service
    participant OrderSvc as Order Service
    participant ProductSvc as Product Service
    
    Client->>Gateway: GET /api/dashboard
    
    par Parallel requests
        Gateway->>UserSvc: GET /users/123
        Gateway->>OrderSvc: GET /users/123/orders?limit=5
        Gateway->>ProductSvc: GET /recommendations?user=123
    end
    
    UserSvc->>Gateway: User data
    OrderSvc->>Gateway: Recent orders
    ProductSvc->>Gateway: Recommendations
    
    Gateway->>Gateway: Aggregate responses
    Gateway->>Client: Combined dashboard data
```

### Circuit Breaker at Gateway

```mermaid
stateDiagram-v2
    [*] --> Closed: Normal operation
    
    Closed --> Open: Failure threshold exceeded
    Open --> HalfOpen: After timeout
    HalfOpen --> Closed: Success
    HalfOpen --> Open: Failure
    
    note right of Closed
        Track failure count
        Reset on success
    end note
    
    note right of Open
        Fail fast
        Return cached/fallback
    end note
    
    note right of HalfOpen
        Allow limited traffic
        Test if service recovered
    end note
```

---

## Interview Scenarios

### Scenario 1: Public API for Third-Party Developers

```mermaid
flowchart TB
    subgraph Requirements
        R1[API Key management]
        R2[Rate limiting per key]
        R3[Usage metering]
        R4[Documentation]
    end
    
    subgraph Solution["Recommended: Managed API Gateway"]
        S1[Kong / AWS API Gateway]
        S2[Developer portal]
        S3[Analytics dashboard]
    end
    
    Requirements --> Solution
```

**Talking Points**:
- API key authentication and management
- Per-key rate limiting and quotas
- Usage tracking for billing
- Developer portal for documentation
- Versioning strategy (URL path versioning)

### Scenario 2: Microservices with Multiple Clients

```mermaid
flowchart TB
    subgraph Clients
        Web[Web App]
        iOS[iOS App]
        Android[Android App]
    end
    
    subgraph Solution["Recommended: BFF Pattern"]
        WebBFF[Web BFF]
        MobileBFF[Mobile BFF]
    end
    
    subgraph Services
        MS1[Service 1]
        MS2[Service 2]
        MS3[Service 3]
    end
    
    Web --> WebBFF
    iOS --> MobileBFF
    Android --> MobileBFF
    
    WebBFF --> MS1
    WebBFF --> MS2
    MobileBFF --> MS1
    MobileBFF --> MS3
```

**Talking Points**:
- Different data needs per client type
- Response aggregation at BFF layer
- Frontend teams can own their BFF
- Consider GraphQL as alternative

### Scenario 3: Zero-Trust Internal Communication

```mermaid
flowchart TB
    subgraph Solution["Recommended: Service Mesh"]
        direction TB
        Mesh[Istio / Linkerd]
        mTLS[mTLS everywhere]
        Policy[Policy enforcement]
        Observe[Distributed tracing]
    end
```

**Talking Points**:
- Service mesh for east-west traffic
- mTLS for service-to-service encryption
- Policy-based access control
- Observability (traces, metrics)
- API Gateway for north-south traffic

---

## Quick Reference Card

```
┌─────────────────────────────────────────────────────────────────┐
│                SERVICE EXPOSURE CHEAT SHEET                      │
├─────────────────────────────────────────────────────────────────┤
│ REVERSE PROXY (Nginx, HAProxy):                                 │
│   • Load balancing, SSL termination, caching                    │
│   • Simple routing, no API management                           │
├─────────────────────────────────────────────────────────────────┤
│ API GATEWAY (Kong, AWS API Gateway):                            │
│   • Authentication, rate limiting, routing                      │
│   • Request/response transformation                             │
│   • Developer portal, analytics                                 │
│   • Use for: External APIs, public-facing services              │
├─────────────────────────────────────────────────────────────────┤
│ BFF (Backend for Frontend):                                     │
│   • Client-specific aggregation and formatting                  │
│   • Owned by frontend teams                                     │
│   • Use for: Multiple client types with different needs         │
├─────────────────────────────────────────────────────────────────┤
│ SERVICE MESH (Istio, Linkerd):                                  │
│   • Service-to-service communication                            │
│   • mTLS, observability, traffic control                        │
│   • Use for: Internal microservices, zero-trust                 │
├─────────────────────────────────────────────────────────────────┤
│ DECISION HEURISTIC:                                             │
│   • External API consumers? → API Gateway                       │
│   • Multiple client types? → BFF + Gateway                      │
│   • Internal microservices? → Service Mesh                      │
│   • Simple routing/LB? → Reverse Proxy                          │
└─────────────────────────────────────────────────────────────────┘
```

---

## Practice Questions

1. Design the gateway architecture for a multi-tenant SaaS platform with per-tenant rate limits.
2. How would you implement canary deployments using an API gateway?
3. Compare the trade-offs of a single API gateway vs. BFF pattern for a company with web, iOS, and Android apps.
4. Explain how you would handle authentication for both external partners and internal services.
5. Design a system where the API gateway needs to aggregate responses from 5 microservices within 100ms SLA.
