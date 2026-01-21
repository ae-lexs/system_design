# 08 - API Design: REST vs RPC

## Overview

API design determines how clients and services communicate. The choice between REST and RPC paradigms (including gRPC and GraphQL) affects developer experience, performance, evolvability, and operational complexity. This document provides a systematic framework for evaluating API styles and designing APIs that scale.

---

## Core Mental Model

```mermaid
flowchart TD
    subgraph Paradigms["API Paradigms"]
        REST[REST<br/>Resource-Oriented]
        RPC[RPC<br/>Action-Oriented]
        GraphQL[GraphQL<br/>Query-Oriented]
    end
    
    subgraph Questions["Design Questions"]
        Q1[What entities exist?]
        Q2[What operations are allowed?]
        Q3[How is data serialized?]
        Q4[How do clients discover capabilities?]
    end
    
    REST -->|"Resources + HTTP verbs"| Q1
    RPC -->|"Procedures + Parameters"| Q2
    GraphQL -->|"Schema + Queries"| Q1
    GraphQL -->|"Mutations"| Q2
```

**Key Insight**: REST models "things" (nouns), RPC models "actions" (verbs), GraphQL lets clients specify exactly what they need.

---

## REST (Representational State Transfer)

### Core Principles

```mermaid
flowchart LR
    subgraph REST_Principles["REST Constraints"]
        CS[Client-Server]
        SL[Stateless]
        Cache[Cacheable]
        Uniform[Uniform Interface]
        Layer[Layered System]
        COD[Code on Demand<br/>Optional]
    end
```

### Resource-Oriented Design

```mermaid
flowchart TB
    subgraph Resources["Resource Hierarchy"]
        Users["/users"]
        User["/users/{id}"]
        Orders["/users/{id}/orders"]
        Order["/users/{id}/orders/{orderId}"]
        Items["/users/{id}/orders/{orderId}/items"]
    end
    
    Users -->|"GET: List all"| User
    User -->|"GET: Get one"| Orders
    Orders -->|"GET: List user's orders"| Order
    Order -->|"GET: Get specific order"| Items
```

### HTTP Methods Mapping

| Method | Operation | Idempotent | Safe | Example |
|--------|-----------|------------|------|---------|
| GET | Read | Yes | Yes | `GET /users/123` |
| POST | Create | No | No | `POST /users` |
| PUT | Replace | Yes | No | `PUT /users/123` |
| PATCH | Partial Update | No* | No | `PATCH /users/123` |
| DELETE | Remove | Yes | No | `DELETE /users/123` |

*PATCH can be idempotent if using JSON Patch with specific operations.

### Status Codes

```mermaid
flowchart LR
    subgraph Success["2xx Success"]
        S200["200 OK"]
        S201["201 Created"]
        S204["204 No Content"]
    end
    
    subgraph Redirect["3xx Redirection"]
        R301["301 Moved"]
        R304["304 Not Modified"]
    end
    
    subgraph ClientErr["4xx Client Error"]
        E400["400 Bad Request"]
        E401["401 Unauthorized"]
        E403["403 Forbidden"]
        E404["404 Not Found"]
        E409["409 Conflict"]
        E429["429 Too Many Requests"]
    end
    
    subgraph ServerErr["5xx Server Error"]
        E500["500 Internal Error"]
        E502["502 Bad Gateway"]
        E503["503 Unavailable"]
    end
```

### HATEOAS (Hypermedia)

```json
{
  "id": 123,
  "name": "John Doe",
  "email": "john@example.com",
  "_links": {
    "self": { "href": "/users/123" },
    "orders": { "href": "/users/123/orders" },
    "update": { "href": "/users/123", "method": "PUT" },
    "delete": { "href": "/users/123", "method": "DELETE" }
  }
}
```

**Interview Point**: HATEOAS makes APIs self-documenting and evolvable, but is rarely fully implemented in practice.

---

## REST Best Practices

### URL Design

```
✅ Good:
GET  /users                     # List users
GET  /users/123                 # Get user 123
GET  /users/123/orders          # Get user's orders
POST /users/123/orders          # Create order for user

❌ Bad:
GET  /getUsers                  # Verb in URL
GET  /user/123                  # Inconsistent plural/singular
POST /users/123/createOrder     # Action in URL
GET  /users/123/orders/active   # Filter as path segment
```

### Filtering, Pagination, Sorting

```
# Filtering
GET /users?status=active&role=admin

# Pagination (offset-based)
GET /users?offset=20&limit=10

# Pagination (cursor-based - preferred for large datasets)
GET /users?cursor=eyJpZCI6MTIzfQ&limit=10

# Sorting
GET /users?sort=-created_at,name  # Descending created_at, ascending name
```

### Cursor vs Offset Pagination

```mermaid
flowchart LR
    subgraph Offset["Offset Pagination"]
        O1["Page 1: OFFSET 0"]
        O2["Page 2: OFFSET 10"]
        O3["Page 3: OFFSET 20"]
        O1 -->|"Insert happens"| O2
        O2 -->|"⚠️ Skipped or<br/>duplicate items"| O3
    end
    
    subgraph Cursor["Cursor Pagination"]
        C1["First: after=null"]
        C2["Next: after=id_10"]
        C3["Next: after=id_20"]
        C1 -->|"Insert happens"| C2
        C2 -->|"✓ Stable<br/>iteration"| C3
    end
```

| Aspect | Offset | Cursor |
|--------|--------|--------|
| Random access | Yes (`?page=50`) | No |
| Performance | O(n) skip | O(1) seek |
| Consistency | Unstable with mutations | Stable |
| Implementation | Simple | Complex |

---

## RPC (Remote Procedure Call)

### Paradigm

```mermaid
sequenceDiagram
    participant Client
    participant Stub as Client Stub
    participant Network
    participant Server as Server Stub
    participant Service
    
    Client->>Stub: createUser(name, email)
    Stub->>Stub: Serialize request
    Stub->>Network: Send bytes
    Network->>Server: Deliver bytes
    Server->>Server: Deserialize request
    Server->>Service: createUser(name, email)
    Service->>Server: Return User object
    Server->>Server: Serialize response
    Server->>Network: Send bytes
    Network->>Stub: Deliver bytes
    Stub->>Stub: Deserialize response
    Stub->>Client: Return User object
```

### gRPC Architecture

```mermaid
flowchart TB
    subgraph Definition["Service Definition (Proto)"]
        Proto["user.proto"]
    end
    
    subgraph Generated["Generated Code"]
        ClientStub["Client Stub<br/>(Python, Go, Java...)"]
        ServerStub["Server Interface<br/>(Abstract class)"]
    end
    
    subgraph Runtime["Runtime"]
        Client[Client App]
        Server[Server App]
        HTTP2["HTTP/2 Transport"]
    end
    
    Proto -->|protoc| ClientStub
    Proto -->|protoc| ServerStub
    Client --> ClientStub
    ServerStub --> Server
    ClientStub <-->|Binary Protocol| HTTP2
    HTTP2 <-->|Binary Protocol| Server
```

### Protocol Buffers (Protobuf)

```protobuf
syntax = "proto3";

package user;

service UserService {
  rpc CreateUser(CreateUserRequest) returns (User);
  rpc GetUser(GetUserRequest) returns (User);
  rpc ListUsers(ListUsersRequest) returns (stream User);  // Server streaming
  rpc UpdateUsers(stream User) returns (UpdateResponse);   // Client streaming
  rpc Chat(stream Message) returns (stream Message);       // Bidirectional
}

message User {
  int64 id = 1;
  string name = 2;
  string email = 3;
  repeated string roles = 4;
  google.protobuf.Timestamp created_at = 5;
}

message CreateUserRequest {
  string name = 1;
  string email = 2;
}

message GetUserRequest {
  int64 id = 1;
}
```

### gRPC Streaming Patterns

```mermaid
flowchart TB
    subgraph Unary["Unary RPC"]
        U1[Request] --> U2[Response]
    end
    
    subgraph ServerStream["Server Streaming"]
        SS1[Request] --> SS2[Response 1]
        SS1 --> SS3[Response 2]
        SS1 --> SS4[Response N]
    end
    
    subgraph ClientStream["Client Streaming"]
        CS1[Request 1] --> CS4[Response]
        CS2[Request 2] --> CS4
        CS3[Request N] --> CS4
    end
    
    subgraph BiDi["Bidirectional Streaming"]
        BD1[Request 1] <--> BD4[Response 1]
        BD2[Request 2] <--> BD5[Response 2]
        BD3[Request N] <--> BD6[Response N]
    end
```

| Pattern | Use Case |
|---------|----------|
| Unary | Simple request-response |
| Server Streaming | Large result sets, real-time feeds |
| Client Streaming | File upload, aggregation |
| Bidirectional | Chat, gaming, collaborative editing |

---

## GraphQL

### Architecture

```mermaid
flowchart TB
    subgraph Clients["Clients"]
        Web[Web App]
        Mobile[Mobile App]
        CLI[CLI Tool]
    end
    
    subgraph GraphQL["GraphQL Server"]
        Schema[Schema Definition]
        Resolver[Resolvers]
        DataLoader[DataLoader<br/>Batching]
    end
    
    subgraph Backend["Backend Services"]
        UserSvc[User Service]
        OrderSvc[Order Service]
        ProductSvc[Product Service]
    end
    
    Web -->|"query { user { orders } }"| GraphQL
    Mobile -->|"query { user { name } }"| GraphQL
    
    Resolver --> UserSvc
    Resolver --> OrderSvc
    Resolver --> ProductSvc
```

### Schema Definition

```graphql
type Query {
  user(id: ID!): User
  users(filter: UserFilter, first: Int, after: String): UserConnection!
}

type Mutation {
  createUser(input: CreateUserInput!): User!
  updateUser(id: ID!, input: UpdateUserInput!): User!
  deleteUser(id: ID!): Boolean!
}

type Subscription {
  userCreated: User!
  orderStatusChanged(userId: ID!): Order!
}

type User {
  id: ID!
  name: String!
  email: String!
  orders(first: Int): [Order!]!
  createdAt: DateTime!
}

type Order {
  id: ID!
  user: User!
  items: [OrderItem!]!
  total: Money!
  status: OrderStatus!
}

input CreateUserInput {
  name: String!
  email: String!
}

type UserConnection {
  edges: [UserEdge!]!
  pageInfo: PageInfo!
}
```

### Query Examples

```graphql
# Fetch exactly what's needed
query GetUserWithOrders {
  user(id: "123") {
    name
    email
    orders(first: 5) {
      id
      total {
        amount
        currency
      }
      status
    }
  }
}

# Response matches query shape exactly
{
  "data": {
    "user": {
      "name": "John Doe",
      "email": "john@example.com",
      "orders": [
        {
          "id": "order-1",
          "total": { "amount": 99.99, "currency": "USD" },
          "status": "DELIVERED"
        }
      ]
    }
  }
}
```

### N+1 Problem and DataLoader

```mermaid
sequenceDiagram
    participant Client
    participant GraphQL
    participant DataLoader
    participant UserDB
    participant OrderDB
    
    Client->>GraphQL: query { orders { user { name } } }
    GraphQL->>OrderDB: SELECT * FROM orders (1 query)
    
    Note over GraphQL: Without DataLoader: N queries
    loop For each order
        GraphQL->>UserDB: SELECT * FROM users WHERE id = ?
    end
    
    Note over GraphQL,DataLoader: With DataLoader: 1 batched query
    GraphQL->>DataLoader: Load user 1, 2, 3, 4, 5
    DataLoader->>UserDB: SELECT * FROM users WHERE id IN (1,2,3,4,5)
    DataLoader->>GraphQL: Return all users
```

---

## Comparison Matrix

```mermaid
flowchart TD
    subgraph Decision["When to Use What"]
        Q1{Client<br/>diversity?}
        Q1 -->|"High (web, mobile,<br/>third-party)"| REST
        Q1 -->|"Internal<br/>microservices"| RPC
        Q1 -->|"Complex, nested<br/>data needs"| GraphQL
        
        Q2{Performance<br/>critical?}
        Q2 -->|"Yes, binary"| gRPC
        Q2 -->|"OK with JSON"| REST_OR_GQL[REST or GraphQL]
        
        Q3{Real-time<br/>streaming?}
        Q3 -->|"Yes"| gRPC_WS[gRPC or WebSocket]
        Q3 -->|"No"| REST2[REST]
    end
```

| Aspect | REST | gRPC | GraphQL |
|--------|------|------|---------|
| **Protocol** | HTTP/1.1 or HTTP/2 | HTTP/2 | HTTP (typically 1.1) |
| **Serialization** | JSON (text) | Protobuf (binary) | JSON (text) |
| **Contract** | OpenAPI/Swagger | Proto files | GraphQL Schema |
| **Caching** | HTTP caching built-in | Custom | Requires effort |
| **Browser support** | Native | Requires gRPC-Web | Native |
| **Streaming** | Limited (SSE) | Native | Subscriptions |
| **Discoverability** | HATEOAS | Reflection | Introspection |
| **Learning curve** | Low | Medium | Medium-High |
| **Overfetching** | Common | Unlikely | Solved |
| **Underfetching** | Common | Unlikely | Solved |

---

## API Versioning

```mermaid
flowchart LR
    subgraph Strategies["Versioning Strategies"]
        URL["URL Path<br/>/api/v1/users"]
        Header["Header<br/>Accept: application/vnd.api+json;v=1"]
        Query["Query Param<br/>/users?version=1"]
        None["No Version<br/>(Evolvable design)"]
    end
```

### Strategy Comparison

| Strategy | Pros | Cons |
|----------|------|------|
| **URL Path** | Clear, cacheable, simple | Breaks URLs on version change |
| **Header** | Clean URLs | Hidden, harder to test |
| **Query Param** | Easy to switch | Pollutes query string |
| **Evolvable** | No version management | Requires discipline |

### Backward-Compatible Changes

```
✅ Safe Changes (No version bump):
- Add new optional field to response
- Add new endpoint
- Add new optional query parameter
- Add new enum value (if clients handle unknown)

❌ Breaking Changes (Require version):
- Remove or rename field
- Change field type
- Change URL structure
- Change authentication mechanism
- Change error format
```

### Evolution Pattern

```mermaid
sequenceDiagram
    participant Old as Old Client
    participant New as New Client
    participant API
    
    Note over API: Phase 1: Add new field alongside old
    Old->>API: GET /users/123
    API->>Old: { name: "...", fullName: "..." }
    
    New->>API: GET /users/123
    API->>New: { name: "...", fullName: "..." }
    
    Note over API: Phase 2: Deprecation notice
    API->>Old: { name: "...", fullName: "..." }<br/>Header: Deprecation: name
    
    Note over API: Phase 3: Remove old field
    API->>New: { fullName: "..." }
```

---

## Error Handling

### REST Error Response

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Request validation failed",
    "details": [
      {
        "field": "email",
        "issue": "Invalid email format",
        "value": "not-an-email"
      }
    ],
    "request_id": "req_abc123",
    "documentation_url": "https://api.example.com/docs/errors#VALIDATION_ERROR"
  }
}
```

### gRPC Status Codes

```mermaid
flowchart TB
    subgraph Codes["gRPC Status Codes"]
        OK["OK (0)"]
        CANCELLED["CANCELLED (1)"]
        UNKNOWN["UNKNOWN (2)"]
        INVALID["INVALID_ARGUMENT (3)"]
        NOT_FOUND["NOT_FOUND (5)"]
        EXISTS["ALREADY_EXISTS (6)"]
        PERMISSION["PERMISSION_DENIED (7)"]
        RESOURCE["RESOURCE_EXHAUSTED (8)"]
        UNAUTHENTICATED["UNAUTHENTICATED (16)"]
    end
```

```protobuf
// Rich error details
import "google/rpc/error_details.proto";

rpc CreateUser(CreateUserRequest) returns (User) {
  // On error, return:
  // - status: INVALID_ARGUMENT
  // - details: [BadRequest { field_violations: [...] }]
}
```

### GraphQL Errors

```json
{
  "data": {
    "user": null
  },
  "errors": [
    {
      "message": "User not found",
      "locations": [{ "line": 2, "column": 3 }],
      "path": ["user"],
      "extensions": {
        "code": "NOT_FOUND",
        "timestamp": "2024-01-15T10:30:00Z"
      }
    }
  ]
}
```

---

## Security Considerations

```mermaid
flowchart TB
    subgraph Auth["Authentication"]
        JWT[JWT Tokens]
        OAuth[OAuth 2.0]
        API_Key[API Keys]
        mTLS[Mutual TLS]
    end
    
    subgraph Threats["Common Threats"]
        Injection[Injection]
        BrokenAuth[Broken Auth]
        Excessive[Excessive Data Exposure]
        RateLimit[Lack of Rate Limiting]
        BOLA[BOLA/IDOR]
    end
    
    subgraph Mitigations["Mitigations"]
        Input[Input Validation]
        AuthZ[Authorization Checks]
        Filter[Response Filtering]
        Throttle[Rate Limiting]
        AuditLog[Audit Logging]
    end
    
    Threats --> Mitigations
```

### GraphQL-Specific Security

```mermaid
flowchart LR
    subgraph Attacks["GraphQL Attacks"]
        Deep[Deep Query]
        Batch[Batching Abuse]
        Intro[Introspection Leak]
    end
    
    subgraph Defenses["Defenses"]
        Depth[Query Depth Limit]
        Cost[Query Cost Analysis]
        Disable[Disable Introspection<br/>in Production]
        Persist[Persisted Queries]
    end
    
    Deep --> Depth
    Batch --> Cost
    Intro --> Disable
```

```graphql
# Malicious deep query
query EvilQuery {
  user(id: "1") {
    friends {
      friends {
        friends {
          friends {
            friends {
              # ... recursively
            }
          }
        }
      }
    }
  }
}
```

**Defense**: Implement query depth limiting (e.g., max depth = 5) and query cost analysis.

---

## Interview Scenarios

### Scenario 1: Public Developer API

**Context**: Building a public API for third-party developers

```mermaid
flowchart TB
    subgraph Choice["Recommended: REST"]
        R1[Industry Standard]
        R2[HTTP Caching]
        R3[Easy to Test]
        R4[Wide Language Support]
        R5[Self-Documenting with OpenAPI]
    end
```

**Talking Points**:
- REST is universally understood
- HTTP caching reduces load
- OpenAPI enables SDK generation
- Versioning via URL path for clarity

### Scenario 2: Microservices Communication

**Context**: Internal service-to-service communication

```mermaid
flowchart TB
    subgraph Choice["Recommended: gRPC"]
        G1[Binary Protocol - 10x faster]
        G2[Strong Typing - Proto contracts]
        G3[Streaming Support]
        G4[Code Generation]
        G5[Deadline Propagation]
    end
```

**Talking Points**:
- Performance critical for internal calls
- Proto files serve as contracts
- Built-in deadline propagation
- Bidirectional streaming for real-time needs

### Scenario 3: Mobile App with Complex UI

**Context**: Mobile app with varied data needs per screen

```mermaid
flowchart TB
    subgraph Choice["Recommended: GraphQL"]
        GQL1[Fetch exactly what's needed]
        GQL2[Reduce round trips]
        GQL3[Single endpoint]
        GQL4[Strong typing]
    end
```

**Talking Points**:
- Mobile bandwidth constraints → no overfetching
- Different screens need different data shapes
- Single endpoint simplifies client
- Schema serves as documentation

---

## API Design Checklist

```
┌─────────────────────────────────────────────────────────────────┐
│                    API DESIGN CHECKLIST                          │
├─────────────────────────────────────────────────────────────────┤
│ NAMING                                                          │
│ □ Use nouns for REST resources (/users not /getUsers)           │
│ □ Consistent plural/singular (/users, not /user)                │
│ □ Use kebab-case for URLs (/user-profiles)                      │
│ □ Use snake_case for JSON fields (user_name)                    │
├─────────────────────────────────────────────────────────────────┤
│ OPERATIONS                                                       │
│ □ Correct HTTP methods (GET=read, POST=create, etc.)            │
│ □ Idempotent operations where possible                          │
│ □ Appropriate status codes (201 for create, 204 for delete)     │
├─────────────────────────────────────────────────────────────────┤
│ DATA                                                            │
│ □ Pagination for list endpoints                                 │
│ □ Filtering and sorting support                                 │
│ □ Consistent error format                                       │
│ □ Request validation with clear messages                        │
├─────────────────────────────────────────────────────────────────┤
│ SECURITY                                                        │
│ □ Authentication on all endpoints                               │
│ □ Authorization checks (not just authentication)                │
│ □ Rate limiting                                                 │
│ □ Input validation                                              │
│ □ Audit logging                                                 │
├─────────────────────────────────────────────────────────────────┤
│ DOCUMENTATION                                                   │
│ □ OpenAPI/Swagger spec                                          │
│ □ Example requests and responses                                │
│ □ Error code documentation                                      │
│ □ Authentication guide                                          │
└─────────────────────────────────────────────────────────────────┘
```

---

## Quick Reference Card

```
┌─────────────────────────────────────────────────────────────────┐
│                   API DESIGN CHEAT SHEET                         │
├─────────────────────────────────────────────────────────────────┤
│ REST:                                                           │
│   • Resource-oriented (nouns, not verbs)                        │
│   • HTTP verbs: GET, POST, PUT, PATCH, DELETE                   │
│   • Stateless, cacheable                                        │
│   • Best for: Public APIs, web applications                     │
├─────────────────────────────────────────────────────────────────┤
│ gRPC:                                                           │
│   • Action-oriented (procedures)                                │
│   • Binary protocol (Protobuf), HTTP/2                          │
│   • Strong typing, code generation                              │
│   • Best for: Microservices, performance-critical               │
├─────────────────────────────────────────────────────────────────┤
│ GraphQL:                                                        │
│   • Query-oriented (ask for exactly what you need)              │
│   • Single endpoint, client-driven                              │
│   • Solves over/under-fetching                                  │
│   • Best for: Complex UIs, mobile apps                          │
├─────────────────────────────────────────────────────────────────┤
│ DECISION HEURISTIC:                                             │
│   • External developers? → REST                                 │
│   • Internal microservices? → gRPC                              │
│   • Complex client data needs? → GraphQL                        │
│   • Need all three? → BFF pattern (Backend for Frontend)        │
└─────────────────────────────────────────────────────────────────┘
```

---

## Practice Questions

1. Design an API for a file storage service (like Dropbox). Would you use REST, gRPC, or GraphQL? Why?
2. How would you handle API versioning for a mobile app where you can't force users to update?
3. Explain the trade-offs between cursor-based and offset-based pagination.
4. Design the API for a real-time collaborative document editor. What protocols would you use?
5. How would you prevent abuse of a GraphQL API? Discuss query complexity analysis.
