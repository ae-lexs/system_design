# 02 — Networking & Communication

> Understanding how components talk to each other is fundamental to distributed systems design.

**Prerequisites:** [01 — Foundational Concepts](./01-FOUNDATIONAL-CONCEPTS.md)  
**Builds toward:** [05 — Distributed Patterns](./05-DISTRIBUTED-PATTERNS.md), [07 — Load Balancing](./07-LOAD-BALANCING-SCALING.md)  
**Estimated study time:** 2-3 hours

---

## Chapter Overview

This module covers the communication layer of distributed systems—from basic protocols to real-time communication patterns.

```mermaid
graph TB
    subgraph "Request-Response"
        HTTP[HTTP/HTTPS]
        DNS[DNS Resolution]
    end
    
    subgraph "Intermediaries"
        FP[Forward Proxy]
        RP[Reverse Proxy]
    end
    
    subgraph "Real-Time"
        WS[WebSockets]
        SSE[Server-Sent Events]
        LP[Long Polling]
    end
    
    Client --> DNS
    DNS --> HTTP
    HTTP --> FP
    FP --> RP
    RP --> Server
    
    Client <--> WS
    Server --> SSE
    Client --> LP
```

---

## 1. HTTP and HTTPS

### HTTP Fundamentals

HTTP (Hypertext Transfer Protocol) is the foundation of web communication—a stateless, request-response protocol.

| Aspect | Description |
|--------|-------------|
| **Model** | Request-Response |
| **State** | Stateless (each request independent) |
| **Connection** | TCP-based |
| **Default Port** | 80 (HTTP), 443 (HTTPS) |

### HTTP Methods and Their Semantics

| Method | Idempotent | Safe | Use Case |
|--------|------------|------|----------|
| GET | ✓ | ✓ | Retrieve resource |
| POST | ✗ | ✗ | Create resource, submit data |
| PUT | ✓ | ✗ | Replace resource entirely |
| PATCH | ✗ | ✗ | Partial update |
| DELETE | ✓ | ✗ | Remove resource |
| HEAD | ✓ | ✓ | Get headers only |
| OPTIONS | ✓ | ✓ | Get allowed methods |

**Idempotent:** Multiple identical requests have same effect as single request  
**Safe:** Does not modify server state

### HTTP Status Codes

| Range | Category | Key Codes |
|-------|----------|-----------|
| 1xx | Informational | 101 Switching Protocols |
| 2xx | Success | 200 OK, 201 Created, 204 No Content |
| 3xx | Redirection | 301 Permanent, 302 Found, 304 Not Modified |
| 4xx | Client Error | 400 Bad Request, 401 Unauthorized, 403 Forbidden, 404 Not Found, 429 Too Many Requests |
| 5xx | Server Error | 500 Internal Error, 502 Bad Gateway, 503 Service Unavailable, 504 Gateway Timeout |

### HTTPS

HTTPS = HTTP + TLS (Transport Layer Security)

```mermaid
sequenceDiagram
    participant C as Client
    participant S as Server
    
    C->>S: ClientHello (supported ciphers)
    S->>C: ServerHello (chosen cipher) + Certificate
    C->>C: Verify certificate
    C->>S: Key exchange (encrypted with server's public key)
    S->>C: Finished (encrypted)
    C->>S: Application data (encrypted)
    S->>C: Application data (encrypted)
```

**Why HTTPS matters:**
- **Confidentiality:** Data encrypted in transit
- **Integrity:** Tampering detected
- **Authentication:** Server identity verified

### HTTP Evolution

| Version | Key Features | Connection Model |
|---------|--------------|------------------|
| HTTP/1.0 | Basic request-response | New connection per request |
| HTTP/1.1 | Keep-alive, pipelining, chunked transfer | Persistent connections |
| HTTP/2 | Multiplexing, header compression, server push | Single connection, multiple streams |
| HTTP/3 | QUIC (UDP-based), faster handshake | Built-in encryption, 0-RTT |

```mermaid
graph LR
    subgraph "HTTP/1.1"
        A1[Request 1] --> R1[Response 1]
        R1 --> A2[Request 2]
        A2 --> R2[Response 2]
    end
    
    subgraph "HTTP/2 Multiplexing"
        B1[Request 1]
        B2[Request 2]
        B3[Request 3]
        B1 --> Stream
        B2 --> Stream
        B3 --> Stream
        Stream --> C1[Response 1]
        Stream --> C2[Response 2]
        Stream --> C3[Response 3]
    end
```

---

## 2. DNS (Domain Name System)

### What DNS Does
DNS translates human-readable domain names to IP addresses.

```mermaid
sequenceDiagram
    participant B as Browser
    participant R as Resolver
    participant Root as Root DNS
    participant TLD as TLD DNS
    participant Auth as Authoritative DNS
    
    B->>R: What is example.com?
    R->>Root: What is example.com?
    Root->>R: Ask .com TLD server
    R->>TLD: What is example.com?
    TLD->>R: Ask ns1.example.com
    R->>Auth: What is example.com?
    Auth->>R: 93.184.216.34
    R->>B: 93.184.216.34
```

### DNS Record Types

| Type | Purpose | Example |
|------|---------|---------|
| **A** | IPv4 address | example.com → 93.184.216.34 |
| **AAAA** | IPv6 address | example.com → 2606:2800:220:1:... |
| **CNAME** | Alias to another domain | www.example.com → example.com |
| **MX** | Mail server | example.com → mail.example.com |
| **NS** | Nameserver | example.com → ns1.example.com |
| **TXT** | Text records | SPF, DKIM, verification |

### DNS Caching Hierarchy

```mermaid
graph TD
    B[Browser Cache] --> OS[OS Cache]
    OS --> R[Resolver Cache<br/>ISP/Corporate]
    R --> Root[Root DNS]
    R --> TLD[TLD DNS]
    R --> Auth[Authoritative DNS]
    
    B -.->|TTL: seconds-minutes| B
    R -.->|TTL: hours-days| R
```

**TTL (Time To Live):** How long to cache before re-querying

### DNS for Load Balancing

DNS can distribute traffic by returning different IPs:

| Technique | How It Works | Trade-offs |
|-----------|--------------|------------|
| **Round Robin** | Return IPs in rotating order | Simple, but ignores server health |
| **Weighted** | Return IPs based on capacity weights | Better distribution, manual config |
| **Geo-based** | Return nearest IP by location | Lower latency, requires geo database |
| **Latency-based** | Return fastest-responding IP | Best performance, complex to implement |

**Limitation:** DNS caching means changes propagate slowly (TTL-dependent).

---

## 3. Proxies

### Forward Proxy

A forward proxy sits between clients and the internet, acting on behalf of clients.

```mermaid
graph LR
    C1[Client 1] --> FP[Forward Proxy]
    C2[Client 2] --> FP
    C3[Client 3] --> FP
    FP --> I[Internet/Server]
```

**Use cases:**
- **Anonymity:** Hide client IP from servers
- **Caching:** Cache responses to reduce bandwidth
- **Filtering:** Block access to certain sites
- **Logging:** Audit client requests

### Reverse Proxy

A reverse proxy sits in front of servers, acting on behalf of servers.

```mermaid
graph LR
    C[Client] --> RP[Reverse Proxy]
    RP --> S1[Server 1]
    RP --> S2[Server 2]
    RP --> S3[Server 3]
```

**Use cases:**
- **Load balancing:** Distribute requests across servers
- **SSL termination:** Handle encryption/decryption
- **Caching:** Cache responses for static content
- **Compression:** Compress responses
- **Security:** Hide server topology, WAF

### Forward vs Reverse Proxy Comparison

| Aspect | Forward Proxy | Reverse Proxy |
|--------|---------------|---------------|
| **Protects** | Clients | Servers |
| **Hides** | Client identity | Server identity |
| **Typical Location** | Client network | Server network |
| **Configured by** | Clients | Server operators |
| **Examples** | Squid, corporate proxies | Nginx, HAProxy, Cloudflare |

### Collapsed Forwarding

An optimization where a proxy combines identical concurrent requests:

```mermaid
sequenceDiagram
    participant C1 as Client 1
    participant C2 as Client 2
    participant P as Proxy
    participant S as Server
    
    C1->>P: GET /resource
    C2->>P: GET /resource
    Note over P: Requests combined
    P->>S: GET /resource (single request)
    S->>P: Response
    P->>C1: Response
    P->>C2: Response (same)
```

---

## 4. Real-Time Communication Patterns

### The Problem with HTTP for Real-Time

HTTP is request-response: clients must ask for data. For real-time updates, we need the server to push data.

| Pattern | Direction | Connection | Latency | Overhead |
|---------|-----------|------------|---------|----------|
| Polling | Client → Server | Multiple | High | High |
| Long Polling | Client → Server | Held open | Medium | Medium |
| SSE | Server → Client | Persistent | Low | Low |
| WebSocket | Bidirectional | Persistent | Very Low | Very Low |

### Regular Polling

Client repeatedly requests updates at fixed intervals.

```mermaid
sequenceDiagram
    participant C as Client
    participant S as Server
    
    loop Every N seconds
        C->>S: Any updates?
        S->>C: No / Here's data
    end
```

**Pros:** Simple, works everywhere  
**Cons:** High overhead, delayed updates, wasted requests

### Long Polling

Client makes request; server holds it open until data available or timeout.

```mermaid
sequenceDiagram
    participant C as Client
    participant S as Server
    
    C->>S: Any updates?
    Note over S: Holds connection...
    Note over S: ...until data ready
    S->>C: Here's the update
    C->>S: Any more updates?
    Note over S: Holds again...
```

**Pros:** Lower latency than polling, works through most firewalls  
**Cons:** Server resource usage (held connections), not truly real-time

### Server-Sent Events (SSE)

Server pushes updates over a persistent HTTP connection (text/event-stream).

```mermaid
sequenceDiagram
    participant C as Client
    participant S as Server
    
    C->>S: Open SSE connection
    S-->>C: Event: update 1
    S-->>C: Event: update 2
    S-->>C: Event: update 3
    Note over C,S: Connection stays open
```

**Pros:** Simple API, automatic reconnection, works with HTTP/2  
**Cons:** Unidirectional (server to client only), limited browser connections per domain

**Best for:** News feeds, stock tickers, notifications—server-to-client only

### WebSockets

Full-duplex, bidirectional communication over a single TCP connection.

```mermaid
sequenceDiagram
    participant C as Client
    participant S as Server
    
    C->>S: HTTP Upgrade Request
    S->>C: 101 Switching Protocols
    Note over C,S: WebSocket connection established
    C-->>S: Message
    S-->>C: Message
    C-->>S: Message
    S-->>C: Message
```

**Handshake:**
1. Client sends HTTP request with `Upgrade: websocket`
2. Server responds with `101 Switching Protocols`
3. Connection upgraded to WebSocket protocol

**Pros:** True bidirectional, low latency, low overhead after handshake  
**Cons:** Stateful (harder to scale), may need proxy configuration

**Best for:** Chat, gaming, collaborative editing—anything bidirectional

### Real-Time Pattern Decision Tree

```mermaid
flowchart TD
    A[Need real-time updates?] --> B{Direction?}
    
    B -->|Server to Client only| C{Update frequency?}
    B -->|Bidirectional| D[WebSocket]
    
    C -->|Seconds apart| E[Long Polling]
    C -->|Sub-second| F[SSE]
    
    D --> G{Infrastructure supports?}
    G -->|Yes| H[WebSocket]
    G -->|No| I[Long Polling fallback]
    
    E --> J{Can you implement SSE?}
    J -->|Yes| F
    J -->|No| E
```

### Comparison Summary

| Feature | Polling | Long Polling | SSE | WebSocket |
|---------|---------|--------------|-----|-----------|
| Latency | High | Medium | Low | Very Low |
| Server load | High | Medium | Low | Low |
| Bidirectional | No | No | No | Yes |
| Browser support | Universal | Universal | Good | Good |
| HTTP/2 compatible | Yes | Yes | Yes | No (separate) |
| Auto-reconnect | Manual | Manual | Built-in | Manual |

---

## 5. API Design Considerations

### REST API Best Practices

| Principle | Description | Example |
|-----------|-------------|---------|
| **Resources as nouns** | URLs represent things, not actions | `/users`, not `/getUsers` |
| **HTTP verbs for actions** | GET, POST, PUT, DELETE | `DELETE /users/123` |
| **Plural resource names** | Consistency | `/users`, not `/user` |
| **Nested resources** | Show relationships | `/users/123/posts` |
| **Query params for filtering** | Don't embed in URL | `/users?status=active` |
| **Versioning** | API evolution | `/v1/users`, `/v2/users` |

### API Gateway Functions

An API Gateway is a single entry point that handles cross-cutting concerns:

```mermaid
graph LR
    C[Clients] --> AG[API Gateway]
    
    AG --> Auth[Authentication]
    AG --> RL[Rate Limiting]
    AG --> Cache[Caching]
    AG --> Route[Routing]
    AG --> Trans[Transformation]
    AG --> Log[Logging]
    
    Route --> S1[Service A]
    Route --> S2[Service B]
    Route --> S3[Service C]
```

**Key responsibilities:**
- **Authentication/Authorization:** Verify identity, check permissions
- **Rate limiting:** Prevent abuse, ensure fair usage
- **Request routing:** Direct to appropriate service
- **Protocol translation:** HTTP ↔ gRPC, REST ↔ GraphQL
- **Response aggregation:** Combine multiple service responses
- **Caching:** Reduce backend load
- **SSL termination:** Handle encryption at the edge

### API Gateway vs Load Balancer

| Aspect | API Gateway | Load Balancer |
|--------|-------------|---------------|
| **Layer** | Application (L7) | Transport (L4) or Application (L7) |
| **Awareness** | Request content | Connection/packet |
| **Functions** | Auth, rate limit, transform | Traffic distribution |
| **Typical position** | Edge, in front of services | In front of server pools |

---

## 6. Rate Limiting

### Why Rate Limit?
- **Prevent abuse:** Stop malicious users from overwhelming the system
- **Ensure fairness:** Prevent one user from consuming all resources
- **Cost control:** Limit expensive operations
- **Maintain SLAs:** Protect service quality for all users

### Rate Limiting Algorithms

| Algorithm | Description | Pros | Cons |
|-----------|-------------|------|------|
| **Token Bucket** | Tokens added at fixed rate, request costs token | Allows bursts, smooth | Memory per user |
| **Leaky Bucket** | Requests processed at fixed rate, excess queued | Consistent output | No burst handling |
| **Fixed Window** | Count requests in fixed time windows | Simple | Burst at window edges |
| **Sliding Window** | Rolling time window | Smooth, no edge bursts | More computation |

### Token Bucket Visualization

```mermaid
graph TD
    subgraph "Token Bucket"
        B[Bucket<br/>Capacity: 10]
        T[Tokens added: 1/sec]
    end
    
    R[Request arrives] --> Check{Tokens > 0?}
    Check -->|Yes| Allow[Allow request<br/>Tokens -= 1]
    Check -->|No| Reject[Reject: 429]
    
    T --> B
```

**Parameters:**
- **Bucket size:** Max burst capacity
- **Refill rate:** Sustained request rate

---

## 7. Chapter Summary

### Key Concepts

| Concept | One-Line Definition |
|---------|---------------------|
| HTTP | Stateless request-response protocol |
| HTTPS | HTTP with TLS encryption |
| DNS | Translates domain names to IP addresses |
| Forward Proxy | Acts on behalf of clients |
| Reverse Proxy | Acts on behalf of servers |
| Long Polling | Server holds request until data ready |
| SSE | Server pushes events over HTTP |
| WebSocket | Bidirectional persistent connection |

### Decision Framework

```
For client-server communication:
├── One-time request? → HTTP
├── Real-time server updates only? → SSE
├── Real-time bidirectional? → WebSocket
└── Must work everywhere? → Long Polling fallback
```

### Interview Checklist

- [ ] Explain HTTP method semantics (idempotent, safe)
- [ ] Describe DNS resolution flow
- [ ] Compare forward vs reverse proxy
- [ ] Choose between SSE, WebSocket, and Long Polling
- [ ] Explain rate limiting algorithms and trade-offs

---

## Navigation

**Previous:** [01 — Foundational Concepts](./01-FOUNDATIONAL-CONCEPTS.md)  
**Next:** [03 — Data Management](./03-DATA-MANAGEMENT.md)  
**Index:** [00 — Handbook Index](./00-INDEX.md)
