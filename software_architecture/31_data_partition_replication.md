## Data Partitioning & Data Replication in Microservices

### Context & Assumptions

In microservices, each service owns its data (Database per Service pattern). As data volume and traffic grow, a single database instance becomes a bottleneck — for storage capacity, query throughput, or write performance. **Data partitioning** (sharding) distributes data across multiple database instances to scale horizontally. **Data replication** copies data across nodes for availability, fault tolerance, and read scalability. Together, they are the foundational strategies for building data layers that match the distributed nature of microservices.

---

### Core Concepts

| Concept | Definition |
|---|---|
| **Partitioning (Sharding)** | Splitting a dataset across multiple nodes so each node holds a subset |
| **Replication** | Copying the same data to multiple nodes for redundancy and read scaling |
| **Partition Key (Shard Key)** | The field used to determine which partition holds a given record |
| **Replica** | A copy of a partition (or full database) on another node |
| **Leader (Primary)** | The node that accepts writes for a partition |
| **Follower (Replica)** | A node that receives replicated data and serves reads |
| **Rebalancing** | Redistributing data when partitions are added or removed |
| **Consistency Model** | The guarantee about how up-to-date replicated data is |

---

## Part 1: Data Partitioning (Sharding)

### Why Partition?

```mermaid
graph TD
    subgraph "Without Partitioning"
        SINGLE[(Single Database<br/>100M orders<br/>10K writes/sec<br/>⚠️ CPU/IO saturated)]
    end

    subgraph "With Partitioning"
        P1[(Partition 1<br/>25M orders<br/>2.5K writes/sec)]
        P2[(Partition 2<br/>25M orders<br/>2.5K writes/sec)]
        P3[(Partition 3<br/>25M orders<br/>2.5K writes/sec)]
        P4[(Partition 4<br/>25M orders<br/>2.5K writes/sec)]
    end

    style SINGLE fill:#ef5350,stroke:#333,color:#fff
    style P1 fill:#66bb6a,stroke:#333,color:#000
    style P2 fill:#66bb6a,stroke:#333,color:#000
    style P3 fill:#66bb6a,stroke:#333,color:#000
    style P4 fill:#66bb6a,stroke:#333,color:#000
```

| Benefit | Description |
|---|---|
| **Write scalability** | Distribute writes across N nodes → N× throughput |
| **Storage scalability** | Each node holds 1/N of data → virtually unlimited capacity |
| **Query performance** | Smaller dataset per node → faster index scans |
| **Isolation** | One hot partition doesn't affect others |

---

### Partitioning Strategies

#### 1. Hash-Based Partitioning

```mermaid
graph TD
    subgraph "Hash-Based Partitioning"
        REC[Record: orderId=12345] --> HASH["hash(orderId) mod N"]
        HASH -->|"hash = 2"| P2[(Partition 2)]
    end

    subgraph "Distribution"
        P0[(Partition 0)] --- D0["hash mod 4 = 0"]
        P1[(Partition 1)] --- D1["hash mod 4 = 1"]
        P2B[(Partition 2)] --- D2["hash mod 4 = 2"]
        P3[(Partition 3)] --- D3["hash mod 4 = 3"]
    end

    style HASH fill:#42a5f5,stroke:#333,color:#fff
```

| Aspect | Detail |
|---|---|
| **Mechanism** | `partition = hash(key) % num_partitions` |
| **Distribution** | Uniform — even data spread across partitions |
| **Range queries** | ❌ Impossible — adjacent keys land on different partitions |
| **Adding partitions** | Requires rehashing all data (unless consistent hashing) |
| **Best for** | Key-value lookups, high cardinality keys (userId, orderId) |

#### 2. Range-Based Partitioning

```mermaid
graph TD
    subgraph "Range-Based Partitioning"
        REC[Record: date=2026-04-18] --> RANGE["Which range?"]
        RANGE -->|"2026-Q1"| P1[(Partition: Q1 2026)]
        RANGE -->|"2026-Q2"| P2[(Partition: Q2 2026)]
    end

    subgraph "Partition Ranges"
        PR1[(Partition 1<br/>2025-01 to 2025-06)]
        PR2[(Partition 2<br/>2025-07 to 2025-12)]
        PR3[(Partition 3<br/>2026-01 to 2026-06)]
        PR4[(Partition 4<br/>2026-07 to 2026-12)]
    end

    style RANGE fill:#f9a825,stroke:#333,color:#000
```

| Aspect | Detail |
|---|---|
| **Mechanism** | Key ranges mapped to partitions |
| **Distribution** | Can be skewed — hot ranges get more traffic |
| **Range queries** | ✅ Efficient — sequential keys on same partition |
| **Adding partitions** | Split a range into two — no full rehash |
| **Best for** | Time-series data, alphabetical ranges, sequential access |

#### 3. Consistent Hashing

```mermaid
graph TD
    subgraph "Consistent Hash Ring"
        direction LR
        RING[Hash Ring 0...2^32]
        N1[Node A<br/>position: 1000] 
        N2[Node B<br/>position: 4000]
        N3[Node C<br/>position: 7000]
        
        K1["Key X<br/>hash=800 → Node A"]
        K2["Key Y<br/>hash=3500 → Node B"]
        K3["Key Z<br/>hash=5000 → Node C"]
    end

    style N1 fill:#42a5f5,stroke:#333,color:#fff
    style N2 fill:#66bb6a,stroke:#333,color:#000
    style N3 fill:#f9a825,stroke:#333,color:#000
```

| Aspect | Detail |
|---|---|
| **Mechanism** | Nodes and keys mapped onto a ring; key goes to nearest clockwise node |
| **Adding a node** | Only ~1/N of keys remapped (minimal disruption) |
| **Removing a node** | Its keys move to next node on ring |
| **Virtual nodes** | Each physical node gets multiple positions → better uniformity |
| **Used by** | Cassandra, DynamoDB, Redis Cluster, Memcached |

#### 4. Directory-Based (Lookup Table)

```mermaid
graph TD
    subgraph "Directory-Based Partitioning"
        APP[Application] --> DIR[(Directory Service<br/>/ Lookup Table)]
        DIR -->|"tenant_A → Shard 1"| S1[(Shard 1)]
        DIR -->|"tenant_B → Shard 2"| S2[(Shard 2)]
        DIR -->|"tenant_C → Shard 1"| S1
        DIR -->|"tenant_D → Shard 3"| S3[(Shard 3)]
    end

    style DIR fill:#ab47bc,stroke:#333,color:#fff
```

| Aspect | Detail |
|---|---|
| **Mechanism** | Explicit mapping table: key → partition |
| **Flexibility** | Full control over placement; move tenants between shards |
| **Overhead** | Lookup on every request (cache aggressively) |
| **Best for** | Multi-tenant SaaS, uneven tenant sizes, compliance (data residency) |

---

### Partitioning Strategy Comparison

| Strategy | Distribution | Range Queries | Rebalancing Cost | Hot Spots | Complexity |
|---|---|---|---|---|---|
| **Hash-based** | Uniform | ❌ No | High (full rehash) | Low | Low |
| **Consistent hashing** | Uniform (with vnodes) | ❌ No | Low (minimal remapping) | Low | Medium |
| **Range-based** | Can be skewed | ✅ Yes | Medium (split range) | High (time-based) | Medium |
| **Directory-based** | Controlled | ✅ Possible | Low (update directory) | Controlled | High |
| **Composite** | Depends | ✅ Within partition | Varies | Reduced | High |

---

### Composite Partition Key

```mermaid
graph TD
    subgraph "Composite Key: tenant_id + order_date"
        REC["Order: tenant=ACME, date=2026-04-18"]
        PART["Partition by: hash(tenant_id)"]
        SORT["Sort within partition by: order_date"]
        
        REC --> PART
        PART --> S1[(Shard 1: ACME)]
        S1 --> SORT
        SORT --> ROW["Rows sorted by date<br/>→ efficient range scan within tenant"]
    end

    style PART fill:#42a5f5,stroke:#333,color:#fff
    style SORT fill:#66bb6a,stroke:#333,color:#000
```

**Pattern used by:** Cassandra (partition key + clustering key), DynamoDB (partition key + sort key), CockroachDB.

This gives you **even distribution** (hash on tenant) plus **efficient range queries** (sort on date within partition).

---

### Cross-Partition Queries (The Hard Problem)

```mermaid
graph TD
    subgraph "Single-Partition Query (Fast)"
        Q1["SELECT * FROM orders<br/>WHERE tenant_id = 'ACME'<br/>AND date > '2026-04-01'"]
        Q1 -->|Routes to one shard| S1[(Shard: ACME)]
    end

    subgraph "Cross-Partition Query (Scatter-Gather)"
        Q2["SELECT * FROM orders<br/>WHERE total > 1000<br/>ORDER BY date"]
        Q2 -->|Fan out to ALL shards| S2A[(Shard 1)]
        Q2 --> S2B[(Shard 2)]
        Q2 --> S2C[(Shard 3)]
        Q2 --> S2D[(Shard 4)]
        S2A --> AGG[Aggregator<br/>Merge + Sort]
        S2B --> AGG
        S2C --> AGG
        S2D --> AGG
    end

    style Q1 fill:#66bb6a,stroke:#333,color:#000
    style Q2 fill:#ef5350,stroke:#333,color:#fff
    style AGG fill:#f9a825,stroke:#333,color:#000
```

| Pattern | How It Handles Cross-Partition | Trade-off |
|---|---|---|
| **Scatter-gather** | Query all partitions, merge results | Latency = slowest partition; expensive at scale |
| **Secondary index (global)** | Separate index spanning all partitions | Index must be kept in sync; eventual consistency |
| **Materialized view / CQRS** | Pre-computed read model for cross-partition queries | Duplication; eventual consistency |
| **Search index (Elasticsearch)** | Denormalized data in search engine | Separate system to maintain |
| **Data warehouse / analytics DB** | ETL into analytical store | High latency; not for real-time |

---

### Database-Specific Partitioning

| Database | Partitioning Type | Partition Key | Automatic? | Cross-Partition Queries |
|---|---|---|---|---|
| **PostgreSQL** | Declarative (range, list, hash) | Table column | Manual DDL | Yes (query planner routes) |
| **MySQL** | Range, list, hash, key | Table column | Manual DDL | Yes (partition pruning) |
| **Cassandra** | Consistent hashing | Partition key column | Automatic | Scatter-gather (ALLOW FILTERING) |
| **DynamoDB** | Hash-based | Partition key | Automatic | Scan (expensive) or GSI |
| **MongoDB** | Range or hash | Shard key | Automatic (via mongos) | Scatter-gather via mongos |
| **CockroachDB** | Range-based (auto-split) | Primary key | Automatic | Distributed SQL (transparent) |
| **Vitess (MySQL)** | Hash or range | VIndex column | Managed | Scatter-gather via vtgate |
| **Citus (PostgreSQL)** | Hash or append | Distribution column | Managed | Distributed queries via coordinator |

---

## Part 2: Data Replication

### Why Replicate?

```mermaid
graph TD
    subgraph "Without Replication"
        SINGLE[(Single Node<br/>⚠️ SPOF<br/>All reads + writes)]
    end

    subgraph "With Replication"
        PRIMARY[(Primary<br/>Writes)] -->|Replicate| R1[(Replica 1<br/>Reads)]
        PRIMARY -->|Replicate| R2[(Replica 2<br/>Reads)]
        PRIMARY -->|Replicate| R3[(Replica 3<br/>Standby<br/>Failover)]
    end

    style SINGLE fill:#ef5350,stroke:#333,color:#fff
    style PRIMARY fill:#42a5f5,stroke:#333,color:#fff
    style R1 fill:#66bb6a,stroke:#333,color:#000
    style R2 fill:#66bb6a,stroke:#333,color:#000
    style R3 fill:#f9a825,stroke:#333,color:#000
```

| Benefit | Description |
|---|---|
| **High availability** | Node dies → replica takes over (failover) |
| **Read scalability** | Distribute reads across N replicas → N× read throughput |
| **Geographic distribution** | Place replicas near users → lower latency |
| **Disaster recovery** | Cross-region replica survives regional outage |

---

### Replication Topologies

#### 1. Single-Leader (Primary-Replica)

```mermaid
graph TD
    subgraph "Single-Leader Replication"
        W[Writes] --> P[(Primary / Leader)]
        P -->|Async replicate| R1[(Replica 1)]
        P -->|Async replicate| R2[(Replica 2)]
        RD[Reads] --> R1
        RD --> R2
        RD --> P
    end

    style P fill:#42a5f5,stroke:#333,color:#fff
    style R1 fill:#66bb6a,stroke:#333,color:#000
    style R2 fill:#66bb6a,stroke:#333,color:#000
```

| Aspect | Detail |
|---|---|
| **Writes** | Only to leader |
| **Reads** | From any replica (may be stale) or leader (always current) |
| **Consistency** | Eventual (async) or strong (sync replicas) |
| **Failover** | Promote a replica to leader |
| **Used by** | PostgreSQL, MySQL, MongoDB (replica set), Redis |

#### 2. Multi-Leader (Active-Active)

```mermaid
graph TD
    subgraph "Multi-Leader Replication"
        L1[(Leader 1<br/>US-East)] <-->|Bi-directional<br/>replication| L2[(Leader 2<br/>EU-West)]
        L1 <-->|Bi-directional| L3[(Leader 3<br/>AP-Southeast)]
        L2 <-->|Bi-directional| L3

        C1[US Clients] --> L1
        C2[EU Clients] --> L2
        C3[APAC Clients] --> L3
    end

    style L1 fill:#42a5f5,stroke:#333,color:#fff
    style L2 fill:#42a5f5,stroke:#333,color:#fff
    style L3 fill:#42a5f5,stroke:#333,color:#fff
```

| Aspect | Detail |
|---|---|
| **Writes** | Any leader accepts writes |
| **Conflict** | Same record modified on two leaders → **conflict resolution required** |
| **Latency** | Low — writes go to local leader |
| **Complexity** | High — conflict resolution is notoriously difficult |
| **Used by** | CockroachDB, Cassandra, DynamoDB Global Tables, MySQL Group Replication |

#### 3. Leaderless (Quorum-Based)

```mermaid
graph TD
    subgraph "Leaderless Replication (Quorum)"
        CLIENT[Client] -->|Write to 3 of 5 nodes<br/>W=3| N1[(Node 1 ✅)]
        CLIENT -->|Write| N2[(Node 2 ✅)]
        CLIENT -->|Write| N3[(Node 3 ✅)]
        CLIENT -.->|Write| N4[(Node 4 ⏳ slow)]
        CLIENT -.->|Write| N5[(Node 5 ❌ down)]

        CR[Client Read] -->|Read from 3 of 5 nodes<br/>R=3| N1
        CR --> N2
        CR --> N4
    end

    style CLIENT fill:#f9a825,stroke:#333,color:#000
    style N5 fill:#ef5350,stroke:#333,color:#fff
```

**Quorum formula:** For N replicas, write to W nodes, read from R nodes. Consistency guaranteed when:

$$W + R > N$$

| Configuration | W | R | N | Consistency | Availability |
|---|---|---|---|---|---|
| **Strong** | 3 | 3 | 5 | Strong (overlap guaranteed) | Tolerates 2 failures |
| **Write-heavy** | 1 | 5 | 5 | Eventual | Fast writes; slow reads |
| **Read-heavy** | 5 | 1 | 5 | Eventually consistent reads | Slow writes; fast reads |
| **Balanced** | 2 | 2 | 3 | Strong | Tolerates 1 failure |

**Used by:** Cassandra, DynamoDB, Riak, Voldemort.

---

### Replication Topology Comparison

| Aspect | Single-Leader | Multi-Leader | Leaderless |
|---|---|---|---|
| **Write throughput** | Limited by leader | High (any leader) | High (any node) |
| **Read consistency** | Strong from leader; eventual from replicas | Eventual across leaders | Tunable (quorum) |
| **Conflict handling** | None (single writer) | Required (LWW, CRDTs, merge) | Required (vector clocks, read repair) |
| **Failover** | Promote replica | Other leaders continue | No failover needed |
| **Geographic distribution** | Leader in one region (write latency from others) | Leader per region (low local latency) | Any node per region |
| **Complexity** | Low | High | Medium |
| **Best for** | Most OLTP workloads | Multi-region active-active | High availability, AP systems |

---

### Synchronous vs. Asynchronous Replication

```mermaid
sequenceDiagram
    participant C as Client
    participant L as Leader
    participant R1 as Replica 1
    participant R2 as Replica 2

    Note over C,R2: Synchronous Replication

    C->>L: Write(x=5)
    L->>L: Write to local WAL
    par Replicate to all
        L->>R1: Replicate x=5
        R1-->>L: ACK ✅
        L->>R2: Replicate x=5
        R2-->>L: ACK ✅
    end
    L-->>C: Write confirmed ✅
    Note over C: High latency but<br/>guaranteed durable

    Note over C,R2: Asynchronous Replication

    C->>L: Write(y=10)
    L->>L: Write to local WAL
    L-->>C: Write confirmed ✅
    Note over C: Low latency but<br/>data may be lost on leader failure
    L->>R1: Replicate y=10 (background)
    L->>R2: Replicate y=10 (background)
```

| Mode | Durability | Latency | Availability | Risk |
|---|---|---|---|---|
| **Synchronous (all)** | Highest — all nodes have data | Highest — wait for slowest replica | Lowest — blocked if any replica down | Zero data loss |
| **Semi-synchronous** | High — at least 1 replica confirmed | Medium | Medium | Minimal data loss |
| **Asynchronous** | Lower — leader only has confirmed data | Lowest | Highest | Data loss on leader failure (replication lag) |

---

### Replication Lag & Consistency Problems

```mermaid
sequenceDiagram
    participant C as Client
    participant L as Leader
    participant R as Replica

    C->>L: UPDATE account SET name='Alice'
    L-->>C: OK ✅
    L->>R: Replicate (asynchronous, delayed)

    C->>R: SELECT name FROM account
    R-->>C: name='Bob' ← STALE! ❌
    Note over C: Read-after-write inconsistency<br/>Client sees old value
    
    Note over R: 200ms later...
    L->>R: Replicate arrives
    Note over R: Now name='Alice' ✅
```

| Problem | Description | Solution |
|---|---|---|
| **Read-after-write inconsistency** | Client writes, then reads from replica before replication | Read-your-writes: route user's reads to leader for N seconds after write |
| **Monotonic read violation** | Client reads newer value, then older from different replica | Sticky sessions: pin client to one replica |
| **Causal ordering violation** | Reply appears before the message it replies to | Causal consistency tracking (vector clocks, logical timestamps) |
| **Stale reads** | Replica is seconds/minutes behind leader | Monitor replication lag; route critical reads to leader |

---

### Conflict Resolution (Multi-Leader / Leaderless)

```mermaid
graph TD
    subgraph "Conflict: Same record updated on two leaders"
        L1[Leader 1<br/>SET price = $10<br/>timestamp: T1] 
        L2[Leader 2<br/>SET price = $15<br/>timestamp: T2]
        L1 -->|Replicate| CONFLICT{Conflict!<br/>price = $10 or $15?}
        L2 -->|Replicate| CONFLICT
    end

    subgraph "Resolution Strategies"
        CONFLICT -->|LWW| RES1[Last-Writer-Wins<br/>T2 > T1 → price = $15<br/>⚠️ Data loss: T1 overwritten]
        CONFLICT -->|Merge| RES2[Application merge<br/>Custom logic decides]
        CONFLICT -->|CRDT| RES3[CRDT: Conflict-free<br/>Automatic merge<br/>e.g., counters, sets]
        CONFLICT -->|Manual| RES4[Flag for human<br/>resolution]
    end

    style CONFLICT fill:#ef5350,stroke:#333,color:#fff
```

| Strategy | Mechanism | Data Loss? | Complexity | Best For |
|---|---|---|---|---|
| **Last-Writer-Wins (LWW)** | Highest timestamp wins | Yes — silent loss | Low | Non-critical, idempotent data |
| **Version vector / vector clock** | Track causal history; detect true conflicts | No — conflicts detected | High | Distributed KV stores (Riak) |
| **CRDTs** | Mathematically convergent data structures | No | Medium | Counters, sets, registers |
| **Application-level merge** | Custom business logic resolves | No — app decides | High | Domain-specific conflict rules |
| **Manual resolution** | Present both versions to user | No | Low (tech), high (UX) | Collaborative editing |

---

## Part 3: Partitioning + Replication Together

### Combined Architecture

```mermaid
graph TD
    subgraph "Partitioned + Replicated Database"
        subgraph "Partition 1 (Users A-M)"
            P1L[(P1 Leader<br/>Node 1)] -->|Replicate| P1R1[(P1 Replica<br/>Node 4)]
            P1L -->|Replicate| P1R2[(P1 Replica<br/>Node 5)]
        end

        subgraph "Partition 2 (Users N-Z)"
            P2L[(P2 Leader<br/>Node 2)] -->|Replicate| P2R1[(P2 Replica<br/>Node 5)]
            P2L -->|Replicate| P2R2[(P2 Replica<br/>Node 6)]
        end

        subgraph "Partition 3 (Orders)"
            P3L[(P3 Leader<br/>Node 3)] -->|Replicate| P3R1[(P3 Replica<br/>Node 4)]
            P3L -->|Replicate| P3R2[(P3 Replica<br/>Node 6)]
        end
    end

    ROUTER[Query Router<br/>/ Coordinator] --> P1L
    ROUTER --> P2L
    ROUTER --> P3L

    style ROUTER fill:#f9a825,stroke:#333,color:#000
    style P1L fill:#42a5f5,stroke:#333,color:#fff
    style P2L fill:#42a5f5,stroke:#333,color:#fff
    style P3L fill:#42a5f5,stroke:#333,color:#fff
```

**Each partition has its own leader and replicas.** The leaders are distributed across nodes so no single node is the bottleneck. If Node 1 dies, P1 Replica on Node 4 is promoted to leader — other partitions are unaffected.

---

### Cross-Service Data Replication in Microservices

Beyond database-level replication, microservices replicate data **across service boundaries** for self-containment:

```mermaid
graph TD
    subgraph "Service-Level Data Replication"
        PS[Product Service] -->|ProductUpdated events| KAFKA[(Kafka)]
        KAFKA --> OS[Order Service<br/>Local product projection]
        KAFKA --> SS[Search Service<br/>Elasticsearch index]
        KAFKA --> RS[Recommendation Service<br/>ML feature store]
        KAFKA --> AS[Analytics Service<br/>Data warehouse]
    end

    subgraph "Order Service Internal"
        OS --> ODB[(Orders DB<br/>Owned: orders table<br/>Projection: products table)]
    end

    style KAFKA fill:#f9a825,stroke:#333,color:#000
    style PS fill:#42a5f5,stroke:#333,color:#fff
    style ODB fill:#66bb6a,stroke:#333,color:#000
```

| Mechanism | Latency | Ordering | Best For |
|---|---|---|---|
| **Event-Carried State Transfer** | Near real-time | Per partition key | Service-to-service data sharing |
| **CDC (Debezium)** | Milliseconds | WAL order | DB-to-DB replication, legacy integration |
| **API Polling** | Seconds-minutes | No guarantee | Simple, low-volume reference data |
| **Shared Cache (Redis)** | Sub-millisecond reads | No guarantee | Session state, hot data |

---

### Multi-Region Partitioning + Replication

```mermaid
graph TD
    subgraph "US-East Region"
        US_LB[Load Balancer] --> US_APP[Services]
        US_APP --> US_DB[(DB: US Partitions<br/>Leader for US users)]
        US_DB -->|Async replicate| EU_DB
        US_DB -->|Async replicate| AP_DB
    end

    subgraph "EU-West Region"
        EU_LB[Load Balancer] --> EU_APP[Services]
        EU_APP --> EU_DB[(DB: EU Partitions<br/>Leader for EU users)]
        EU_DB -->|Async replicate| US_DB
        EU_DB -->|Async replicate| AP_DB
    end

    subgraph "AP-Southeast Region"
        AP_LB[Load Balancer] --> AP_APP[Services]
        AP_APP --> AP_DB[(DB: AP Partitions<br/>Leader for AP users)]
        AP_DB -->|Async replicate| US_DB
        AP_DB -->|Async replicate| EU_DB
    end

    style US_DB fill:#42a5f5,stroke:#333,color:#fff
    style EU_DB fill:#66bb6a,stroke:#333,color:#000
    style AP_DB fill:#f9a825,stroke:#333,color:#000
```

**Geo-partitioning:** Each region is the **leader** for its local users' data. Cross-region reads use local replicas. This gives low write latency (writes go to local leader) and low read latency (reads from local replica), with eventual consistency across regions.

| Pattern | Write Latency | Read Latency | Consistency | Complexity |
|---|---|---|---|---|
| **Single-region leader, global replicas** | High (remote writes) | Low (local reads) | Strong writes, eventual reads | Low |
| **Geo-partitioned leaders** | Low (local writes) | Low (local reads) | Eventual across regions | High |
| **Global consensus (Spanner/CockroachDB)** | Medium (quorum across regions) | Low (leader lease reads) | Strong (linearizable) | Very High |

---

### Partition Key Selection Guide

Choosing the right partition key is the **most critical decision** — a bad key causes hot spots, cross-partition queries, and poor scalability.

| Data Domain | Good Partition Key | Bad Partition Key | Why |
|---|---|---|---|
| **Orders** | `customer_id` | `order_date` | Date = time-series hot spot; customer = even distribution |
| **Multi-tenant SaaS** | `tenant_id` | `created_at` | Tenant isolates workloads; date skews to current |
| **Social posts** | `user_id` | `post_id` (sequential) | Sequential IDs cluster on one partition |
| **IoT telemetry** | `device_id` | `timestamp` | Timestamp = all writes to one partition |
| **E-commerce products** | `category_id` (with hash) | `product_name` | Names cluster alphabetically |
| **Chat messages** | `conversation_id` | `sender_id` | Conversation = natural query boundary |

**Rules:**
1. High cardinality (many distinct values → even distribution)
2. Matches query pattern (most queries filter by this key → single-partition access)
3. Even write distribution (no single value gets disproportionate writes)
4. Stable (doesn't change → no record migration)

---

### Anti-Patterns

| Anti-Pattern | Problem | Remedy |
|---|---|---|
| **Premature sharding** | Sharding before needed adds massive complexity | Vertical scaling + read replicas first; shard when single node capacity exceeded |
| **Sequential partition key** | Auto-increment ID → all writes to latest partition (hot spot) | Use UUID, hash, or composite key |
| **Cross-partition joins** | Queries that span all partitions negate sharding benefits | Denormalize; use CQRS read models; choose partition key aligned with query |
| **Ignoring replication lag** | Reading from replica immediately after write → stale data | Read-your-writes to leader; monitor lag; set SLOs on lag |
| **No rebalancing strategy** | Adding a partition requires reshuffling all data | Use consistent hashing; plan rebalancing tooling upfront |
| **LWW without understanding** | Last-Writer-Wins silently drops data in multi-leader | Understand conflict semantics; use CRDTs or app-level merge for important data |
| **Homogeneous replicas** | All replicas have same spec → no cost optimization | Use heterogeneous replicas: fast SSD for leader, cheaper storage for read replicas |
| **No partition monitoring** | One partition is 100× larger than others — undetected | Monitor partition size distribution; alert on skew |
| **Replicating everything cross-service** | Every service has a copy of every other service's data | Replicate only what each service needs for its primary function |
| **Synchronous replication everywhere** | All writes wait for all replicas → high latency, low availability | Semi-sync (1 confirmed replica) for durability; async for read replicas |

---

### Monitoring

| Metric | Alert Threshold | Meaning |
|---|---|---|
| `replication_lag_seconds` | > 5s sustained | Replica falling behind — may serve stale data |
| `partition_size_ratio` (max/min) | > 3:1 | Data skew — hot partition or bad partition key |
| `cross_partition_query_ratio` | > 20% of queries | Partition key misaligned with query patterns |
| `partition_write_rate_skew` | > 5:1 hottest/average | Write hot spot — rebalance or change partition key |
| `failover_time_seconds` | > 30s | Replica promotion too slow — impacts availability |
| `replication_error_rate` | > 0 | Replication pipeline broken — data divergence |

---

### Decision Framework

```mermaid
graph TD
    Q1{Single DB instance<br/>hitting limits?} -->|No| SKIP[No partitioning needed<br/>Use read replicas for read scaling]
    Q1 -->|Write throughput limit| SHARD[Partition / Shard]
    Q1 -->|Storage limit| SHARD
    Q1 -->|Read throughput limit| REPLICA[Add read replicas first]

    SHARD --> Q2{Query pattern?}
    Q2 -->|Point lookups by key| HASH[Hash partitioning]
    Q2 -->|Range scans by time/value| RANGE[Range partitioning]
    Q2 -->|Multi-tenant isolation| DIR[Directory-based<br/>or tenant-per-shard]
    Q2 -->|Mixed| COMPOSITE[Composite key<br/>hash partition + range sort]

    REPLICA --> Q3{Availability requirement?}
    Q3 -->|Single region, HA| SR[Single-leader + 2 replicas<br/>Semi-sync]
    Q3 -->|Multi-region, active-active| MR[Multi-leader or<br/>geo-partitioned]
    Q3 -->|Maximum availability| LL[Leaderless / quorum<br/>Cassandra, DynamoDB]

    style SHARD fill:#42a5f5,stroke:#333,color:#fff
    style REPLICA fill:#66bb6a,stroke:#333,color:#000
    style HASH fill:#f9a825,stroke:#333,color:#000
```

---

### Practical Checklist

**Partitioning:**
- [ ] Exhaust vertical scaling and read replicas before sharding
- [ ] Choose partition key based on **query patterns**, not just data distribution
- [ ] Verify key has **high cardinality** and **even write distribution**
- [ ] Use **consistent hashing** (or managed sharding) to simplify rebalancing
- [ ] Plan for **cross-partition queries** — CQRS read models or search indexes
- [ ] Monitor partition **size skew** and **write rate skew**
- [ ] Test rebalancing procedure before you need it in production

**Replication:**
- [ ] Use **read replicas** for read scaling (simplest, most impactful)
- [ ] Use **semi-synchronous** replication for durability without sacrificing availability
- [ ] Monitor **replication lag** — set SLO (e.g., < 1s p99)
- [ ] Implement **read-your-writes** consistency for post-write reads
- [ ] Plan **failover procedure** — automated promotion, DNS update, connection draining
- [ ] For multi-region: choose between **geo-partitioned leaders** and **global consensus** based on consistency needs
- [ ] For cross-service replication: use **event-carried state transfer** via Kafka/CDC

---

### Recommendation

**Start simple:** single database with **read replicas** handles most microservice workloads. Add partitioning only when a single node's write throughput or storage is genuinely exhausted. When you shard, choose a **partition key that matches your dominant query pattern** — this single decision determines whether sharding helps or hurts. Use **consistent hashing** (Cassandra, DynamoDB, CockroachDB) to avoid painful rebalancing. For replication, **semi-synchronous single-leader** is the right default for most OLTP workloads. Move to **multi-leader or leaderless** only for active-active multi-region or extreme availability requirements — and prepare for conflict resolution complexity. For cross-service data sharing, use **event-driven replication** (Kafka + event-carried state transfer) to keep services self-contained.

---

### Next Steps to Explore

1. **CockroachDB / Spanner** — globally distributed SQL with automatic partitioning and strong consistency
2. **Vitess / Citus** — sharding layers for MySQL / PostgreSQL
3. **CQRS read models** — purpose-built projections for cross-partition query patterns
4. **CRDTs** — conflict-free replicated data types for multi-leader / leaderless systems
5. **Chaos testing replication** — kill leaders, introduce network partitions, measure failover