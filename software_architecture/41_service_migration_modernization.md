

# Service Migration & Modernization in Microservices

---

## 1. Why Migrate and Modernize?

Most organizations don't start with microservices. They start with a monolith that served them well — until it didn't. Migration is the disciplined process of moving from a legacy system to a modern architecture **without stopping the business**.

```mermaid
graph LR
    subgraph "Drivers for Modernization"
        D1["Slow release cycles<br/>(quarterly deploys)"]
        D2["Scaling bottleneck<br/>(scale everything or nothing)"]
        D3["Team coupling<br/>(50 devs, 1 codebase, merge hell)"]
        D4["Technology lock-in<br/>(framework EOL, no patches)"]
        D5["Reliability fragility<br/>(one bug crashes everything)"]
        D6["Cost inefficiency<br/>(over-provisioned monolith)"]
    end

    D1 --> DECIDE{"Modernize?"}
    D2 --> DECIDE
    D3 --> DECIDE
    D4 --> DECIDE
    D5 --> DECIDE
    D6 --> DECIDE

    DECIDE -->|"Yes — but incrementally"| STRATEGY["Migration Strategy"]

    style DECIDE fill:#4ecdc4,color:#000
```

**Critical principle: Migration is not a rewrite.** Full rewrites ("Big Bang") have a ~70% failure rate. Incremental, risk-managed migration is the only approach that reliably works.

| Approach | Success Rate | Risk | Duration |
|---|---|---|---|
| **Big Bang rewrite** | Low (~30%) | Extreme — years of parallel development, feature parity gap | 2–5 years |
| **Incremental migration** | High (~75%+) | Controlled — small pieces, validated continuously | 6 months – 3 years |
| **Lift and shift** | Medium | Low migration risk, but no modernization benefit | 1–3 months |
| **Do nothing** | N/A | Growing tech debt, increasing incident rate | ∞ |

---

## 2. Migration Strategies Overview

### 2.1 The Strategy Spectrum

```mermaid
graph LR
    subgraph "The 6 Rs of Cloud Migration (adapted)"
        RETAIN["Retain<br/>(keep as-is)"]
        RETIRE["Retire<br/>(decommission)"]
        REHOST["Rehost<br/>(lift & shift)"]
        REPLATFORM["Replatform<br/>(lift, tinker & shift)"]
        REFACTOR["Refactor<br/>(re-architect to microservices)"]
        REBUILD["Rebuild<br/>(rewrite from scratch)"]
    end

    RETAIN -->|"low effort, no benefit"| REHOST
    REHOST -->|"moderate effort"| REPLATFORM
    REPLATFORM -->|"high effort, high benefit"| REFACTOR
    REFACTOR -->|"highest effort & risk"| REBUILD

    style RETAIN fill:#f0f0f0,color:#000
    style REHOST fill:#a8e6cf,color:#000
    style REPLATFORM fill:#ffe66d,color:#000
    style REFACTOR fill:#4ecdc4,color:#000
    style REBUILD fill:#ff6b6b,color:#fff
```

| Strategy | What It Means | When to Use |
|---|---|---|
| **Retain** | Leave the component in the monolith | Stable, low-change area; not worth extracting |
| **Retire** | Delete the functionality entirely | Dead code, unused features |
| **Rehost** | Move to containers/cloud, no code changes | Quick cloud migration, buy time |
| **Replatform** | Minor changes (containerize, managed DB) | Modernize infra without touching business logic |
| **Refactor / Re-architect** | Extract into microservices | Core business domains that need independent scaling/deployment |
| **Rebuild** | Rewrite from scratch | Only when legacy code is truly unmaintainable AND domain is well-understood |

### 2.2 Decision Framework: Which Strategy for Which Component?

```mermaid
graph TD
    COMP["Monolith Component"] --> Q1{"Business value<br/>& change frequency?"}
    
    Q1 -->|"Low value, rarely changes"| RETAIN2["Retain in monolith<br/>or Retire if unused"]
    Q1 -->|"High value, frequently changes"| Q2{"Code quality?"}
    
    Q2 -->|"Maintainable"| REFACTOR2["Refactor: extract as microservice<br/>(Strangler Fig)"]
    Q2 -->|"Unmaintainable, well-understood domain"| REBUILD2["Rebuild: rewrite the bounded context"]
    Q2 -->|"Unmaintainable, poorly understood"| REPLATFORM2["Replatform first<br/>(containerize, add tests,<br/>then decide)"]

    REFACTOR2 --> PRIORITY["Prioritize by:<br/>1. Business impact<br/>2. Team readiness<br/>3. Dependency complexity"]

    style REFACTOR2 fill:#4ecdc4,color:#000
    style REBUILD2 fill:#ff6b6b,color:#fff
    style RETAIN2 fill:#f0f0f0,color:#000
```

---

## 3. The Strangler Fig Pattern (Primary Migration Pattern)

### 3.1 Core Concept

Named after strangler fig trees that grow around a host tree, eventually replacing it. New microservices grow around the monolith, gradually taking over functionality until the monolith can be removed.

```mermaid
graph TB
    subgraph "Phase 1: Intercept"
        C1["Client"] --> F1["Facade / API Gateway"]
        F1 -->|"100%"| M1["Monolith<br/>(all functionality)"]
    end
```

```mermaid
graph TB
    subgraph "Phase 2: Extract First Service"
        C2["Client"] --> F2["Facade / API Gateway"]
        F2 -->|"orders/*"| SVC1["Order Service<br/>(new microservice)"]
        F2 -->|"everything else"| M2["Monolith<br/>(minus orders)"]
    end

    style SVC1 fill:#4ecdc4,color:#000
```

```mermaid
graph TB
    subgraph "Phase 3: Extract More"
        C3["Client"] --> F3["API Gateway"]
        F3 -->|"orders/*"| SVC2["Order Service"]
        F3 -->|"payments/*"| SVC3["Payment Service"]
        F3 -->|"inventory/*"| SVC4["Inventory Service"]
        F3 -->|"remaining"| M3["Monolith<br/>(shrinking)"]
    end

    style SVC2 fill:#4ecdc4,color:#000
    style SVC3 fill:#4ecdc4,color:#000
    style SVC4 fill:#4ecdc4,color:#000
    style M3 fill:#ffe66d,color:#000
```

```mermaid
graph TB
    subgraph "Phase 4: Monolith Retired"
        C4["Client"] --> F4["API Gateway"]
        F4 --> SVC5["Order Service"]
        F4 --> SVC6["Payment Service"]
        F4 --> SVC7["Inventory Service"]
        F4 --> SVC8["User Service"]
        F4 --> SVC9["Notification Service"]
    end

    style SVC5 fill:#4ecdc4,color:#000
    style SVC6 fill:#4ecdc4,color:#000
    style SVC7 fill:#4ecdc4,color:#000
    style SVC8 fill:#4ecdc4,color:#000
    style SVC9 fill:#4ecdc4,color:#000
```

### 3.2 Strangler Fig Implementation Steps

```mermaid
sequenceDiagram
    participant CLIENT as Client
    participant GW as API Gateway
    participant MONO as Monolith
    participant NEW as New Microservice
    participant DB_OLD as Monolith DB
    participant DB_NEW as Service DB

    Note over GW: Step 1: Intercept — route all traffic through gateway
    CLIENT->>GW: POST /orders
    GW->>MONO: Forward (unchanged)
    MONO->>DB_OLD: Read/Write
    MONO-->>GW: Response
    GW-->>CLIENT: Response

    Note over NEW: Step 2: Build — create new service with own DB
    Note over NEW,DB_NEW: Implement same API contract

    Note over GW: Step 3: Migrate data — sync old → new DB
    
    Note over GW: Step 4: Shadow — mirror traffic
    CLIENT->>GW: POST /orders
    GW->>MONO: Forward (primary)
    GW-->>NEW: Mirror (shadow, response discarded)
    
    Note over GW: Step 5: Canary — shift real traffic
    CLIENT->>GW: POST /orders
    GW->>NEW: 10% canary
    GW->>MONO: 90%
    
    Note over GW: Step 6: Cutover — 100% to new service
    CLIENT->>GW: POST /orders
    GW->>NEW: 100%
    NEW->>DB_NEW: Read/Write
    NEW-->>GW: Response
    GW-->>CLIENT: Response
    
    Note over MONO: Step 7: Remove old code from monolith
```

### 3.3 Routing Strategies for Strangler Fig

| Strategy | Mechanism | Best For |
|---|---|---|
| **URL-path routing** | `/api/v2/orders/*` → new service | Clean URL separation |
| **Header-based routing** | `X-Route-To: new-order-service` | A/B testing during migration |
| **Feature-flag routing** | Flag determines old vs. new path | Per-user or per-tenant rollout |
| **Content-based routing** | Route by request body field | Migrate specific entity types first |

---

## 4. Data Migration

### 4.1 The Hardest Part

Data migration is the most challenging aspect of monolith decomposition. The monolith's shared database must be split into per-service databases — while the system continues running.

```mermaid
graph TB
    subgraph "Before: Shared Database"
        MONO_APP["Monolith"]
        SHARED_DB[("Shared DB<br/>orders + payments +<br/>inventory + users<br/>(all tables, foreign keys)")]
        MONO_APP --> SHARED_DB
    end

    subgraph "After: Database per Service"
        OS["Order Service"] --> ODB[("Order DB")]
        PS["Payment Service"] --> PDB[("Payment DB")]
        IS["Inventory Service"] --> IDB[("Inventory DB")]
        US["User Service"] --> UDB[("User DB")]
    end

    style SHARED_DB fill:#ff6b6b,color:#fff
    style ODB fill:#4ecdc4,color:#000
    style PDB fill:#4ecdc4,color:#000
    style IDB fill:#4ecdc4,color:#000
    style UDB fill:#4ecdc4,color:#000
```

### 4.2 Data Migration Patterns

#### Pattern 1: Shared Database (Transitional)

```mermaid
graph TB
    subgraph "Transitional: Both Access Shared DB"
        MONO2["Monolith"] --> DB["Shared DB"]
        NEW2["New Service"] --> DB
    end

    NOTE["⚠️ Acceptable temporarily<br/>NOT a target state"]

    style NOTE fill:#ffe66d,color:#000
```

Use this as a **stepping stone** — extract the service code first, share the DB temporarily, then split the DB.

#### Pattern 2: Database View / API Wrapping

```mermaid
graph LR
    subgraph "New Service Owns Data, Monolith Reads via API"
        MONO3["Monolith"]
        NEW3["Order Service"]
        NEW_DB[("Order DB")]
    end

    MONO3 -->|"API call<br/>(replaces direct DB query)"| NEW3
    NEW3 --> NEW_DB

    style NEW3 fill:#4ecdc4,color:#000
```

#### Pattern 3: Change Data Capture (CDC)

```mermaid
graph LR
    subgraph "CDC-Based Data Sync"
        OLD_DB[("Monolith DB")]
        CDC["Debezium<br/>(CDC)"]
        KAFKA["Kafka"]
        NEW_SVC["New Service"]
        NEW_DB2[("Service DB")]
    end

    OLD_DB -->|"binlog / WAL"| CDC
    CDC -->|"change events"| KAFKA
    KAFKA -->|"consume"| NEW_SVC
    NEW_SVC -->|"write"| NEW_DB2

    style CDC fill:#ffe66d,color:#000
    style NEW_SVC fill:#4ecdc4,color:#000
```

**CDC is the safest pattern for data migration** — it keeps the new service's database in sync with the monolith in near-real-time, without modifying the monolith's code.

#### Pattern 4: Dual-Write (During Cutover)

```mermaid
sequenceDiagram
    participant GW as API Gateway
    participant NEW4 as New Service
    participant OLD_DB2 as Monolith DB
    participant NEW_DB3 as Service DB

    Note over GW,NEW_DB3: During transition: dual-write
    GW->>NEW4: POST /orders
    NEW4->>NEW_DB3: Write (primary)
    NEW4->>OLD_DB2: Write (legacy sync)
    NEW4-->>GW: 201 Created
    
    Note over NEW4: After full cutover
    GW->>NEW4: POST /orders
    NEW4->>NEW_DB3: Write (only)
    NEW4-->>GW: 201 Created
```

**Warning:** Dual-write without a transactional outbox risks inconsistency. Prefer CDC or outbox pattern.

### 4.3 Data Migration Sequence

```mermaid
graph TD
    S1["1. Create new service DB<br/>(schema designed for service's bounded context)"]
    S2["2. Set up CDC / sync pipeline<br/>(Debezium → Kafka → new DB)"]
    S3["3. Backfill historical data<br/>(snapshot + CDC catch-up)"]
    S4["4. Verify data consistency<br/>(reconciliation checks)"]
    S5["5. Switch reads to new service<br/>(monolith calls service API)"]
    S6["6. Switch writes to new service<br/>(traffic routed via gateway)"]
    S7["7. Verify: new DB is source of truth"]
    S8["8. Decommission old tables<br/>(after retention period)"]

    S1 --> S2 --> S3 --> S4 --> S5 --> S6 --> S7 --> S8

    style S4 fill:#ffe66d,color:#000
    style S8 fill:#ff6b6b,color:#fff
```

### 4.4 Breaking Foreign Keys

The monolith's shared database uses foreign keys across what will become service boundaries. These must be replaced:

| Before (Monolith) | After (Microservices) |
|---|---|
| `orders.user_id REFERENCES users.id` | Order Service stores `user_id` as opaque ID; calls User Service API when user data needed |
| `JOIN orders ON payments.order_id` | Payment Service calls Order Service API; or uses event-driven eventual consistency |
| Database enforces referential integrity | Application-level consistency via sagas / events |

---

## 5. Incremental Migration Approach

### 5.1 Domain-Driven Decomposition

```mermaid
graph TB
    subgraph "Step 1: Identify Bounded Contexts in Monolith"
        MONO4["Monolith Codebase"]
        BC1["📦 Orders"]
        BC2["📦 Payments"]
        BC3["📦 Inventory"]
        BC4["📦 Users"]
        BC5["📦 Notifications"]
        BC6["📦 Reporting"]
    end

    MONO4 --> BC1
    MONO4 --> BC2
    MONO4 --> BC3
    MONO4 --> BC4
    MONO4 --> BC5
    MONO4 --> BC6

    subgraph "Step 2: Prioritize Extraction"
        P1["🥇 Orders (high change frequency, clear boundary)"]
        P2["🥈 Payments (high business value, security isolation)"]
        P3["🥉 Notifications (simple, few dependencies)"]
        P4["Later: Users, Inventory, Reporting"]
    end

    BC1 --> P1
    BC2 --> P2
    BC5 --> P3

    style P1 fill:#4ecdc4,color:#000
    style P2 fill:#ffe66d,color:#000
    style P3 fill:#a8e6cf,color:#000
```

### 5.2 Extraction Prioritization Matrix

| Factor | Weight | Order | Payment | Notification | Inventory |
|---|---|---|---|---|---|
| **Change frequency** | 30% | High (9) | Medium (6) | Low (3) | Medium (6) |
| **Business value of independence** | 25% | High (9) | High (9) | Low (3) | Medium (6) |
| **Coupling to monolith** | 20% (inverse) | Medium (5) | Low (8) | Low (8) | High (3) |
| **Team readiness** | 15% | High (9) | Medium (6) | High (9) | Medium (6) |
| **Risk if it fails** | 10% (inverse) | Medium (5) | High (3) | Low (9) | Medium (5) |
| **Weighted Score** | | **7.5** | **6.6** | **5.1** | **5.4** |

**Extract in order of score: Orders first, then Payments, then Inventory, then Notifications.**

### 5.3 The Migration Roadmap

```mermaid
gantt
    title Monolith → Microservices Migration Roadmap
    dateFormat YYYY-MM
    axisFormat %b %Y

    section Foundation
    API Gateway / Facade             :done, 2026-01, 2026-02
    CI/CD Pipeline per service       :done, 2026-01, 2026-03
    Observability Stack (OTel+Grafana):done, 2026-02, 2026-03
    Service Catalog (Backstage)      :done, 2026-02, 2026-03

    section Wave 1
    Extract Order Service            :active, 2026-03, 2026-06
    Order DB split (CDC)             :active, 2026-04, 2026-06
    Shadow + Canary rollout          :2026-05, 2026-06

    section Wave 2
    Extract Payment Service          :2026-06, 2026-09
    Payment DB split                 :2026-07, 2026-09
    Extract Notification Service     :2026-07, 2026-08

    section Wave 3
    Extract Inventory Service        :2026-09, 2026-12
    Extract User Service             :2026-10, 2027-01
    
    section Cleanup
    Decommission monolith            :2027-01, 2027-03
    Remove legacy DB tables          :2027-02, 2027-03
```

---

## 6. Anti-Corruption Layer (ACL)

### 6.1 Concept

When the new microservice must communicate with the legacy monolith, an **Anti-Corruption Layer** translates between the legacy model and the new service's domain model — preventing legacy concepts from leaking into clean new code.

```mermaid
graph LR
    subgraph "New Service (Clean Domain)"
        OS2["Order Service"]
        ACL["Anti-Corruption Layer<br/>(translator)"]
    end

    subgraph "Legacy"
        MONO5["Monolith API"]
        LDB[("Legacy DB<br/>denormalized, messy")]
    end

    OS2 --> ACL
    ACL -->|"translate legacy → clean model"| MONO5
    ACL -->|"translate legacy → clean model"| LDB

    style ACL fill:#ffe66d,color:#000
    style OS2 fill:#4ecdc4,color:#000
    style MONO5 fill:#ff6b6b,color:#fff
```

### 6.2 ACL Implementation

```python
# Anti-Corruption Layer: translate legacy payment response to clean model

# Legacy monolith returns:
# {"PAY_ID": "00123", "PAY_STAT": "OK", "AMT_CENTS": 4999, 
#  "CUST_REF": "C-789", "TS": "20260418102345"}

class PaymentACL:
    """Translates legacy payment system responses to clean domain model."""

    def to_payment_result(self, legacy_response: dict) -> PaymentResult:
        return PaymentResult(
            payment_id=f"pay_{legacy_response['PAY_ID'].lstrip('0')}",
            status=self._map_status(legacy_response["PAY_STAT"]),
            amount=Decimal(legacy_response["AMT_CENTS"]) / 100,
            customer_id=legacy_response["CUST_REF"],
            timestamp=datetime.strptime(
                legacy_response["TS"], "%Y%m%d%H%M%S"
            ),
        )

    def _map_status(self, legacy_status: str) -> PaymentStatus:
        mapping = {
            "OK": PaymentStatus.CHARGED,
            "FAIL": PaymentStatus.FAILED,
            "PEND": PaymentStatus.PENDING,
            "REF": PaymentStatus.REFUNDED,
        }
        return mapping.get(legacy_status, PaymentStatus.UNKNOWN)
```

**The ACL is temporary** — it exists during migration and is removed when the legacy system is decommissioned.

---

## 7. Feature Parity & Verification

### 7.1 Ensuring the New Service Matches the Old

```mermaid
graph TD
    subgraph "Verification Strategies"
        SHADOW["Shadow Testing<br/>(mirror traffic, compare responses)"]
        CONTRACT["Contract Tests<br/>(same API contract)"]
        RECON["Data Reconciliation<br/>(compare old DB vs new DB)"]
        SYNTH["Synthetic Tests<br/>(scripted user journeys)"]
        CANARY2["Canary with Metrics<br/>(compare error rate, latency)"]
    end

    SHADOW --> CONFIDENCE["Confidence to cut over"]
    CONTRACT --> CONFIDENCE
    RECON --> CONFIDENCE
    SYNTH --> CONFIDENCE
    CANARY2 --> CONFIDENCE

    style CONFIDENCE fill:#4ecdc4,color:#000
```

### 7.2 Shadow Testing (Diff Testing)

```mermaid
sequenceDiagram
    participant CLIENT2 as Client
    participant GW2 as API Gateway
    participant MONO6 as Monolith (primary)
    participant NEW5 as New Service (shadow)
    participant DIFF as Diff Comparator

    CLIENT2->>GW2: GET /orders/123
    GW2->>MONO6: Forward (primary)
    GW2->>NEW5: Mirror (shadow)
    MONO6-->>GW2: Response A
    NEW5-->>DIFF: Response B
    MONO6-->>DIFF: Response A

    DIFF->>DIFF: Compare A vs B
    alt Responses match
        DIFF->>DIFF: ✅ Log: match
    else Responses differ
        DIFF->>DIFF: ⚠️ Log: diff details<br/>{field: "total", old: 49.99, new: 50.00}
    end

    GW2-->>CLIENT2: Response A (always from monolith)
```

**Shadow testing runs for weeks before cutover** — building confidence that the new service produces identical results. Only the monolith's response is returned to the client.

### 7.3 Data Reconciliation

```mermaid
graph LR
    subgraph "Periodic Reconciliation Job"
        OLD_DB3[("Monolith DB")]
        NEW_DB4[("Service DB")]
        RECON2["Reconciliation<br/>Worker"]
        REPORT["Discrepancy Report"]
    end

    OLD_DB3 --> RECON2
    NEW_DB4 --> RECON2
    RECON2 --> REPORT

    REPORT -->|"0 discrepancies"| READY["✅ Ready for cutover"]
    REPORT -->|"discrepancies found"| FIX["🔧 Investigate + fix sync"]

    style READY fill:#4ecdc4,color:#000
    style FIX fill:#ff6b6b,color:#fff
```

---

## 8. Organizational Migration

### 8.1 Conway's Law in Migration

> "Organizations design systems that mirror their communication structures."

Migrating architecture **without** migrating the team structure fails. If a monolith team still owns all microservices, you get a "distributed monolith."

```mermaid
graph TB
    subgraph "Before: Feature Teams on Monolith"
        FT1["Team 1: Feature X"]
        FT2["Team 2: Feature Y"]
        FT3["Team 3: Feature Z"]
        MONO7["Monolith"]
        
        FT1 --> MONO7
        FT2 --> MONO7
        FT3 --> MONO7
    end

    subgraph "After: Stream-Aligned Teams on Services"
        ST1["Team Orders<br/>(owns Order Service)"]
        ST2["Team Payments<br/>(owns Payment Service)"]
        ST3["Team Catalog<br/>(owns Inventory + Search)"]
        PT["Platform Team<br/>(owns infra, CI/CD, mesh)"]
    end

    ST1 -->|owns| SVC_O["Order Service"]
    ST2 -->|owns| SVC_P["Payment Service"]
    ST3 -->|owns| SVC_I["Inventory Service"]
    PT -->|enables| ST1
    PT -->|enables| ST2
    PT -->|enables| ST3

    style ST1 fill:#4ecdc4,color:#000
    style ST2 fill:#4ecdc4,color:#000
    style ST3 fill:#4ecdc4,color:#000
    style PT fill:#ffe66d,color:#000
```

### 8.2 Team Topology During Migration

| Phase | Team Structure | Focus |
|---|---|---|
| **Phase 0** | Existing monolith teams | Business as usual |
| **Phase 1** | Small "pioneer" team (2–3 people) | Extract first service, build platform foundations |
| **Phase 2** | Pioneer team becomes platform team; first stream-aligned team formed | First service in production, patterns established |
| **Phase 3** | More stream-aligned teams spun up | Parallel extraction of multiple bounded contexts |
| **Phase 4** | Full stream-aligned + platform + enabling teams | Monolith decommissioned |

### 8.3 Skill Migration

| Skill Gap | Training / Hiring |
|---|---|
| Containerization (Docker, K8s) | Hands-on workshops + platform team support |
| Distributed systems thinking | Book clubs (Designing Data-Intensive Applications), chaos engineering exercises |
| Observability (OTel, Grafana) | Guided setup of first service's dashboards |
| CI/CD automation | Platform team provides golden-path templates |
| Domain-Driven Design | DDD workshops to identify bounded contexts |

---

## 9. Modernization Patterns Beyond Strangler Fig

### 9.1 Branch by Abstraction

When you can't route at the network level (internal monolith logic), use code-level abstraction:

```mermaid
graph TD
    subgraph "Step 1: Identify Component to Replace"
        CLIENT3["Calling Code"]
        OLD_IMPL["Old Implementation<br/>(tightly coupled)"]
        CLIENT3 --> OLD_IMPL
    end

    subgraph "Step 2: Introduce Abstraction"
        CLIENT4["Calling Code"]
        IFACE["Interface / Abstraction"]
        OLD_IMPL2["Old Implementation"]
        CLIENT4 --> IFACE
        IFACE --> OLD_IMPL2
    end

    subgraph "Step 3: Add New Implementation"
        CLIENT5["Calling Code"]
        IFACE2["Interface"]
        OLD_IMPL3["Old Implementation"]
        NEW_IMPL["New Implementation<br/>(calls microservice)"]
        FLAG["Feature Flag"]
        CLIENT5 --> IFACE2
        IFACE2 --> FLAG
        FLAG -->|off| OLD_IMPL3
        FLAG -->|on| NEW_IMPL
    end

    subgraph "Step 4: Remove Old"
        CLIENT6["Calling Code"]
        IFACE3["Interface"]
        NEW_IMPL2["New Implementation<br/>(only path)"]
        CLIENT6 --> IFACE3 --> NEW_IMPL2
    end

    style NEW_IMPL fill:#4ecdc4,color:#000
    style NEW_IMPL2 fill:#4ecdc4,color:#000
    style FLAG fill:#ffe66d,color:#000
```

### 9.2 Parallel Run

Run both old and new implementations simultaneously, compare results, only return the old implementation's result to the user:

```python
# Parallel run pattern (using GitHub Scientist-style library)
from scientist import Experiment

def get_order_total(order_id):
    experiment = Experiment("order-total-migration")
    
    # Control: monolith calculation (always returned)
    experiment.use(lambda: monolith_order_service.get_total(order_id))
    
    # Candidate: new microservice (compared, never returned)
    experiment.try(lambda: new_order_service.get_total(order_id))
    
    # Compare results
    experiment.compare(lambda control, candidate: 
        abs(control - candidate) < 0.01)  # tolerate rounding
    
    # Run: returns control result, logs mismatches
    return experiment.run()
```

### 9.3 Event Interception

Intercept events/messages from the monolith to build new read models or trigger new service logic:

```mermaid
graph LR
    MONO8["Monolith"] -->|"publishes events<br/>(add CDC if needed)"| KAFKA2["Kafka"]
    KAFKA2 --> NEW6["New Search Service<br/>(builds own index)"]
    KAFKA2 --> NEW7["New Analytics Service<br/>(builds own projections)"]

    MONO8 -->|"still handles writes"| DB5[("Monolith DB")]

    style NEW6 fill:#4ecdc4,color:#000
    style NEW7 fill:#4ecdc4,color:#000
```

### 9.4 UI Composition (Micro-Frontends)

```mermaid
graph TB
    subgraph "Monolith UI Migration"
        SHELL["App Shell / Container"]
        LEGACY_UI["Legacy Monolith UI<br/>(iframe or bundled)"]
        NEW_UI1["Order Widget<br/>(React micro-frontend)"]
        NEW_UI2["Payment Widget<br/>(React micro-frontend)"]
    end

    SHELL --> LEGACY_UI
    SHELL --> NEW_UI1
    SHELL --> NEW_UI2

    NEW_UI1 -->|API| SVC_O2["Order Service"]
    NEW_UI2 -->|API| SVC_P2["Payment Service"]
    LEGACY_UI -->|API| MONO9["Monolith"]

    style NEW_UI1 fill:#4ecdc4,color:#000
    style NEW_UI2 fill:#4ecdc4,color:#000
    style LEGACY_UI fill:#ff6b6b,color:#fff
```

---

## 10. Migration Risks & Mitigation

### 10.1 Risk Registry

| Risk | Impact | Likelihood | Mitigation |
|---|---|---|---|
| **Data loss during migration** | Critical | Medium | CDC with reconciliation; never delete source data until verified |
| **Feature parity gap** | High | High | Shadow testing + comprehensive contract tests |
| **Performance regression** | High | Medium | Load test new service before cutover; canary with latency analysis |
| **Team burnout** | High | Medium | Time-box migration waves; celebrate milestones |
| **Distributed monolith** | High | High | Enforce bounded contexts; no shared databases in target state |
| **Migration stalls** | Medium | High | Deliver business value each wave; avoid "infrastructure only" phases |
| **Dual maintenance** | Medium | High | Minimize overlap period; feature-freeze monolith areas being extracted |

### 10.2 Migration Rollback Strategy

```mermaid
graph TD
    CUTOVER["Service cutover:<br/>100% traffic to new service"] --> MONITOR2["Monitor for 1–2 weeks"]
    
    MONITOR2 --> OK{"Issues?"}
    OK -->|"None"| DECOM["Decommission old code path"]
    OK -->|"Minor"| FIX2["Fix forward in new service"]
    OK -->|"Major"| REVERT["Revert: route traffic<br/>back to monolith"]
    
    REVERT --> INVESTIGATE["Investigate + fix"]
    INVESTIGATE --> CUTOVER

    NOTE2["⚠️ Keep monolith code path<br/>functional for 2–4 weeks<br/>after cutover"]

    style REVERT fill:#ff6b6b,color:#fff
    style DECOM fill:#4ecdc4,color:#000
    style NOTE2 fill:#ffe66d,color:#000
```

**Never decommission the old path immediately after cutover.** Keep the monolith's code runnable (even if no traffic flows to it) as a safety net for 2–4 weeks.

---

## 11. Measuring Migration Progress

### 11.1 Migration Metrics

```mermaid
graph TB
    subgraph "Migration Health Dashboard"
        M1["% of traffic served by<br/>microservices vs monolith"]
        M2["# of bounded contexts<br/>extracted / remaining"]
        M3["Monolith deploy frequency<br/>(should decrease)"]
        M4["New services deploy frequency<br/>(should increase)"]
        M5["Monolith LOC<br/>(should shrink)"]
        M6["Incident rate: monolith<br/>vs microservices"]
    end

    style M1 fill:#4ecdc4,color:#000
    style M2 fill:#ffe66d,color:#000
```

### 11.2 Progress Visualization

```
Migration Progress (April 2026)

Bounded Contexts:
  ████████████░░░░░░░░  60% extracted (6/10)
  
Traffic Distribution:
  Monolith:       ██████░░░░░░░░░░░░░░  30%
  Microservices:  ██████████████░░░░░░  70%

Monolith Size:
  Jan 2026: ████████████████████  250K LOC
  Apr 2026: ████████████░░░░░░░░  145K LOC
  Target:   ░░░░░░░░░░░░░░░░░░░░  0 LOC (Q1 2027)

Deploy Frequency:
  Monolith:       2x / month
  Microservices:  8x / day (average across all services)
```

---

## 12. Anti-Patterns

| Anti-Pattern | Problem | Fix |
|---|---|---|
| **Big Bang rewrite** | 2-year rewrite, feature parity never reached, project canceled | Strangler Fig: incremental extraction |
| **Distributed monolith** | Extracted services but they share a database and deploy together | True bounded contexts; database-per-service; independent pipelines |
| **Migration without business value** | "We're modernizing infrastructure" — 6 months, no user-visible improvement | Each wave must deliver measurable business value |
| **Premature optimization** | Extracting services that rarely change | Prioritize by change frequency × business value |
| **No data migration plan** | "We'll figure out the database later" | Data migration is the hardest part — plan it first |
| **Ignoring the ACL** | Legacy data model leaks into new service | Anti-Corruption Layer translates at the boundary |
| **Feature freeze during migration** | "Stop all features until migration is done" | Unacceptable to business; migrate incrementally alongside feature work |
| **Ignoring Conway's Law** | New architecture, same team structure | Align teams to bounded contexts; form stream-aligned teams |
| **No rollback plan** | Cut over to new service, decommission monolith immediately | Keep monolith runnable for 2–4 weeks post-cutover |
| **Migrating everything** | Every module must become a microservice | Some things should stay in the monolith (low change, low value) |
| **No verification** | "It works in staging, ship it" | Shadow testing + data reconciliation + canary analysis |

---

## 13. Decision Framework

```mermaid
graph TD
    START{"What are you<br/>modernizing?"} -->|"Monolith → Microservices"| Q1{"How big is the monolith?"}

    Q1 -->|"< 50K LOC, < 5 devs"| MODULAR["Consider modular monolith first<br/>(cheaper, less risk)"]
    Q1 -->|"50K–500K LOC"| STRANGLER["Strangler Fig<br/>+ CDC for data"]
    Q1 -->|"> 500K LOC"| STRANGLER_PLUS["Strangler Fig<br/>+ dedicated migration team<br/>+ platform team"]

    STRANGLER --> Q2{"Where to start?"}
    STRANGLER_PLUS --> Q2

    Q2 --> ASSESS2["Score bounded contexts:<br/>change frequency × business value<br/>÷ coupling complexity"]
    ASSESS2 --> FIRST["Extract highest-scoring<br/>bounded context first"]

    FIRST --> Q3{"Data migration approach?"}
    Q3 -->|"Tables are isolated"| SIMPLE_SPLIT["Direct split +<br/>one-time migration"]
    Q3 -->|"Heavy cross-table joins"| CDC_APPROACH["CDC (Debezium) +<br/>reconciliation"]
    Q3 -->|"Complex shared state"| SHARED_TEMP["Shared DB temporarily →<br/>split after code extraction"]

    MODULAR --> Q4{"Scaling/deploy independence<br/>truly needed later?"}
    Q4 -->|"Yes"| STRANGLER
    Q4 -->|"No"| STAY_MODULAR["Stay modular monolith<br/>✅"]

    style STRANGLER fill:#4ecdc4,color:#000
    style MODULAR fill:#ffe66d,color:#000
    style CDC_APPROACH fill:#4ecdc4,color:#000
```

---

## 14. Checklist

### Assessment
- [ ] Business case for migration documented (not just "microservices are modern")
- [ ] Bounded contexts identified via Domain-Driven Design workshops
- [ ] Each bounded context scored: change frequency, business value, coupling
- [ ] Extraction order prioritized and time-boxed into waves
- [ ] "Do nothing" / "modular monolith" alternatives considered and dismissed with rationale

### Foundation
- [ ] API Gateway / Facade deployed in front of monolith (intercept point)
- [ ] CI/CD pipeline templates ready for new services
- [ ] Observability stack deployed (metrics, logs, traces — for both monolith and new services)
- [ ] Service catalog (Backstage) set up with both monolith and new services registered
- [ ] Platform team formed (or platform responsibilities assigned)

### Extraction
- [ ] Anti-Corruption Layer built between new service and legacy
- [ ] New service has its own database (or shared DB with documented transition plan)
- [ ] Data migration via CDC (Debezium) with reconciliation checks
- [ ] Shadow testing validates feature parity (diff old vs. new responses)
- [ ] Contract tests ensure API compatibility
- [ ] Canary deployment with automated analysis for cutover
- [ ] Monolith code path kept runnable for 2–4 weeks post-cutover

### Data
- [ ] Foreign keys across bounded contexts replaced with API calls or events
- [ ] Data reconciliation job runs continuously during migration
- [ ] Historical data backfilled and verified
- [ ] Rollback plan: can route traffic back to monolith if needed

### Organization
- [ ] Teams aligned to bounded contexts (stream-aligned teams)
- [ ] Platform team supports service infrastructure
- [ ] Training provided: containers, DDD, observability, distributed systems
- [ ] Each migration wave delivers measurable business value

### Verification
- [ ] Shadow testing covers > 95% of API endpoints
- [ ] Data reconciliation shows zero discrepancies for 1+ week
- [ ] Performance load-tested: new service meets or exceeds monolith latency
- [ ] Rollback tested: traffic can be switched back to monolith within seconds

### Cleanup
- [ ] Old code removed from monolith after successful cutover + bake period
- [ ] Old database tables archived then dropped
- [ ] Legacy CI/CD pipelines decommissioned
- [ ] Service catalog updated: old component → Retired

---

## 15. Recommendation

**Migrate in waves, not all at once:**

| Phase | Focus | Key Outcome |
|---|---|---|
| **Phase 0** | Foundation: gateway, CI/CD, observability, catalog | Infrastructure ready for extraction |
| **Phase 1** | Extract 1 bounded context (highest-value, lowest-coupling) | Prove the pattern works; build team confidence |
| **Phase 2** | Extract 2–3 more bounded contexts in parallel | Accelerate with proven patterns and platform support |
| **Phase 3** | Remaining high-value extractions | Most traffic served by microservices |
| **Phase 4** | Decommission monolith; clean up residual | Migration complete |

The golden rule: **every migration wave must deliver business value, not just architectural purity**. If wave 1 extracts the Order Service, the business should see faster order-feature delivery, better order-page performance, or higher order-system reliability — something tangible. Architecture migration without visible business improvement loses executive support and dies on the vine.

---

**Next steps to explore:** Domain-Driven Design for Bounded Context Discovery, Modular Monolith as Migration Stepping Stone, Platform Engineering for Migration Support.