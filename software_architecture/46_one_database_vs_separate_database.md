

# Database-per-Service vs. Shared Database in Microservices

---

## 1. The Core Trade-Off

This is one of the most consequential architectural decisions in microservices. It determines how tightly or loosely your services are coupled at the data layer.

```mermaid
graph TB
    subgraph "Shared Database"
        S1["Order Service"]
        S2["Payment Service"]
        S3["Inventory Service"]
        DB_SHARED[("Single Database<br/>orders + payments + inventory<br/>+ foreign keys across all")]

        S1 --> DB_SHARED
        S2 --> DB_SHARED
        S3 --> DB_SHARED
    end

    subgraph "Database per Service"
        S4["Order Service"]
        S5["Payment Service"]
        S6["Inventory Service"]
        DB1[("Order DB")]
        DB2[("Payment DB")]
        DB3[("Inventory DB")]

        S4 --> DB1
        S5 --> DB2
        S6 --> DB3
    end

    style DB_SHARED fill:#ff6b6b,color:#fff
    style DB1 fill:#4ecdc4,color:#000
    style DB2 fill:#4ecdc4,color:#000
    style DB3 fill:#4ecdc4,color:#000
```

**Short answer: Database-per-service is the recommended pattern for microservices.** A shared database negates the core benefits of the architecture. But the real answer has nuance — here's why, when, and how.

---

## 2. Why Shared Database Is Problematic

### 2.1 Coupling at the Data Layer

```mermaid
graph TD
    subgraph "Shared Database Coupling"
        OS["Order Service<br/>SELECT * FROM orders<br/>JOIN users ON ..."]
        PS["Payment Service<br/>SELECT * FROM orders<br/>WHERE order_id = ..."]
        IS["Inventory Service<br/>UPDATE orders<br/>SET status = 'reserved'"]
        
        DB[("Shared DB")]
        
        OS --> DB
        PS --> DB
        IS --> DB
    end

    PROBLEM["⚠️ All three services know<br/>the 'orders' table schema.<br/>Changing a column breaks them all."]

    style PROBLEM fill:#ff6b6b,color:#fff
```

| Problem | What Happens |
|---|---|
| **Schema coupling** | Changing a column in `orders` requires coordinating changes across Order, Payment, and Inventory services simultaneously |
| **No independent deployment** | Can't deploy Order Service v2 (new schema) without deploying all other services that read `orders` |
| **No independent scaling** | Can't scale the database for one service's read load without affecting others |
| **Ownership ambiguity** | Who owns the `orders` table — Order Service or everyone? |
| **Testing complexity** | Every service's tests need the full shared schema |
| **Technology lock-in** | All services must use the same database engine |

### 2.2 The Independence Test

> **If you can't deploy Service A without coordinating with Service B, they are not independent microservices — they are a distributed monolith.**

A shared database fails this test. Every schema change requires cross-team coordination.

---

## 3. Why Database-per-Service Is Recommended

```mermaid
graph TB
    subgraph "Database per Service"
        OS2["Order Service"] --> ODB[("Order DB<br/>PostgreSQL")]
        PS2["Payment Service"] --> PDB[("Payment DB<br/>PostgreSQL")]
        IS2["Inventory Service"] --> IDB[("Inventory DB<br/>PostgreSQL")]
        SS["Search Service"] --> SDB[("Search Index<br/>Elasticsearch")]
        CS["Cache Service"] --> CDB[("Cache<br/>Redis")]
    end

    OS2 -.->|"API call or event"| PS2
    OS2 -.->|"API call or event"| IS2

    style ODB fill:#4ecdc4,color:#000
    style PDB fill:#4ecdc4,color:#000
    style IDB fill:#4ecdc4,color:#000
    style SDB fill:#ffe66d,color:#000
    style CDB fill:#ff8c42,color:#fff
```

| Benefit | How |
|---|---|
| **Independent deployment** | Change Order DB schema without touching Payment Service |
| **Independent scaling** | Scale Inventory DB reads separately from Order DB writes |
| **Technology freedom** | PostgreSQL for orders, Elasticsearch for search, Redis for cache |
| **Clear ownership** | Team Alpha owns Order Service *and* its database |
| **Fault isolation** | Inventory DB crash doesn't take down Payments |
| **Schema evolution** | Expand-contract migrations within one team, no cross-team coordination |

---

## 4. Comparison

| Dimension | Shared Database | Database per Service |
|---|---|---|
| **Data consistency** | Strong (ACID transactions, foreign keys) | Eventual (sagas, events) |
| **Query flexibility** | Easy JOINs across all data | No cross-service JOINs |
| **Schema changes** | Coordinate across all consuming services | Team-local change |
| **Independent deploy** | ❌ No — schema couples all services | ✅ Yes |
| **Independent scale** | ❌ No — one DB for all load | ✅ Yes — per-service scaling |
| **Polyglot persistence** | ❌ No — one engine for all | ✅ Yes — best tool per use case |
| **Operational complexity** | Low (one DB to manage) | Higher (N databases to manage) |
| **Infrastructure cost** | Lower (one instance) | Higher (N instances — mitigated by managed services) |
| **Data duplication** | None | Some (event-carried state, read models) |
| **Debugging** | Easy (single DB, SQL JOINs) | Harder (distributed traces, multiple data stores) |

---

## 5. How to Handle Cross-Service Data Needs

The main objection to database-per-service: "But I need to JOIN data from multiple services!" Here are the patterns:

### 5.1 API Composition

```mermaid
sequenceDiagram
    participant C as Client / BFF
    participant OS3 as Order Service
    participant US as User Service
    participant PS3 as Payment Service

    C->>OS3: GET /orders/123
    OS3-->>C: {order_id: 123, user_id: "u-1", total: 49.99}
    C->>US: GET /users/u-1
    US-->>C: {name: "Alice", email: "alice@..."}
    C->>PS3: GET /payments?order=123
    PS3-->>C: {status: "paid", method: "visa"}

    Note over C: Compose response in<br/>BFF / API layer
    C->>C: {order: {...}, user: {...}, payment: {...}}
```

**When:** Real-time data needed, low fan-out (2–3 services).

### 5.2 Event-Carried State Transfer

```mermaid
sequenceDiagram
    participant US2 as User Service
    participant K as Kafka
    participant OS4 as Order Service
    participant ODB2 as Order DB

    US2->>K: UserUpdated {id:"u-1", name:"Alice", tier:"premium"}
    K->>OS4: Consume event
    OS4->>ODB2: UPDATE local_users SET name='Alice', tier='premium' WHERE id='u-1'

    Note over OS4: Order Service now has user data locally<br/>No need to call User Service for reads
```

**When:** High-read, low-write data that other services need frequently. Trades consistency (eventual) for performance (no network call).

### 5.3 CQRS — Separate Read Model

```mermaid
graph TB
    subgraph "Write Side (per service)"
        OS5["Order Service"] --> ODB3[("Order DB")]
        PS4["Payment Service"] --> PDB2[("Payment DB")]
        US3["User Service"] --> UDB[("User DB")]
    end

    OS5 -->|"OrderPlaced event"| KAFKA2["Kafka"]
    PS4 -->|"PaymentCompleted event"| KAFKA2
    US3 -->|"UserUpdated event"| KAFKA2

    subgraph "Read Side (denormalized)"
        PROJ["Projector / Consumer"]
        READ_DB[("Read Model<br/>Elasticsearch / DynamoDB<br/>denormalized: order + user + payment")]
    end

    KAFKA2 --> PROJ --> READ_DB

    DASHBOARD["Dashboard / Search"] --> READ_DB

    style READ_DB fill:#4ecdc4,color:#000
```

**When:** Complex queries across multiple domains (dashboards, search, reporting). Build a **denormalized read model** from events — optimized for the specific query pattern.

### 5.4 Saga for Distributed Transactions

```mermaid
sequenceDiagram
    participant ORCH as Saga Orchestrator
    participant OS6 as Order Service
    participant IS3 as Inventory Service
    participant PS5 as Payment Service

    ORCH->>OS6: CreateOrder
    OS6-->>ORCH: OrderCreated

    ORCH->>IS3: ReserveInventory
    IS3-->>ORCH: InventoryReserved

    ORCH->>PS5: ChargePayment
    PS5-->>ORCH: PaymentFailed ❌

    Note over ORCH: Compensation
    ORCH->>IS3: ReleaseInventory (compensate)
    ORCH->>OS6: CancelOrder (compensate)
```

**When:** Multi-service write operations that must be atomic (all-or-nothing). Replaces distributed transactions (2PC) with compensating actions.

### 5.5 Pattern Selection Guide

```mermaid
graph TD
    NEED{"What do you need?"} -->|"Read data from<br/>another service"| Q1{"How often?"}
    NEED -->|"Write across<br/>multiple services"| SAGA["Saga Pattern"]
    NEED -->|"Complex cross-service<br/>query / report"| CQRS2["CQRS Read Model"]

    Q1 -->|"Occasionally"| API_CALL["API Composition<br/>(call the service)"]
    Q1 -->|"Very frequently"| LOCAL_CACHE["Event-Carried State<br/>(local read copy)"]

    style SAGA fill:#ff8c42,color:#fff
    style CQRS2 fill:#4ecdc4,color:#000
    style API_CALL fill:#a8e6cf,color:#000
    style LOCAL_CACHE fill:#ffe66d,color:#000
```

---

## 6. Database-per-Service Doesn't Mean One Database Server per Service

A common misconception: "database per service = buy N database servers." In practice:

```mermaid
graph TB
    subgraph "Option 1: Separate Schema on Shared Server"
        PG1["PostgreSQL Server"]
        SCHEMA1["schema: orders"]
        SCHEMA2["schema: payments"]
        SCHEMA3["schema: inventory"]
        PG1 --> SCHEMA1
        PG1 --> SCHEMA2
        PG1 --> SCHEMA3
    end

    subgraph "Option 2: Separate Database on Shared Server"
        PG2["PostgreSQL Server"]
        DB_A["database: orders_db"]
        DB_B["database: payments_db"]
        PG2 --> DB_A
        PG2 --> DB_B
    end

    subgraph "Option 3: Fully Separate Instances"
        PG3["PostgreSQL (orders)"]
        PG4["PostgreSQL (payments)"]
        ES["Elasticsearch (search)"]
    end

    style SCHEMA1 fill:#a8e6cf,color:#000
    style DB_A fill:#ffe66d,color:#000
    style PG3 fill:#4ecdc4,color:#000
```

| Isolation Level | Mechanism | Cost | Isolation Strength |
|---|---|---|---|
| **Separate schema** (same server) | Different schema/namespace, same DB server | Lowest | Logical only — shared CPU, memory, connections |
| **Separate database** (same server) | Different databases, same server process | Low | Better — separate connection pools, but shared resources |
| **Separate instance** | Different DB servers (or managed instances) | Highest | Full — independent scaling, failure isolation |

**Start with separate schemas; graduate to separate instances** as services grow and need independent scaling or different database engines.

---

## 7. Polyglot Persistence

Database-per-service enables choosing the **best database for each service's access pattern**:

```mermaid
graph TB
    subgraph "Polyglot Persistence"
        OS7["Order Service<br/>(complex transactions)"] --> PG5[("PostgreSQL<br/>ACID, relational")]
        SS2["Search Service<br/>(full-text search)"] --> ES2[("Elasticsearch<br/>inverted index")]
        REC["Recommendation<br/>(graph traversal)"] --> NEO[("Neo4j<br/>graph DB")]
        SESS["Session Service<br/>(key-value, fast)"] --> REDIS[("Redis<br/>in-memory")]
        IOT2["IoT Telemetry<br/>(time-series)"] --> TS[("TimescaleDB<br/>time-series")]
        CONTENT["Content Service<br/>(flexible schema)"] --> MONGO[("MongoDB<br/>document store")]
    end

    style PG5 fill:#4ecdc4,color:#000
    style ES2 fill:#ffe66d,color:#000
    style NEO fill:#ff8c42,color:#fff
    style REDIS fill:#a8e6cf,color:#000
    style TS fill:#ff6b6b,color:#fff
```

| Access Pattern | Best Fit Database |
|---|---|
| Complex transactions, relations | PostgreSQL, MySQL |
| Full-text search, filtering | Elasticsearch, OpenSearch |
| Key-value, caching, sessions | Redis, Memcached |
| Document store, flexible schema | MongoDB, DynamoDB |
| Time-series data | TimescaleDB, InfluxDB |
| Graph relationships | Neo4j, Amazon Neptune |
| Wide-column, high write throughput | Cassandra, ScyllaDB |
| Event log, streaming | Kafka (as storage) |

---

## 8. When a Shared Database Might Be Acceptable

Despite the recommendation, there are scenarios where sharing is pragmatic:

```mermaid
graph TD
    Q{"When is shared DB acceptable?"} -->|"Early startup"| A["< 3 services, 1 team,<br/>rapidly changing domain"]
    Q -->|"Migration transition"| B["Strangler Fig: monolith + new service<br/>share DB temporarily"]
    Q -->|"Read-only replicas"| C["Services read from a shared<br/>replica / data warehouse"]
    Q -->|"Same bounded context"| D["Two services in the SAME<br/>bounded context, same team"]

    A --> WARN1["⚠️ Plan to split when services stabilize"]
    B --> WARN2["⚠️ Transitional — not a target state"]
    C --> ACCEPT["✅ Acceptable — read replica, no write coupling"]
    D --> ACCEPT2["✅ Acceptable — really one service, two processes"]

    style A fill:#ffe66d,color:#000
    style B fill:#ffe66d,color:#000
    style C fill:#4ecdc4,color:#000
    style D fill:#4ecdc4,color:#000
```

| Scenario | Shared DB OK? | Condition |
|---|---|---|
| **Startup with 2–3 services, 1 team** | Temporarily yes | Plan the split; use separate schemas at minimum |
| **During monolith → microservices migration** | Temporarily yes | Strangler Fig transitional phase |
| **Shared read replica / data warehouse** | Yes | Read-only; no write coupling |
| **Two processes in same bounded context** | Yes | Same team, same domain, same deploy — arguably one service |
| **Stable production microservices** | No | Violates independent deployability |

---

## 9. Operational Considerations

### 9.1 Managing N Databases

```mermaid
graph LR
    subgraph "How to Manage Multiple Databases"
        IAC["Infrastructure-as-Code<br/>(Terraform modules)"]
        MANAGED["Managed Services<br/>(RDS, Cloud SQL, Atlas)"]
        BACKUP["Automated Backups<br/>(per-service retention policy)"]
        MONITOR["Per-DB Monitoring<br/>(connections, slow queries, disk)"]
        MIGRATE["Schema Migrations<br/>(Flyway, Alembic per service)"]
    end

    style IAC fill:#4ecdc4,color:#000
    style MANAGED fill:#ffe66d,color:#000
```

| Concern | Solution |
|---|---|
| **Provisioning N databases** | Terraform modules; service template auto-provisions DB |
| **Backup and recovery** | Managed service handles it (RDS snapshots, Cloud SQL backups) |
| **Monitoring** | Per-service dashboards: connection pool, query latency, disk |
| **Schema migrations** | Each service runs its own migrations (Flyway, Alembic, Liquibase) |
| **Cost** | Start with shared server + separate schemas; separate instances only for high-traffic services |
| **Connection pool management** | PgBouncer / ProxySQL for connection pooling at scale |

### 9.2 Migration Path from Shared to Separate

```mermaid
graph LR
    S1["1. Shared DB<br/>(all tables together)"]
    S2["2. Separate schemas<br/>(same server, no cross-schema queries)"]
    S3["3. Remove cross-schema<br/>foreign keys → use API/events"]
    S4["4. Separate instances<br/>(if needed for scaling)"]

    S1 --> S2 --> S3 --> S4

    style S1 fill:#ff6b6b,color:#fff
    style S2 fill:#ffe66d,color:#000
    style S3 fill:#a8e6cf,color:#000
    style S4 fill:#4ecdc4,color:#000
```

---

## 10. Data Consistency Patterns Summary

| Pattern | Consistency | Complexity | Use Case |
|---|---|---|---|
| **Single DB transaction** | Strong (ACID) | Low | Only within one service's database |
| **Saga (orchestration)** | Eventual (compensating) | Medium | Multi-service writes with defined steps |
| **Saga (choreography)** | Eventual (event-driven) | Medium | Decoupled workflows |
| **Outbox pattern** | At-least-once delivery | Medium | Reliable event publishing from DB writes |
| **Event sourcing** | Eventual | High | Full audit trail, temporal queries |
| **Two-phase commit (2PC)** | Strong | High | ❌ Avoid in microservices (blocks, doesn't scale) |

---

## 11. Anti-Patterns

| Anti-Pattern | Problem | Fix |
|---|---|---|
| **Shared database as the integration layer** | Services coupled at schema level; can't deploy independently | Database per service + API/event integration |
| **Cross-service JOINs via shared DB** | Tight data coupling; schema change breaks consumers | API composition or CQRS read model |
| **Direct table access by other services** | No encapsulation; any service can read/write any table | Each service's DB is private; expose data via API |
| **Distributed transactions (2PC)** | Locks, latency, partial failures, doesn't scale | Saga pattern with compensating actions |
| **"Just add a foreign key"** | Cross-service FK creates schema coupling | Store IDs as opaque references; validate via API |
| **One giant read model** | Central data warehouse becomes a bottleneck | Purpose-built read models per consumer |
| **Premature polyglot** | Using 5 different databases for 5 services when PostgreSQL works for all | Start with one engine; diversify when access patterns demand it |
| **No data ownership** | "The `orders` table belongs to everyone" | One service owns each table; others access via API |

---

## 12. Decision Framework

```mermaid
graph TD
    START{"How many services<br/>and teams?"} -->|"1–3 services, 1 team"| SHARED["Shared DB OK<br/>(separate schemas minimum)"]
    START -->|"3+ services, multiple teams"| PER_SVC["Database per service"]

    PER_SVC --> Q1{"Need cross-service reads?"}
    Q1 -->|"Occasionally"| API["API Composition"]
    Q1 -->|"Frequently"| ECST["Event-Carried State Transfer<br/>(local read copy)"]
    Q1 -->|"Complex queries / dashboards"| CQRS3["CQRS Read Model"]

    PER_SVC --> Q2{"Need cross-service writes?"}
    Q2 -->|"Yes"| SAGA2["Saga Pattern"]
    Q2 -->|"No"| SIMPLE["Simple — each service writes its own DB"]

    PER_SVC --> Q3{"Different access patterns?"}
    Q3 -->|"Yes"| POLYGLOT["Polyglot Persistence<br/>(right DB per service)"]
    Q3 -->|"No, relational works for all"| SAME_ENGINE["Same engine, separate instances/schemas"]

    style PER_SVC fill:#4ecdc4,color:#000
    style SHARED fill:#ffe66d,color:#000
```

---

## 13. Checklist

### Data Ownership
- [ ] Each service owns its database exclusively (no shared write access)
- [ ] Other services access data only via the owning service's API or events
- [ ] No cross-service foreign keys in any database
- [ ] Data ownership documented in service catalog

### Data Access
- [ ] Cross-service reads use API composition or event-carried state
- [ ] Cross-service queries use CQRS read models (not shared DB JOINs)
- [ ] Multi-service writes use saga pattern (not distributed transactions)
- [ ] Outbox pattern used for reliable event publishing

### Operations
- [ ] Each service manages its own schema migrations
- [ ] Databases provisioned via IaC (Terraform / Pulumi)
- [ ] Per-service database monitoring (connections, queries, disk)
- [ ] Backup and recovery tested per service
- [ ] Connection pooling configured appropriately

### Evolution
- [ ] Migration path from shared schema → separate schema → separate instance documented
- [ ] Polyglot persistence considered where access patterns diverge
- [ ] Data consistency requirements explicitly documented per interaction (strong vs. eventual)

---

## 14. Recommendation

**Database-per-service is the correct default for microservices.**

| Phase | Approach |
|---|---|
| **Starting out (< 3 services, 1 team)** | Shared server, separate schemas — low cost, logical isolation |
| **Growing (3–10 services)** | Separate schemas, eliminate cross-schema queries, add event-driven data sharing |
| **Mature (10+ services)** | Separate instances for high-traffic services; polyglot persistence where needed |
| **At scale** | Managed databases, CQRS read models, event-carried state, automated provisioning via platform |

The core principle: **a service's database is a private implementation detail, just like its code.** No other service should know or care what tables exist, what engine is used, or how data is stored. The only contract is the service's API. This is what makes independent deployment, independent scaling, and independent evolution possible.

---

**Next steps to explore:** Saga Pattern Deep Dive, CQRS & Event Sourcing, Data Mesh for Microservices.