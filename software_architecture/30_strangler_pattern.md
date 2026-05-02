

## Strangler (Fig) Pattern in Microservices Architecture

### Context & Assumptions

The **Strangler Fig pattern** (named after strangler fig trees that grow around a host tree, eventually replacing it) is an **incremental migration strategy** for replacing a legacy monolithic system with microservices — without a risky "big bang" rewrite. You build new functionality as microservices around the monolith, gradually routing traffic away from the old system to the new services, until the monolith is fully replaced and can be decommissioned.

Martin Fowler coined the term in 2004. It remains the **safest and most proven** approach for monolith-to-microservices migration.

---

### The Problem: Big Bang Rewrite

```mermaid
graph TD
    subgraph "Big Bang Rewrite — High Risk"
        M[Legacy Monolith<br/>5 years of features] -->|"Freeze features<br/>Rewrite everything<br/>12-18 months"| NEW[New System<br/>Must replicate ALL features]
        NEW -->|"Cutover day:<br/>Flip the switch"| PROD[Production]
    end

    style M fill:#ef5350,stroke:#333,color:#fff
    style NEW fill:#ef5350,stroke:#333,color:#fff
```

| Problem | Impact |
|---|---|
| **Feature freeze during rewrite** | Business cannot evolve for months/years |
| **Must replicate all features perfectly** | Unknown behaviors, undocumented edge cases, tribal knowledge lost |
| **All-or-nothing cutover** | One defect on launch day → rollback to old system → months wasted |
| **Team morale** | Year-long rewrite with no production value delivered |
| **Second system effect** | Over-engineering the replacement; scope creep |
| **Business context changes** | By the time rewrite is done, requirements have shifted |

**Historical failure rate of big bang rewrites: extremely high.** Netscape's browser rewrite is the canonical cautionary tale.

---

### The Solution: Strangler Fig Pattern

```mermaid
graph TD
    subgraph "Phase 1: Identify & Extract"
        C1[Clients] --> PROXY1[Facade / Proxy]
        PROXY1 -->|"90% of routes"| M1[Monolith<br/>All features]
        PROXY1 -->|"10% — new feature"| MS1[Microservice 1<br/>Product Catalog]
    end

    subgraph "Phase 2: Expand"
        C2[Clients] --> PROXY2[Facade / Proxy]
        PROXY2 -->|"60% of routes"| M2[Monolith<br/>Shrinking]
        PROXY2 -->|"20%"| MS2A[Service: Catalog]
        PROXY2 -->|"10%"| MS2B[Service: Orders]
        PROXY2 -->|"10%"| MS2C[Service: Users]
    end

    subgraph "Phase 3: Replace"
        C3[Clients] --> PROXY3[Facade / Proxy]
        PROXY3 -->|"5% — legacy edge cases"| M3[Monolith<br/>Almost empty]
        PROXY3 -->|"95%"| MS3[Microservices<br/>Full coverage]
    end

    subgraph "Phase 4: Decommission"
        C4[Clients] --> PROXY4[Facade / Proxy]
        PROXY4 -->|"100%"| MS4[Microservices<br/>Complete]
        M4[Monolith<br/>Decommissioned ☠️]
    end

    style M1 fill:#ef5350,stroke:#333,color:#fff
    style M2 fill:#f9a825,stroke:#333,color:#000
    style M3 fill:#f9a825,stroke:#333,color:#000
    style M4 fill:#9e9e9e,stroke:#333,color:#fff
    style MS4 fill:#66bb6a,stroke:#333,color:#000
```

---

### Core Mechanism: The Facade / Proxy

```mermaid
graph LR
    subgraph "Strangler Facade"
        CLIENT[Client] --> PROXY[API Gateway / Reverse Proxy<br/>NGINX / Envoy / Kong / Traefik]
        
        PROXY -->|"/api/products/*"| MS_PROD[Product Service<br/>NEW ✅]
        PROXY -->|"/api/orders/*"| MS_ORD[Order Service<br/>NEW ✅]
        PROXY -->|"/api/users/*"| MONO[Monolith<br/>LEGACY — still handles users]
        PROXY -->|"/api/inventory/*"| MONO
        PROXY -->|"/api/reports/*"| MONO
    end

    style PROXY fill:#42a5f5,stroke:#333,color:#fff
    style MS_PROD fill:#66bb6a,stroke:#333,color:#000
    style MS_ORD fill:#66bb6a,stroke:#333,color:#000
    style MONO fill:#ef5350,stroke:#333,color:#fff
```

The proxy is the **strangler vine** — it intercepts all traffic and **routes** each request to either the new microservice or the legacy monolith. As more routes shift to microservices, the monolith receives less traffic until it receives none.

---

### Migration Execution Flow

```mermaid
sequenceDiagram
    participant C as Client
    participant GW as API Gateway<br/>(Strangler Facade)
    participant NEW as New Product Service
    participant OLD as Monolith

    Note over GW: Route table:<br/>/api/products → NEW<br/>/api/orders → OLD<br/>/api/users → OLD

    C->>GW: GET /api/products/123
    GW->>NEW: Forward to Product Service
    NEW-->>GW: Product details
    GW-->>C: Response (from new service)

    C->>GW: POST /api/orders
    GW->>OLD: Forward to Monolith
    OLD-->>GW: Order created
    GW-->>C: Response (from legacy)

    Note over GW: Week later: deploy Order Service<br/>Update route: /api/orders → NEW

    C->>GW: POST /api/orders
    GW->>NEW: Forward to Order Service ✅
    NEW-->>GW: Order created
    GW-->>C: Response (from new service)
```

---

### Three Strangler Strategies

#### 1. Route-Based (URL/Path)

```mermaid
graph TD
    subgraph "Route-Based Strangling"
        GW[API Gateway] -->|"/products/*"| NEW1[Product Service ✅]
        GW -->|"/orders/*"| NEW2[Order Service ✅]
        GW -->|"/users/*"| OLD[Monolith]
        GW -->|"Everything else"| OLD
    end

    style NEW1 fill:#66bb6a,stroke:#333,color:#000
    style NEW2 fill:#66bb6a,stroke:#333,color:#000
    style OLD fill:#ef5350,stroke:#333,color:#fff
```

**Best when:** API routes map cleanly to bounded contexts.

#### 2. Feature-Based (Functionality)

```mermaid
graph TD
    subgraph "Feature-Based Strangling"
        GW[API Gateway] -->|"Search features"| NEW1[Search Service ✅]
        GW -->|"Checkout features"| NEW2[Checkout Service ✅]
        GW -->|"Account features"| OLD[Monolith]
    end

    style NEW1 fill:#66bb6a,stroke:#333,color:#000
    style NEW2 fill:#66bb6a,stroke:#333,color:#000
```

**Best when:** Features cross multiple URL paths but form a cohesive domain.

#### 3. Traffic-Based (Percentage / Canary)

```mermaid
graph TD
    subgraph "Traffic-Based Strangling"
        GW[API Gateway] -->|"90% /orders/"| OLD[Monolith Orders]
        GW -->|"10% /orders/"| NEW[New Order Service<br/>Canary]
    end

    style NEW fill:#66bb6a,stroke:#333,color:#000
    style OLD fill:#ef5350,stroke:#333,color:#fff
```

**Best when:** Validating the new service under production load before full cutover.

---

### Data Migration Strategy

This is the **hardest part** of the strangler pattern — the monolith's database contains all the data, and the new microservices need their own databases.

```mermaid
graph TD
    subgraph "Phase 1: Shared Database (Temporary)"
        MS1[New Product Service] -->|Read/Write| MONO_DB[(Monolith DB<br/>Products table)]
        MONO[Monolith] -->|Read/Write| MONO_DB
    end

    subgraph "Phase 2: CDC + Dual Read"
        MS2[New Product Service] -->|Write| NEW_DB[(Product Service DB)]
        MS2 -->|Read fallback| MONO_DB2[(Monolith DB)]
        CDC[Debezium CDC] -->|Sync changes| NEW_DB
        MONO2[Monolith] -->|Read/Write| MONO_DB2
    end

    subgraph "Phase 3: Independent Database"
        MS3[Product Service] -->|Read/Write| PROD_DB[(Product DB<br/>Owned exclusively)]
        MONO3[Monolith] -->|API call| MS3
    end

    style MONO_DB fill:#ef5350,stroke:#333,color:#fff
    style PROD_DB fill:#66bb6a,stroke:#333,color:#000
```

| Phase | Data Ownership | Risk | Duration |
|---|---|---|---|
| **Shared DB** | Monolith owns schema; service reads/writes same tables | Schema change breaks both | Weeks (keep as short as possible) |
| **CDC + Sync** | Service has own DB; CDC syncs from monolith | Data lag; eventual consistency | Weeks-months |
| **Database per service** | Service fully owns its data; monolith calls service API | API dependency | Target state |

---

### Data Synchronization During Migration

```mermaid
sequenceDiagram
    participant OLD_DB as Monolith DB
    participant CDC as CDC / Debezium
    participant NEW_DB as Service DB
    participant NEW as New Service
    participant OLD as Monolith

    Note over OLD_DB,NEW_DB: Phase: Dual-write transition

    OLD->>OLD_DB: Write product update
    CDC->>OLD_DB: Capture change from WAL
    CDC->>NEW_DB: Replicate to new DB

    NEW->>NEW_DB: Read product (local)
    Note over NEW: Uses local DB — self-contained

    Note over OLD_DB,NEW_DB: Phase: Cutover — new service is authoritative

    NEW->>NEW_DB: Write product update
    NEW->>OLD_DB: Sync back to monolith DB<br/>(temporary, for other monolith features)

    Note over OLD_DB,NEW_DB: Phase: Final — monolith no longer uses products table

    NEW->>NEW_DB: Full ownership ✅
    OLD_DB->>OLD_DB: Drop products table
```

---

### Identifying What to Extract First

```mermaid
graph TD
    subgraph "Extraction Priority Matrix"
        direction TB
        Q1{High business value<br/>+ frequent changes?} -->|Yes| FIRST[Extract FIRST<br/>Maximum business impact]
        Q1 -->|No| Q2{Well-defined boundary<br/>+ low coupling?}
        Q2 -->|Yes| SECOND[Extract SECOND<br/>Easy win, build confidence]
        Q2 -->|No| Q3{Performance bottleneck?<br/>Needs independent scaling?}
        Q3 -->|Yes| THIRD[Extract THIRD<br/>Operational necessity]
        Q3 -->|No| LAST[Extract LAST<br/>or leave in monolith]
    end

    style FIRST fill:#ef5350,stroke:#333,color:#fff
    style SECOND fill:#66bb6a,stroke:#333,color:#000
    style THIRD fill:#42a5f5,stroke:#333,color:#fff
    style LAST fill:#9e9e9e,stroke:#333,color:#fff
```

**Extraction prioritization criteria:**

| Factor | Weight | High Score (Extract First) | Low Score (Extract Later) |
|---|---|---|---|
| **Business value / change frequency** | High | Feature changes weekly | Stable for years |
| **Bounded context clarity** | High | Clear domain boundary | Deeply entangled |
| **Data coupling** | Critical | Own tables, few JOINs | Shared tables everywhere |
| **Team ownership** | Medium | Dedicated team ready | Shared across teams |
| **Scaling needs** | Medium | Needs independent scaling | Scales with monolith fine |
| **Technical debt** | Low-Medium | Code is unmaintainable | Code is acceptable |
| **External dependencies** | Low | Few external integrations | Heavily integrated |

---

### Complete Strangler Architecture

```mermaid
graph TD
    subgraph "Edge"
        CDN[CDN] --> LB[Load Balancer]
    end

    subgraph "Strangler Facade"
        LB --> GW[API Gateway<br/>Route Table]
    end

    subgraph "New Microservices"
        GW -->|"/api/products"| S1[Product Service]
        GW -->|"/api/orders"| S2[Order Service]
        GW -->|"/api/search"| S3[Search Service]
        S1 --> DB1[(Product DB)]
        S2 --> DB2[(Order DB)]
        S3 --> ES[(Elasticsearch)]
    end

    subgraph "Legacy (Shrinking)"
        GW -->|"/api/users"| MONO[Monolith]
        GW -->|"/api/reports"| MONO
        GW -->|"/api/admin"| MONO
        MONO --> MONO_DB[(Monolith DB)]
    end

    subgraph "Data Sync"
        MONO_DB -->|CDC| DEB[Debezium]
        DEB --> KAFKA[(Kafka)]
        KAFKA --> S1
        KAFKA --> S2
        KAFKA --> S3
    end

    subgraph "Anti-Corruption Layer"
        S2 -->|ACL: Translate<br/>legacy formats| MONO
    end

    style GW fill:#42a5f5,stroke:#333,color:#fff
    style MONO fill:#ef5350,stroke:#333,color:#fff
    style S1 fill:#66bb6a,stroke:#333,color:#000
    style S2 fill:#66bb6a,stroke:#333,color:#000
    style S3 fill:#66bb6a,stroke:#333,color:#000
```

---

### Anti-Corruption Layer (ACL)

When new microservices must still communicate with the monolith, an **Anti-Corruption Layer** prevents legacy data models and idioms from leaking into the new clean domain model.

```mermaid
graph LR
    subgraph "New Service"
        OS[Order Service<br/>Clean domain model]
        ACL[Anti-Corruption Layer<br/>Translator / Adapter]
    end

    subgraph "Legacy"
        MONO[Monolith<br/>Legacy data model]
    end

    OS -->|Domain objects| ACL
    ACL -->|Legacy format<br/>XML, SOAP, legacy IDs| MONO
    MONO -->|Legacy response| ACL
    ACL -->|Clean domain objects| OS

    style ACL fill:#f9a825,stroke:#333,color:#000
```

| ACL Responsibility | Example |
|---|---|
| **Data format translation** | JSON ↔ XML, REST ↔ SOAP |
| **ID mapping** | New UUID ↔ legacy integer PKs |
| **Schema mapping** | `customer.firstName` ↔ `CUST_FNAME` |
| **Error translation** | Legacy error codes → domain exceptions |
| **Protocol bridging** | gRPC → legacy HTTP/1.0 |

---

### Verification: Parallel Run (Shadow Traffic)

Before cutting over, verify the new service produces the **same results** as the monolith:

```mermaid
sequenceDiagram
    participant C as Client
    participant GW as API Gateway
    participant OLD as Monolith
    participant NEW as New Service
    participant CMP as Comparator

    C->>GW: GET /api/products/123
    
    par Parallel execution
        GW->>OLD: Forward (primary — returns response to client)
        OLD-->>GW: Response A

        GW->>NEW: Shadow (silent — response NOT returned to client)
        NEW-->>GW: Response B
    end

    GW-->>C: Response A (from monolith)

    GW->>CMP: Compare(Response A, Response B)
    CMP->>CMP: Log differences
    
    Note over CMP: "price field differs:<br/>monolith=$29.99, new=$30.00"<br/>→ Bug in new service
```

| Technique | Description | Use When |
|---|---|---|
| **Shadow traffic (dark launch)** | Send copy of production traffic to new service; discard its response | Validating read operations |
| **Parallel run with comparison** | Run both, compare results, return old response | High-stakes migration (financial) |
| **Canary with monitoring** | Route 1-5% of real traffic; monitor error rates and latency | Validating under real load |
| **Feature flag toggle** | Flip between old/new per request, user, or percentage | Instant rollback capability |

---

### Timeline: Realistic Migration Phases

```mermaid
gantt
    title Strangler Fig Migration — 18 Month Example
    dateFormat  YYYY-MM
    axisFormat  %Y-%m

    section Foundation
    Set up API Gateway / Proxy           :done, 2025-01, 2025-02
    Establish CI/CD for microservices     :done, 2025-01, 2025-03
    Set up observability stack            :done, 2025-02, 2025-03

    section Wave 1 — Easy Wins
    Extract Product Catalog Service       :active, 2025-03, 2025-05
    Extract Search Service                :active, 2025-04, 2025-06
    Shadow traffic + verify               :2025-05, 2025-06
    Cutover products + search             :milestone, 2025-06, 0d

    section Wave 2 — Core Business
    Extract Order Service + data migration :2025-06, 2025-09
    Extract Payment Service               :2025-07, 2025-10
    Parallel run + verify                 :2025-09, 2025-10
    Cutover orders + payments             :milestone, 2025-10, 0d

    section Wave 3 — Remaining
    Extract User/Auth Service             :2025-10, 2025-12
    Extract Reporting                     :2025-11, 2026-01
    Extract Admin                         :2026-01, 2026-03

    section Decommission
    Remove monolith routes                :2026-03, 2026-04
    Archive monolith                      :2026-05, 2026-06
    Decommission monolith DB              :milestone, 2026-06, 0d
```

---

### Anti-Patterns

| Anti-Pattern | Problem | Remedy |
|---|---|---|
| **Extract everything at once** | Becomes a big bang rewrite in disguise | One bounded context at a time; deliver value incrementally |
| **No strangler facade** | Services added but no routing mechanism → two systems, neither complete | Set up API gateway/proxy FIRST — it's the foundation |
| **Keep the monolith DB** | All services share the monolith DB → distributed monolith | Migrate data ownership per service; use CDC for transition |
| **Never decommissioning the monolith** | "Temporary" shared state becomes permanent → two systems to maintain | Set decommission milestones; track remaining routes |
| **Extracting the hardest part first** | Team loses momentum on gnarly domain; no quick wins | Start with well-bounded, high-value, low-coupling domains |
| **No anti-corruption layer** | Legacy data model infects new services | ACL isolates legacy format; new services use clean domain model |
| **No verification (shadow/canary)** | Cutover without verifying parity → production incidents | Shadow traffic or parallel run before any cutover |
| **Feature freeze on monolith** | Business can't evolve during migration → stakeholder pressure → abort | Continue delivering features in monolith while extracting; strangler allows both |
| **Microservice calls monolith DB directly** | Hidden coupling; monolith schema change breaks service | API or event-based communication only; no shared DB access |
| **No rollback plan** | Cutover to new service fails → no way back | Feature flag in gateway to instantly revert route to monolith |

---

### Measuring Migration Progress

| Metric | How to Measure | Target |
|---|---|---|
| **Route coverage** | % of API routes handled by microservices vs. monolith | 100% microservices |
| **Traffic percentage** | % of requests going to new services vs. monolith | Increasing over time |
| **Monolith code churn** | Lines of code changed in monolith per month | Approaching zero |
| **Monolith deployment frequency** | How often monolith is deployed | Decreasing |
| **Feature delivery velocity** | Time from idea to production for extracted domains | Faster than monolith era |
| **Monolith DB table count** | Tables still accessed by monolith | Decreasing |
| **Incident rate** | Production incidents related to migrated domains | Lower than monolith era |

```mermaid
graph LR
    subgraph "Migration Progress Dashboard"
        M1[Route Coverage<br/>65% microservices] --> GRAF[Grafana]
        M2[Traffic Split<br/>70% new / 30% legacy] --> GRAF
        M3[Monolith Deploys<br/>2/week → 1/month] --> GRAF
        M4[Services Extracted<br/>8 of 12 bounded contexts] --> GRAF
    end

    style GRAF fill:#ff7043,stroke:#333,color:#fff
```

---

### Practical Checklist

- [ ] **Analyze monolith** — map bounded contexts, dependencies, data ownership
- [ ] **Set up strangler facade** (API gateway) as the first step — before any extraction
- [ ] **Start with easy wins** — well-bounded, high-value, low-coupling domains
- [ ] **Build anti-corruption layer** — isolate new services from legacy data models
- [ ] **Migrate data incrementally** — shared DB → CDC sync → independent DB
- [ ] **Verify before cutover** — shadow traffic, parallel run, or canary with comparison
- [ ] **Feature flag the route** — instantly revert to monolith if new service fails
- [ ] **Continue delivering features** in monolith during migration — no feature freeze
- [ ] **Decommission route by route** — track remaining monolith routes; set milestones
- [ ] **Monitor both systems** — compare latency, error rates, correctness between old and new
- [ ] **Communicate progress** — dashboard showing migration percentage to stakeholders
- [ ] **Plan monolith decommission date** — avoid permanent dual-system maintenance
- [ ] **Celebrate each extraction** — team morale matters during long migrations

---

### Recommendation

The Strangler Fig pattern is the **only proven safe approach** to migrating from a monolith to microservices. Never attempt a big bang rewrite. Set up the **API gateway/proxy first** — this is non-negotiable, as it's the mechanism that enables incremental routing. Extract services starting with **well-bounded, high-value contexts** (not the hardest or most entangled — save those for when the team has experience). Use **CDC (Debezium)** for data synchronization during the transition period, and always verify parity through **shadow traffic or parallel runs** before cutting over. Plan for **18-36 months** for a medium-sized monolith — this is a marathon, not a sprint. The monolith continues serving features it still owns, and each extracted service delivers immediate value.

---

### Next Steps to Explore

1. **Domain-Driven Design for decomposition** — using bounded contexts to identify extraction candidates
2. **Anti-Corruption Layer patterns** — translation, facade, adapter implementations
3. **Database decomposition strategies** — shared DB → DB per service migration in detail
4. **Shadow traffic / parallel run tooling** — Diffy, Scientist, custom comparators
5. **Organizational alignment** — team topology changes needed alongside technical migration