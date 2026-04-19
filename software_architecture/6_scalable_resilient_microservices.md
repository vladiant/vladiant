# How do you ensure that Microservices are scalable and resilient?

Scalability and resilience are **different properties** that require different strategies but overlap in implementation. Systems that scale but aren't resilient collapse under partial failure. Systems that are resilient but don't scale hit capacity walls under growth.

| Property | Definition | Enemy |
|----------|-----------|-------|
| **Scalability** | Handle increasing load by adding resources without redesign | Bottlenecks, shared state, synchronous coupling |
| **Resilience** | Continue operating (possibly degraded) when components fail | Cascading failures, single points of failure, tight coupling |

---

## 1. Scalability: The Two Dimensions

```mermaid
graph TB
    subgraph "Scale Cube (AKF)"
        direction TB
        X["X-Axis: Horizontal Duplication<br/>Clone the service N times behind a load balancer"]
        Y["Y-Axis: Functional Decomposition<br/>Split by business capability (microservices!)"]
        Z["Z-Axis: Data Partitioning<br/>Shard by customer, region, tenant"]
    end
```

| Axis | What It Scales | Example | Limitation |
|------|---------------|---------|------------|
| **X-axis** | Throughput | 10 replicas of Order Service behind a load balancer | All replicas need access to the same data — DB becomes the bottleneck |
| **Y-axis** | Complexity / team throughput | Order Service split from Payment Service | Already done if you're in microservices |
| **Z-axis** | Data volume / multi-tenancy | Customer A → Shard 1, Customer B → Shard 2 | Cross-shard queries are expensive |

---

## 2. Scalability Patterns

### Option A: Stateless Services + Horizontal Scaling

```mermaid
graph LR
    LB[Load Balancer] --> S1[Service Instance 1]
    LB --> S2[Service Instance 2]
    LB --> S3[Service Instance 3]
    LB --> SN[Service Instance N]

    S1 --> CACHE[(Redis Cache)]
    S2 --> CACHE
    S3 --> CACHE
    SN --> CACHE

    S1 --> DB[(Database)]
    S2 --> DB
    S3 --> DB
    SN --> DB
```

| Criterion | Detail |
|-----------|--------|
| **Principle** | No in-process state — session/state externalized to Redis, DB, or object store |
| **Scaling trigger** | CPU, memory, or request queue depth via auto-scaler (HPA in K8s) |
| **Bottleneck** | Database — solved with read replicas, connection pooling, caching |
| **Complexity** | Low — just add instances |

### Option B: CQRS — Separate Read and Write Paths

```mermaid
graph TB
    subgraph "Write Path"
        CMD[Commands] --> WS[Write Service]
        WS --> WDB[(Write DB<br/>Postgres)]
    end

    subgraph "Event Propagation"
        WDB -- "CDC / Outbox" --> BUS[Event Bus]
    end

    subgraph "Read Path (Scale Independently)"
        BUS --> RS1[Read Service 1]
        BUS --> RS2[Read Service 2]
        RS1 --> RDB1[(Read DB<br/>Elasticsearch)]
        RS2 --> RDB2[(Read DB<br/>Redis Cache)]
    end

    Q[Queries] --> RS1
    Q --> RS2
```

| Criterion | Detail |
|-----------|--------|
| **When** | Read:write ratio > 10:1 — most systems are read-heavy |
| **Benefit** | Scale reads and writes independently; optimize read stores for query patterns |
| **Trade-off** | Eventual consistency between write and read models; added infrastructure |

### Option C: Event-Driven with Backpressure

```mermaid
graph LR
    P[Producers<br/>10K msgs/sec] --> Q[(Message Broker<br/>Kafka / RabbitMQ)]
    Q --> C1[Consumer Group<br/>3 instances]
    Q --> C2[Consumer Group<br/>5 instances]

    style Q fill:#ff9,stroke:#333
```

| Criterion | Detail |
|-----------|--------|
| **When** | Bursty traffic, async workloads, batch processing |
| **Benefit** | Broker absorbs spikes; consumers scale independently from producers |
| **Backpressure** | Consumer lag triggers auto-scaling; producers never overwhelmed by slow consumers |
| **Trade-off** | Eventual consistency; message ordering per partition only (Kafka) |

---

## 3. Resilience Patterns

### The Cascading Failure Problem

```mermaid
sequenceDiagram
    participant A as Service A
    participant B as Service B
    participant C as Service C (SLOW)

    A->>B: Request (timeout: 5s)
    B->>C: Request (timeout: 5s)
    Note over C: C is slow (10s response)
    C--xB: Timeout after 5s
    Note over B: B's thread pool filling up...
    B--xA: Timeout after 5s
    Note over A: A's thread pool filling up...
    Note over A,C: All services degraded — cascading failure
```

**One slow service brings down the entire chain.** Resilience patterns break this cascade.

### Pattern 1: Circuit Breaker

```mermaid
stateDiagram-v2
    [*] --> Closed
    Closed --> Open: Failure threshold exceeded<br/>(e.g., 50% errors in 10s)
    Open --> HalfOpen: Timeout expires<br/>(e.g., after 30s)
    HalfOpen --> Closed: Probe request succeeds
    HalfOpen --> Open: Probe request fails
    
    note right of Closed: Normal operation<br/>Requests pass through
    note right of Open: Fast-fail immediately<br/>No requests to downstream
    note right of HalfOpen: Allow one test request<br/>to check recovery
```

| Aspect | Detail |
|--------|--------|
| **Implementation** | Resilience4j (Java), Polly (.NET), or service mesh (Istio) |
| **Config** | Failure rate threshold, sliding window size, wait duration in open state |
| **Fallback** | Return cached data, default response, or graceful error |

### Pattern 2: Bulkhead Isolation

```mermaid
graph TB
    subgraph "Without Bulkhead"
        TP1[Shared Thread Pool: 100 threads]
        TP1 --> A[Calls to Service A]
        TP1 --> B[Calls to Service B — SLOW]
        TP1 --> C[Calls to Service C]
        Note1[Service B exhausts all 100 threads<br/>A and C starved]
    end
```

```mermaid
graph TB
    subgraph "With Bulkhead"
        TP_A[Pool A: 30 threads] --> A2[Calls to Service A]
        TP_B[Pool B: 40 threads] --> B2[Calls to Service B — SLOW]
        TP_C[Pool C: 30 threads] --> C2[Calls to Service C]
        Note2[Service B exhausts its 40 threads<br/>A and C unaffected]
    end
```

| Aspect | Detail |
|--------|--------|
| **Principle** | Isolate failure domains — one slow dependency can't saturate the entire service |
| **Types** | Thread pool isolation, semaphore isolation, process-level (sidecar) |
| **Implementation** | Resilience4j Bulkhead, Polly Bulkhead, separate connection pools per downstream |

### Pattern 3: Retry with Exponential Backoff + Jitter

```
Attempt 1: immediate
Attempt 2: wait 100ms + random(0-50ms)
Attempt 3: wait 200ms + random(0-100ms)
Attempt 4: wait 400ms + random(0-200ms)
→ Give up, trigger fallback
```

| Aspect | Detail |
|--------|--------|
| **Why jitter?** | Without jitter, all retries hit at the same instant → thundering herd |
| **Idempotency** | Only retry **idempotent** operations (GETs, operations with idempotency keys) |
| **Anti-pattern** | Retrying non-idempotent writes → duplicate orders, double charges |

### Pattern 4: Timeout Budgets

```mermaid
graph LR
    A["Service A<br/>Total budget: 3s"] --> B["Service B<br/>Budget: 1.5s"]
    B --> C["Service C<br/>Budget: 500ms"]
    B --> D["Service D<br/>Budget: 800ms"]
```

| Aspect | Detail |
|--------|--------|
| **Principle** | Propagate a **deadline** through the call chain; each service deducts its own processing time |
| **Implementation** | gRPC deadlines (built-in), custom `X-Request-Deadline` header for REST |
| **Benefit** | No service waits longer than the caller is willing to wait — prevents thread pool starvation |

### Pattern 5: Fallback Strategies

| Strategy | When | Example |
|----------|------|---------|
| **Cache fallback** | Downstream is unavailable | Serve last-known product prices from local cache |
| **Default value** | Non-critical data missing | Show "N/A" instead of failing the entire page |
| **Graceful degradation** | Feature partially unavailable | Show product listing without personalized recommendations |
| **Queue for retry** | Operation must eventually complete | Write to outbox, process when downstream recovers |

---

## 4. Infrastructure-Level Resilience

```mermaid
graph TB
    subgraph "Multi-AZ Deployment"
        subgraph "AZ-1"
            S1A[Service A]
            S1B[Service B]
        end
        subgraph "AZ-2"
            S2A[Service A]
            S2B[Service B]
        end
        subgraph "AZ-3"
            S3A[Service A]
            S3B[Service B]
        end
    end

    LB[Load Balancer<br/>Health-check aware] --> S1A
    LB --> S2A
    LB --> S3A

    subgraph "Data Layer"
        DB1[(Primary DB<br/>AZ-1)] -- "Sync replication" --> DB2[(Replica<br/>AZ-2)]
        DB1 -- "Sync replication" --> DB3[(Replica<br/>AZ-3)]
    end
```

| Pattern | What It Provides |
|---------|-----------------|
| **Multi-AZ deployment** | Survives entire datacenter failures |
| **Health checks + readiness probes** | Load balancer routes away from unhealthy instances |
| **Auto-scaling (HPA / KEDA)** | Scale on CPU, memory, queue depth, custom metrics |
| **PodDisruptionBudgets (K8s)** | Ensure minimum replicas during rolling updates or node drains |
| **Database replicas** | Read scaling + failover if primary goes down |
| **Multi-region (active-active)** | Survives entire region failures; adds latency and consistency complexity |

---

## 5. Comparison: Where to Invest First

| Strategy | Impact on Scalability | Impact on Resilience | Complexity | Priority |
|----------|----------------------|---------------------|------------|----------|
| Stateless services + auto-scaling | High | Medium | Low | **P0** |
| Circuit breakers | Low | **High** | Low | **P0** |
| Timeouts on every outbound call | Low | **High** | Low | **P0** |
| Health checks + readiness probes | Medium | **High** | Low | **P0** |
| Bulkhead isolation | Low | High | Medium | **P1** |
| Retry + backoff + jitter | Low | Medium | Low | **P1** |
| CQRS (separate read/write) | **High** | Medium | Medium-High | **P1** if read-heavy |
| Event-driven async | High | High | Medium | **P1** |
| Multi-AZ deployment | Medium | **High** | Medium | **P1** |
| Data partitioning / sharding | **High** | Medium | High | **P2** |
| Multi-region active-active | **High** | **High** | Very High | **P3** |

---

## 6. The Full Picture

```mermaid
graph TB
    CLIENT[Client] --> LB[Load Balancer<br/>Health-aware routing]

    subgraph "API Gateway"
        GW[Rate Limiting<br/>Auth<br/>Request ID]
    end

    LB --> GW

    subgraph "Service A (Stateless, Auto-scaled)"
        CB_B[Circuit Breaker → Service B]
        CB_C[Circuit Breaker → Service C]
        BH[Bulkhead: Isolated Pools]
        RETRY[Retry + Backoff]
        TIMEOUT[Timeout Budget]
        FALLBACK[Fallback: Cache / Default]
    end

    GW --> CB_B
    GW --> CB_C

    subgraph "Async Path"
        OUTBOX[Outbox Table] --> CDC[CDC Relay]
        CDC --> BROKER[(Kafka)]
        BROKER --> CONSUMERS[Consumer Group<br/>Auto-scaled on lag]
    end

    CB_B --> SB[Service B]
    CB_C --> SC[Service C]
    CB_B -.-> FALLBACK

    subgraph "Data Layer"
        DB_P[(Primary)] --> DB_R1[(Read Replica)]
        DB_P --> DB_R2[(Read Replica)]
        REDIS[(Redis Cache)]
    end
```

---

## 7. Anti-Patterns

| Anti-Pattern | Consequence |
|--------------|------------|
| No timeouts on outbound calls | One slow dependency locks up all threads |
| Retry storms without backoff/jitter | DDoS your own downstream service |
| Scaling vertically instead of horizontally | Single point of failure; ceiling on capacity |
| Shared in-memory state across instances | Can't scale horizontally; sticky sessions required |
| No circuit breakers | Cascading failures propagate in milliseconds |
| Health check that returns 200 always | Load balancer routes to broken instances |
| Auto-scaling on CPU only | Queue-based services need to scale on queue depth |
| Same SLA for all endpoints | Critical paths get degraded by bulk/reporting traffic |

---

## 8. Observability for Scalability & Resilience

You can't scale or recover from what you can't measure.

| Metric | Tells You | Alert Threshold |
|--------|----------|-----------------|
| **Request rate** (per service) | Traffic patterns, hotspots | Unexpected spikes (> 2× baseline) |
| **Error rate** (5xx) | Service health | > 1% of requests |
| **P99 latency** | Tail latency — the real user experience | > SLA target |
| **Circuit breaker state** | Downstream health | Any transition to OPEN |
| **Consumer lag** (Kafka) | Processing falling behind | Growing lag for > 5 min |
| **Thread pool saturation** | Bulkhead pressure | > 80% utilization |
| **Pod restart count** | OOM kills, crash loops | Any restart |

---

## 9. Next Steps

1. **What are your current scaling bottlenecks?** — CPU-bound? IO-bound? Database?
2. **What's your deployment platform?** — Kubernetes, ECS, VMs? HPA/KEDA availability matters.
3. **Have you experienced cascading failures?** — Knowing the failure mode shapes the priority.
4. **What are your SLA targets?** — 99.9% uptime? P99 < 200ms? This drives the investment level.
5. **Stateful services?** — Any services with in-memory state or sticky sessions that need refactoring?
