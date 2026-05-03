## Service Partitioning & Service Scaling in Microservices Architecture

### Context & Assumptions

**Service partitioning** is about decomposing workloads within a service so that different partitions handle different subsets of traffic (by tenant, geography, feature, or load). **Service scaling** is about adjusting compute capacity — horizontally (more instances) or vertically (bigger instances) — to meet demand. Together, they determine how your microservice architecture handles growth from 10 requests/second to 100,000 requests/second without re-architecture.

These are distinct from *data partitioning* (splitting database rows across shards). Here we focus on the **compute layer** — the service instances themselves.

---

### Core Concepts

| Concept | Definition |
|---|---|
| **Horizontal Scaling (Scale Out)** | Add more service instances behind a load balancer |
| **Vertical Scaling (Scale Up)** | Give existing instances more CPU/memory |
| **Service Partitioning** | Splitting a service into partitioned subsets that handle distinct traffic slices |
| **Auto-scaling** | Automatically adjusting instance count based on metrics |
| **Load Balancing** | Distributing requests across service instances |
| **Stateless Service** | Instance holds no client state — any instance can serve any request |
| **Stateful Service** | Instance holds client/session state — requests must route to specific instance |
| **Affinity / Sticky Sessions** | Routing related requests to the same instance |

---

## Part 1: Service Scaling

### Vertical vs. Horizontal Scaling

```mermaid
graph LR
    subgraph "Vertical Scaling (Scale Up)"
        V1[1 instance<br/>2 CPU, 4GB RAM<br/>500 req/s] -->|Scale up| V2[1 instance<br/>16 CPU, 64GB RAM<br/>4000 req/s]
    end

    subgraph "Horizontal Scaling (Scale Out)"
        H1[1 instance<br/>2 CPU, 4GB RAM<br/>500 req/s] -->|Scale out| H2[8 instances<br/>2 CPU, 4GB each<br/>4000 req/s total]
    end

    style V2 fill:#f9a825,stroke:#333,color:#000
    style H2 fill:#66bb6a,stroke:#333,color:#000
```

| Aspect | Vertical Scaling | Horizontal Scaling |
|---|---|---|
| **Mechanism** | Bigger machine | More machines |
| **Limit** | Hardware ceiling (largest available VM/server) | Virtually unlimited |
| **Downtime** | Usually requires restart | Zero-downtime (rolling) |
| **Cost curve** | Exponential (2× CPU ≠ 2× cost) | Linear (2× instances ≈ 2× cost) |
| **Failure blast radius** | Total — single instance dies, service is down | Partial — one instance dies, others serve |
| **State handling** | Simple — one instance | Must be stateless or use external state |
| **Complexity** | Low | Medium (load balancing, health checks, session management) |
| **Best for** | Databases, in-memory workloads, quick fix | Stateless services, web APIs, workers |

---

### Kubernetes Horizontal Pod Autoscaler (HPA)

```mermaid
graph TD
    subgraph "HPA Scaling Loop"
        METRICS[Metrics Server<br/>CPU, Memory, Custom] -->|Report| HPA[Horizontal Pod<br/>Autoscaler]
        HPA -->|"currentCPU=80%<br/>target=60%<br/>→ scale to 4 replicas"| DEPLOY[Deployment<br/>order-service]
        DEPLOY --> P1[Pod 1]
        DEPLOY --> P2[Pod 2]
        DEPLOY --> P3[Pod 3]
        DEPLOY --> P4[Pod 4 — NEW]
    end

    SVC[Service / Ingress] --> P1
    SVC --> P2
    SVC --> P3
    SVC --> P4

    style HPA fill:#42a5f5,stroke:#333,color:#fff
    style P4 fill:#66bb6a,stroke:#333,color:#000
```

**HPA scaling formula:**

$$\text{desiredReplicas} = \lceil \text{currentReplicas} \times \frac{\text{currentMetric}}{\text{targetMetric}} \rceil$$

Example: 3 replicas at 80% CPU, target 60%:

$$\text{desired} = \lceil 3 \times \frac{80}{60} \rceil = \lceil 4.0 \rceil = 4$$

---

### Scaling Metrics

```mermaid
graph TD
    subgraph "What to Scale On"
        CPU[CPU Utilization<br/>Default, simple] 
        MEM[Memory Utilization<br/>Good for caching services]
        RPS[Requests per Second<br/>Direct load indicator]
        LAT[Response Latency P99<br/>User experience proxy]
        QUEUE[Queue Depth<br/>Consumer lag]
        CUSTOM[Custom Business Metric<br/>Active orders, concurrent users]
        SCHED[Schedule / Cron<br/>Known traffic patterns]
    end

    style CPU fill:#66bb6a,stroke:#333,color:#000
    style RPS fill:#42a5f5,stroke:#333,color:#fff
    style QUEUE fill:#f9a825,stroke:#333,color:#000
```

| Metric | Scaling Type | Lag | Best For |
|---|---|---|---|
| **CPU utilization** | Reactive | 30-60s | General compute-bound services |
| **Memory utilization** | Reactive | 30-60s | Caching services, JVM heap pressure |
| **Requests/second (RPS)** | Reactive | 15-30s | API services with known per-instance capacity |
| **Latency (P99, P95)** | Reactive | 30-60s | Latency-sensitive user-facing services |
| **Queue depth / consumer lag** | Reactive | 10-30s | Async workers, event consumers |
| **Custom business metric** | Reactive | Varies | Domain-specific (active sessions, cart count) |
| **Schedule-based (cron)** | Predictive | Zero | Known patterns (sale events, business hours) |
| **ML-based predictive** | Predictive | Negative (ahead of load) | Complex traffic patterns with historical data |

---

### Multi-Dimensional Autoscaling

```mermaid
graph TD
    subgraph "Scaling Decision Engine"
        HPA_CPU[HPA: CPU<br/>target: 60%<br/>→ wants 4 replicas]
        HPA_RPS[HPA: RPS<br/>target: 200/pod<br/>→ wants 6 replicas]
        HPA_CUSTOM[HPA: Queue Depth<br/>target: 100 msgs<br/>→ wants 3 replicas]

        HPA_CPU --> MAX{Take Maximum}
        HPA_RPS --> MAX
        HPA_CUSTOM --> MAX

        MAX -->|"max(4,6,3) = 6"| DEPLOY[Scale to 6 replicas]
    end

    style MAX fill:#ef5350,stroke:#333,color:#fff
```

Kubernetes HPA with multiple metrics always takes the **maximum** recommended count — ensuring all SLOs are met.

---

### Vertical Pod Autoscaler (VPA)

```mermaid
graph TD
    subgraph "VPA — Right-sizing"
        VPA[Vertical Pod Autoscaler] -->|Analyze historical<br/>CPU/Memory usage| RECO[Recommendation:<br/>requests: cpu=250m, mem=384Mi<br/>limits: cpu=500m, mem=768Mi]
        RECO -->|Apply| POD[Pod: order-service<br/>Auto-updated resources]
    end

    style VPA fill:#ab47bc,stroke:#333,color:#fff
```

| Mode | Behavior | Disruption |
|---|---|---|
| **Off** | Recommendations only — no changes applied | None |
| **Auto** | Evicts pods and recreates with new resources | Pod restart |
| **Initial** | Sets resources only at creation time | None after start |

**Warning:** VPA and HPA generally conflict on the same metric (CPU). Use VPA for right-sizing requests/limits; HPA for replica count scaling.

---

### KEDA: Event-Driven Autoscaling

```mermaid
graph TD
    subgraph "KEDA — Scale to/from Zero"
        KAFKA[(Kafka Topic<br/>order.events<br/>Consumer lag: 5000)]
        KEDA[KEDA Scaler] -->|Monitor consumer lag| KAFKA
        KEDA -->|"lag=5000, threshold=500<br/>→ scale to 10 pods"| DEPLOY[Deployment:<br/>order-event-consumer]
        KEDA -.->|"lag=0<br/>→ scale to 0 pods"| DEPLOY
    end

    style KEDA fill:#42a5f5,stroke:#333,color:#fff
```

| KEDA Scaler | Source | Best For |
|---|---|---|
| Kafka | Consumer group lag | Event consumers |
| RabbitMQ | Queue length | Task workers |
| Prometheus | Custom query | Any Prometheus metric |
| Cron | Schedule | Predictive scaling |
| AWS SQS | Queue depth | AWS message processing |
| Redis | List length, stream pending | Job queues |
| HTTP | External API metric | External signals |

**Key advantage:** KEDA scales to **zero** — HPA has a minimum of 1 replica. Critical for cost optimization of event-driven workloads with intermittent traffic.

---

### Scaling Patterns by Service Type

```mermaid
graph TD
    subgraph "Scaling Strategy per Service Type"
        API[Stateless API Service<br/>HPA on CPU + RPS<br/>min=2, max=20]
        WORKER[Async Worker<br/>KEDA on queue depth<br/>min=0, max=50]
        GW[API Gateway<br/>HPA on connections<br/>min=3, max=10]
        CACHE[Cache Service<br/>VPA for right-sizing<br/>Horizontal via sharding]
        DB[Stateful Database<br/>Vertical scaling primary<br/>Read replicas for reads]
        CRON[Scheduled Job<br/>CronJob — K8s native<br/>No persistent replicas]
    end

    style API fill:#42a5f5,stroke:#333,color:#fff
    style WORKER fill:#66bb6a,stroke:#333,color:#000
    style GW fill:#f9a825,stroke:#333,color:#000
    style DB fill:#ef5350,stroke:#333,color:#fff
```

| Service Type | Primary Scaling | Metric | Min | Scale-to-Zero |
|---|---|---|---|---|
| **Stateless API** | HPA (horizontal) | CPU, RPS, latency | 2+ (HA) | No |
| **Event consumer / worker** | KEDA | Queue depth, consumer lag | 0 | Yes |
| **API Gateway** | HPA (horizontal) | Connections, RPS | 3+ (HA) | No |
| **Real-time (WebSocket)** | HPA + sticky sessions | Connections | 2+ | No |
| **Batch processor** | KEDA / CronJob | Schedule or job queue | 0 | Yes |
| **ML inference** | HPA on GPU utilization | GPU%, latency | 1+ | Yes (KEDA) |
| **Database** | Vertical + read replicas | CPU, IOPS, connections | N/A | No |

---

### Warm-up & Graceful Scaling

```mermaid
sequenceDiagram
    participant HPA as HPA Controller
    participant K8S as Kubernetes
    participant POD as New Pod
    participant LB as Service / LB

    HPA->>K8S: Scale to 4 replicas
    K8S->>POD: Create Pod
    POD->>POD: Container starts
    POD->>POD: Application initializes<br/>Load config, warm caches,<br/>establish DB connections

    Note over POD: Startup Probe passes ✅
    Note over POD: Readiness Probe passes ✅

    K8S->>LB: Add Pod to Endpoints
    LB->>POD: Start routing traffic

    Note over POD: Traffic ramps up gradually<br/>(slow-start in Envoy/Istio)
```

| Kubernetes Probe | Purpose in Scaling |
|---|---|
| **Startup Probe** | Wait for slow-starting apps (JVM, .NET) before liveness kicks in |
| **Readiness Probe** | Pod receives traffic only after it's ready to serve |
| **Liveness Probe** | Restart pod if it becomes unhealthy |

**Slow-start / warm-up:** Service mesh (Envoy/Istio) can gradually ramp traffic to new pods instead of immediately sending full load to a cold instance.

---

### Graceful Scale-Down

```mermaid
sequenceDiagram
    participant HPA as HPA Controller
    participant K8S as Kubernetes
    participant LB as Service / LB
    participant POD as Pod (being terminated)

    HPA->>K8S: Scale down to 3 replicas
    K8S->>LB: Remove Pod from Endpoints
    Note over LB: New requests stop<br/>going to this Pod

    K8S->>POD: SIGTERM
    POD->>POD: Stop accepting new requests
    POD->>POD: Drain in-flight requests<br/>(complete ongoing work)
    POD->>POD: Close DB connections
    POD->>POD: Flush metrics/logs

    Note over POD: terminationGracePeriodSeconds<br/>(default: 30s)

    POD->>K8S: Exit 0 ✅

    alt Grace period exceeded
        K8S->>POD: SIGKILL ☠️
    end
```

**Critical settings:**

| Setting | Purpose | Recommended |
|---|---|---|
| `terminationGracePeriodSeconds` | Time allowed for graceful shutdown | Match your longest request timeout |
| `preStop` hook | Delay before SIGTERM (allow LB to drain) | `sleep 5` to let endpoints update propagate |
| HPA `stabilizationWindowSeconds` | Prevent rapid scale down oscillation | 300s (scale-down), 0s (scale-up) |
| HPA `scaleDown.policies` | Rate limit scale-down | Max 1 pod per 60s, or 10% per minute |

---

## Part 2: Service Partitioning

### Service Partitioning Strategies

Service partitioning splits the **compute layer** so that different instances handle different subsets of work — beyond simple round-robin load balancing.

```mermaid
graph TD
    subgraph "Partitioning Strategies"
        T1[Tenant-Based<br/>Partition by customer/tenant]
        T2[Geographic<br/>Partition by region/location]
        T3[Functional<br/>Partition by use case or feature]
        T4[Priority / Tier<br/>Partition by traffic priority]
        T5[Shard-Based<br/>Partition by data shard affinity]
    end

    style T1 fill:#42a5f5,stroke:#333,color:#fff
    style T2 fill:#66bb6a,stroke:#333,color:#000
    style T3 fill:#f9a825,stroke:#333,color:#000
    style T4 fill:#ff7043,stroke:#333,color:#fff
    style T5 fill:#ab47bc,stroke:#333,color:#fff
```

---

#### 1. Tenant-Based Partitioning

```mermaid
graph TD
    subgraph "Tenant Partitioning"
        GW[API Gateway<br/>Route by tenant header]
        GW -->|"X-Tenant: enterprise-a"| POOL_E[Enterprise Pool<br/>8 pods, 4 CPU each<br/>Dedicated resources]
        GW -->|"X-Tenant: enterprise-b"| POOL_E
        GW -->|"X-Tenant: standard-*"| POOL_S[Standard Pool<br/>4 pods, 2 CPU each<br/>Shared]
        GW -->|"X-Tenant: free-*"| POOL_F[Free Tier Pool<br/>2 pods, 1 CPU each<br/>Rate limited]
    end

    POOL_E --> DB_E[(Dedicated DB<br/>Enterprise)]
    POOL_S --> DB_S[(Shared DB<br/>Standard tenants)]
    POOL_F --> DB_S

    style POOL_E fill:#ef5350,stroke:#333,color:#fff
    style POOL_S fill:#42a5f5,stroke:#333,color:#fff
    style POOL_F fill:#66bb6a,stroke:#333,color:#000
```

| Tenant Tier | Isolation | Resource Allocation | SLA |
|---|---|---|---|
| **Enterprise** | Dedicated pod pool + DB | Guaranteed CPU/memory, no contention | 99.99% |
| **Standard** | Shared pod pool, namespace quotas | Burstable resources | 99.9% |
| **Free** | Shared pool, strict rate limits | Best-effort, heavily throttled | 99.0% |

**Implementation:**
- Kubernetes: separate Deployments per tenant tier, with different resource requests/limits
- Service mesh (Istio): VirtualService routing by header to different destination subsets
- API Gateway (Kong/Envoy): route by tenant ID header or JWT claim

---

#### 2. Geographic Partitioning

```mermaid
graph TD
    subgraph "Geographic Service Partitioning"
        DNS[GeoDNS / Global LB<br/>Route by client location]
        DNS -->|US clients| US_SVC[US Service Cluster<br/>us-east-1]
        DNS -->|EU clients| EU_SVC[EU Service Cluster<br/>eu-west-1]
        DNS -->|APAC clients| AP_SVC[APAC Service Cluster<br/>ap-southeast-1]

        US_SVC --> US_DB[(US Database<br/>Leader for US data)]
        EU_SVC --> EU_DB[(EU Database<br/>Leader for EU data)]
        AP_SVC --> AP_DB[(APAC Database<br/>Leader for APAC data)]
    end

    US_DB <-->|Cross-region<br/>replication| EU_DB
    EU_DB <-->|Cross-region<br/>replication| AP_DB
    US_DB <-->|Cross-region<br/>replication| AP_DB

    style DNS fill:#f9a825,stroke:#333,color:#000
```

| Benefit | Trade-off |
|---|---|
| Lowest latency (serve from nearest region) | Cross-region data consistency is eventual |
| Data residency compliance (GDPR, data sovereignty) | Operational complexity of multi-region deployment |
| Blast radius isolation (regional failure) | Must handle cross-region requests (user traveling) |

---

#### 3. Functional Partitioning (Swimlanes)

```mermaid
graph TD
    subgraph "Functional Swimlanes"
        GW[API Gateway]
        GW -->|"Read traffic<br/>GET requests"| READ[Read Partition<br/>Optimized: many lightweight pods<br/>Caching layer, read replicas]
        GW -->|"Write traffic<br/>POST/PUT/DELETE"| WRITE[Write Partition<br/>Optimized: fewer, resourceful pods<br/>Transactional, leader DB]
        GW -->|"Bulk import<br/>Batch operations"| BATCH[Batch Partition<br/>Isolated: large pods, GPU<br/>Won't affect online traffic]
    end

    READ --> READ_DB[(Read Replicas)]
    WRITE --> WRITE_DB[(Primary DB)]
    BATCH --> BATCH_DB[(Separate DB instance<br/>or off-peak hours)]

    style READ fill:#66bb6a,stroke:#333,color:#000
    style WRITE fill:#42a5f5,stroke:#333,color:#fff
    style BATCH fill:#f9a825,stroke:#333,color:#000
```

| Swimlane | Characteristics | Scaling Strategy |
|---|---|---|
| **Read path** | High throughput, cacheable, idempotent | HPA on RPS; aggressive caching |
| **Write path** | Lower throughput, transactional, sequential | HPA on CPU; queue-based buffering |
| **Batch / import** | Bursty, resource-intensive, tolerate latency | KEDA on job queue; scale-to-zero |
| **Search / analytics** | CPU-intensive, fan-out queries | HPA on latency; separate cluster |

---

#### 4. Priority-Based Partitioning (Traffic Tiers)

```mermaid
graph TD
    subgraph "Priority-Based Partitioning"
        GW[API Gateway<br/>Classify by priority]
        GW -->|"Critical: checkout, payment"| CRIT["Critical Pool<br/>Over-provisioned<br/>PriorityClass: system-critical<br/>Dedicated nodes (taint)"]
        GW -->|"Standard: browse, search"| STD[Standard Pool<br/>Auto-scaled<br/>PriorityClass: high]
        GW -->|"Background: recommendations,<br/>analytics, reports"| BG[Background Pool<br/>Best-effort<br/>PriorityClass: low<br/>Preemptible]
    end

    style CRIT fill:#ef5350,stroke:#333,color:#fff
    style STD fill:#42a5f5,stroke:#333,color:#fff
    style BG fill:#66bb6a,stroke:#333,color:#000
```

Under resource pressure, Kubernetes **preempts** low-priority pods to make room for critical ones:

| Priority Tier | Resources | Under Pressure | Pod Disruption |
|---|---|---|---|
| **Critical** | Guaranteed QoS, dedicated nodes | Protected — never preempted | PDB: minAvailable=80% |
| **Standard** | Burstable QoS | May be throttled | PDB: minAvailable=50% |
| **Background** | BestEffort QoS, spot instances | Preempted first | No PDB (can be fully evicted) |

---

#### 5. Shard-Affinity Partitioning

```mermaid
graph TD
    subgraph "Shard-Affinity: Service instances pinned to data shards"
        ROUTER["Request Router<br/>hash(userId) → shard"]
        ROUTER -->|"Shard 0 users"| I1[Instance Group 0<br/>Connected to Shard 0 DB]
        ROUTER -->|"Shard 1 users"| I2[Instance Group 1<br/>Connected to Shard 1 DB]
        ROUTER -->|"Shard 2 users"| I3[Instance Group 2<br/>Connected to Shard 2 DB]

        I1 --> DB0[(Database Shard 0)]
        I2 --> DB1[(Database Shard 1)]
        I3 --> DB2[(Database Shard 2)]
    end

    style ROUTER fill:#f9a825,stroke:#333,color:#000
```

| Benefit | Trade-off |
|---|---|
| Connection pool efficiency (each instance connects to one shard) | Routing complexity |
| Better cache locality (instance caches only its shard's data) | Uneven load if shards are skewed |
| Reduced cross-shard queries | Must handle shard rebalancing at compute layer too |

---

### Load Balancing Strategies

```mermaid
graph TD
    subgraph "Load Balancing Algorithms"
        RR[Round Robin<br/>Simple rotation] 
        WRR[Weighted Round Robin<br/>Different capacity per instance]
        LC[Least Connections<br/>Route to least busy]
        RL[Random<br/>Simple, surprisingly effective]
        HASH[Consistent Hash<br/>Route by key → same instance]
        P2C[Power of Two Choices<br/>Pick best of 2 random candidates]
    end

    style RR fill:#66bb6a,stroke:#333,color:#000
    style LC fill:#42a5f5,stroke:#333,color:#fff
    style P2C fill:#ab47bc,stroke:#333,color:#fff
```

| Algorithm | When to Use | Statefulness | Hot Spot Risk |
|---|---|---|---|
| **Round Robin** | Uniform instances, stateless service | None | Low |
| **Weighted Round Robin** | Heterogeneous instances (different CPU/memory) | None | Low |
| **Least Connections** | Variable request duration | None | Low |
| **IP Hash** | Session affinity without cookies | Sticky | Medium |
| **Consistent Hash** | Cache affinity, stateful routing | Sticky | Medium |
| **Power of Two Choices** | High-throughput, latency-sensitive | None | Very Low |
| **Locality-Aware** | Multi-zone — prefer same-zone instances | None | Low |

---

### Scaling Stateful Services

Stateless services scale trivially — add instances behind a load balancer. **Stateful services require special handling:**

```mermaid
graph TD
    subgraph "Strategies for Stateful Scaling"
        S1[Externalize State<br/>Move to Redis/DB<br/>→ Service becomes stateless]
        S2[Sticky Sessions<br/>Route same user to same instance<br/>via cookie/header hash]
        S3[StatefulSet<br/>K8s: stable network ID,<br/>persistent volume per pod]
        S4[Distributed State<br/>CRDTs, Akka Cluster,<br/>Orleans virtual actors]
        S5[Partition State<br/>Each instance owns a<br/>subset of the state space]
    end

    style S1 fill:#66bb6a,stroke:#333,color:#000
    style S5 fill:#42a5f5,stroke:#333,color:#fff
```

| Strategy | Complexity | Scalability | Failover | Best For |
|---|---|---|---|---|
| **Externalize state** | Low | Highest | Seamless | ✅ Default recommendation |
| **Sticky sessions** | Low | Medium | Session lost on instance death | WebSocket, short-lived sessions |
| **StatefulSet** | Medium | Medium | Volume survives, pod restart | Databases, message brokers |
| **Distributed state** | High | High | Automatic rebalancing | Real-time games, actor systems |
| **Partitioned state** | High | High | Must re-partition on failure | Event sourcing, stream processing |

---

### Cost-Optimized Scaling

```mermaid
graph TD
    subgraph "Cost Optimization"
        BASE[Baseline: Reserved/Committed<br/>instances for steady load] -->|Always running| MIN[min replicas: 3<br/>On-demand / reserved pricing]
        BURST[Burst: On-demand<br/>instances for peaks] -->|Scale up| MED[Scale to 10 pods<br/>On-demand pricing]
        SPIKE[Spike: Spot/Preemptible<br/>instances for non-critical] -->|Overflow| MAX[Scale to 20 pods<br/>Spot pricing — 60-70% cheaper]
    end

    style BASE fill:#66bb6a,stroke:#333,color:#000
    style BURST fill:#42a5f5,stroke:#333,color:#fff
    style SPIKE fill:#f9a825,stroke:#333,color:#000
```

| Instance Type | Cost | Availability | Best For |
|---|---|---|---|
| **Reserved / Committed** | Lowest (1-3 year) | Guaranteed | Baseline steady-state load |
| **On-demand** | Standard | Guaranteed | Variable production workloads |
| **Spot / Preemptible** | 60-90% discount | Can be reclaimed with 2min notice | Batch, stateless overflow, dev/test |
| **Scale-to-Zero (KEDA)** | Zero when idle | Cold start latency | Event consumers, scheduled jobs |

**Mixed strategy:** Run baseline on reserved instances, burst to on-demand, overflow non-critical to spot — can reduce compute cost by 40-60%.

---

### Scaling Anti-Patterns

| Anti-Pattern | Problem | Remedy |
|---|---|---|
| **Scaling on memory with leak** | More instances just delays OOM; masks the bug | Fix the memory leak; use VPA for right-sizing |
| **Scaling CPU without profiling** | May addmore pods but bottleneck is DB or external API | Profile first; scale the bottleneck (Amdahl's Law) |
| **No max replica limit** | Runaway scaling → exhausts cluster, huge cloud bill | Always set `maxReplicas`; alert on reaching max |
| **Aggressive scale-down** | Scale down too fast → oscillation (scale up again immediately) | `stabilizationWindowSeconds: 300` on scale-down |
| **Scaling stateful services like stateless** | Data loss, split-brain, inconsistent state | Externalize state or use StatefulSet with proper PVCs |
| **Same scaling config everywhere** | Checkout service and analytics service have same HPA | Tune per service based on criticality, traffic pattern, and cost |
| **Ignoring cold start** | New pods not ready for 30s → latency spike during scale-out | Readiness probes, slow-start in mesh, pre-warm strategies |
| **No PodDisruptionBudget** | Scale-down + node maintenance → too many pods killed simultaneously | PDB: `minAvailable: 50%` or `maxUnavailable: 1` |
| **Scaling partitions unevenly** | One tenant pool has 20 pods, another is starved | Monitor per-partition utilization; auto-scale each independently |
| **Premature partitioning** | Splitting service into pools before traffic justifies it | Start with shared pool + load balancing; partition when isolation needed |

---

### Capacity Planning

```mermaid
graph TD
    subgraph "Capacity Planning Process"
        M1[Measure<br/>Current throughput per pod<br/>at target latency SLO] --> M2[Model<br/>Expected traffic growth<br/>Seasonal patterns]
        M2 --> M3[Calculate<br/>Required pods =<br/>peak RPS / per-pod capacity<br/>+ headroom]
        M3 --> M4[Validate<br/>Load test at projected scale<br/>Verify bottlenecks]
        M4 --> M5[Automate<br/>HPA/KEDA for reactive<br/>Schedule for predictive]
        M5 --> M1
    end

    style M1 fill:#42a5f5,stroke:#333,color:#fff
    style M4 fill:#ef5350,stroke:#333,color:#fff
```

**Capacity formula:**

$$\text{minReplicas} = \left\lceil \frac{\text{peakRPS}}{\text{rpsPerPod}} \times \text{headroom} \right\rceil + \text{haBuffer}$$

Example: Peak = 5000 RPS, each pod handles 500 RPS, 1.3× headroom, 2 pods HA buffer:

$$\text{min} = \lceil \frac{5000}{500} \times 1.3 \rceil + 2 = 13 + 2 = 15 \text{ pods}$$

---

### Monitoring Scaling Health

| Metric | Alert Threshold | Meaning |
|---|---|---|
| `hpa_current_replicas / hpa_max_replicas` | > 80% | Approaching scaling ceiling |
| `pod_restart_count` | > 3 in 5min | OOM kills or crash loops during scaling |
| `pod_scheduling_latency` | > 30s | Cluster lacks capacity; need more nodes |
| `request_queue_depth` | Growing | Pods can't keep up — need to scale faster |
| `hpa_scaling_events` | > 10/hour | Oscillation — tune stabilization window |
| `pod_readiness_duration` | > 30s | Slow startup — cold-start issue |
| `cluster_node_utilization` | > 80% | Add nodes or enable cluster autoscaler |
| `spot_instance_interruptions` | > 0 | Spot instances being reclaimed — ensure graceful handling |

---

### Decision Framework

```mermaid
graph TD
    Q1{Scaling need?} -->|Read throughput| RR[Add read replicas<br/>or cache layer]
    Q1 -->|Write throughput| WS[Shard data +<br/>horizontally scale writers]
    Q1 -->|Compute throughput| Q2{Service stateful?}

    Q2 -->|Stateless| HPA_Q[HPA on CPU/RPS<br/>Standard horizontal scaling]
    Q2 -->|Stateful| Q3{Can externalize state?}
    Q3 -->|Yes| EXT[Move state to Redis/DB<br/>→ Scale as stateless]
    Q3 -->|No| SS[StatefulSet + partition<br/>or distributed state framework]

    HPA_Q --> Q4{Need isolation?}
    Q4 -->|Yes — tenants| TENANT[Tenant-based partitioning<br/>Separate pools per tier]
    Q4 -->|Yes — criticality| PRIO[Priority-based partitioning<br/>Critical/standard/background pools]
    Q4 -->|Yes — geography| GEO[Geographic partitioning<br/>Regional clusters]
    Q4 -->|No| SHARED[Shared pool with HPA<br/>Simplest approach]

    style HPA_Q fill:#66bb6a,stroke:#333,color:#000
    style SHARED fill:#66bb6a,stroke:#333,color:#000
    style TENANT fill:#42a5f5,stroke:#333,color:#fff
```

---

### Practical Checklist

**Scaling:**
- [ ] Make all API services **stateless** — externalize state to Redis, DB, or object store
- [ ] Configure **HPA** on every production Deployment — CPU + at least one business metric
- [ ] Set **`minReplicas ≥ 2`** for HA (survive single pod failure)
- [ ] Set **`maxReplicas`** with alert at 80% — prevent runaway scaling
- [ ] Use **KEDA** for event-driven workers — scale-to-zero when no messages
- [ ] Configure **PodDisruptionBudget** — prevent mass eviction during scaling/maintenance
- [ ] Set **readiness probes** — prevent routing traffic to unready pods during scale-out
- [ ] Tune **scale-down stabilization** (300s) — prevent oscillation
- [ ] Use **preStop hooks** (`sleep 5`) — allow endpoint removal to propagate before SIGTERM
- [ ] Enable **Cluster Autoscaler** — scale nodes when pods are unschedulable
- [ ] **Load test** at 2× projected peak — validate scaling behavior before production

**Partitioning:**
- [ ] Start with **shared pool** — partition only when isolation is required
- [ ] For multi-tenant SaaS: **tenant-tier partitioning** (enterprise/standard/free pools)
- [ ] For global services: **geographic partitioning** with regional clusters
- [ ] For mixed workloads: **functional swimlanes** (read/write/batch pools)
- [ ] For critical paths: **priority partitioning** with PriorityClass and dedicated nodes
- [ ] Monitor **per-partition utilization** — auto-scale each partition independently
- [ ] Set **resource quotas** per namespace to prevent one partition from starving others

---

### Recommendation

**Default to stateless horizontal scaling with HPA** — this handles 90% of microservice scaling needs. Externalize all state to managed databases or Redis. Use **CPU + RPS** as HPA metrics for API services; **KEDA + queue depth** for event consumers. Add **service partitioning** only when you need **tenant isolation** (SaaS tiers), **geographic locality** (latency/compliance), or **priority isolation** (protect checkout from analytics). Always set minimum replicas ≥ 2, maximum with an alert, and PodDisruptionBudgets. Budget for **Cluster Autoscaler** to add nodes when pod scaling demands exceed current cluster capacity. Optimize cost with a mix of reserved instances (baseline) and spot instances (burst/non-critical).

---

### Next Steps to Explore

1. **KEDA deep-dive** — event-driven autoscaling with custom scalers
2. **Cluster Autoscaler + Karpenter** — node-level scaling for pod demand
3. **Multi-cluster federation** — scaling beyond a single Kubernetes cluster
4. **Load testing for scaling validation** — k6, Locust, Gatling at production scale
5. **Cell-based architecture** — the ultimate service partitioning pattern for massive scale