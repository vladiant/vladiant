# How to handle deployments and rollbacks in a Microservices architecture?

In a monolith, deployment is one artifact, one process, one rollback decision. In microservices, you deploy **N services independently, each on its own cadence**, potentially dozens of times per day. The architectural challenge:

> **Every deployment is a production change.** The difference between "10 deploys/day with confidence" and "1 deploy/month with terror" is your deployment strategy and rollback capability.

The goal is **zero-downtime deployments with instant rollback** — where a bad release is detected automatically and reverted before users notice.

---

## 1. The Deployment Problem Space

```mermaid
graph TB
    subgraph "Monolith Deployment"
        M1[Build v1.0] --> M2[Test] --> M3[Deploy ALL<br/>or NOTHING] --> M4[Rollback ALL<br/>or NOTHING]
    end
```

```mermaid
graph TB
    subgraph "Microservices Deployment"
        A[Service A v2.1] --> DA[Deploy independently]
        B[Service B v1.8] --> DB[Deploy independently]
        C[Service C v3.0] --> DC[Deploy independently]
        
        DA --> COMPAT{Compatible with<br/>B v1.8 and C v3.0?}
        COMPAT -- "Yes" --> OK[✓ Success]
        COMPAT -- "No" --> BREAK[✗ Integration failure<br/>in production]
    end
```

| Challenge | Why It's Hard |
|-----------|--------------|
| **Independent deployment cadence** | Service A deploys 5x/day, Service B deploys weekly — versions must be compatible |
| **N-way compatibility** | A new version of Service A must work with current *and* previous versions of its dependencies |
| **Stateful rollbacks** | Rolling back code is easy; rolling back database migrations is not |
| **Partial rollouts** | You want 5% of traffic on the new version before committing 100% |
| **Blast radius** | A bad deploy of one service shouldn't take down the entire system |

---

## 2. Deployment Strategies

### Strategy A: Rolling Update

```mermaid
graph LR
    subgraph "Time 0: All v1"
        A1[v1] 
        A2[v1]
        A3[v1]
        A4[v1]
    end
```
```mermaid
graph LR
    subgraph "Time 1: Rolling"
        B1[v2 ✓]
        B2[v1]
        B3[v1]
        B4[v1]
    end
```
```mermaid
graph LR
    subgraph "Time 2: Rolling"
        C1[v2 ✓]
        C2[v2 ✓]
        C3[v1]
        C4[v1]
    end
```
```mermaid
graph LR
    subgraph "Time 3: Complete"
        D1[v2 ✓]
        D2[v2 ✓]
        D3[v2 ✓]
        D4[v2 ✓]
    end
```

| Criterion | Assessment |
|-----------|-----------|
| **Zero downtime** | Yes — always some instances available |
| **Rollback speed** | Slow — must roll forward or reverse the rolling update |
| **Compatibility** | v1 and v2 run simultaneously during rollout — must be backward compatible |
| **Resource cost** | Low — replaces instances in-place (respects `maxSurge` / `maxUnavailable`) |
| **Complexity** | Low — Kubernetes default strategy |
| **Best for** | Standard deployments with backward-compatible changes |

**Kubernetes config:**
```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 25%
    maxUnavailable: 0     # Never reduce capacity during rollout
```

### Strategy B: Blue-Green Deployment

```mermaid
graph TB
    LB[Load Balancer] --> BLUE

    subgraph "Blue (Current — v1)"
        BLUE[v1 instances<br/>Serving 100% traffic]
    end

    subgraph "Green (New — v2)"
        GREEN[v2 instances<br/>Idle, fully tested]
    end

    LB -. "Switch in one step" .-> GREEN
```

**After switch:**

```mermaid
graph TB
    LB[Load Balancer] --> GREEN

    subgraph "Blue (Previous — v1)"
        BLUE[v1 instances<br/>Idle, ready for rollback]
    end

    subgraph "Green (Current — v2)"
        GREEN[v2 instances<br/>Serving 100% traffic]
    end
```

| Criterion | Assessment |
|-----------|-----------|
| **Zero downtime** | Yes — instant traffic switch |
| **Rollback speed** | Instant — switch LB back to Blue |
| **Compatibility** | v1 and v2 never serve simultaneously — no mixed-version window |
| **Resource cost** | High — double the infrastructure during deployment |
| **Complexity** | Medium — need to manage two identical environments |
| **Best for** | Critical services where instant rollback is essential; database-incompatible changes |

### Strategy C: Canary Deployment

```mermaid
graph TB
    LB[Load Balancer<br/>Traffic Split]

    subgraph "Stable (v1)"
        S1[v1]
        S2[v1]
        S3[v1]
        S4[v1]
    end

    subgraph "Canary (v2)"
        C1[v2]
    end

    LB -- "95%" --> S1
    LB -- "5%" --> C1

    MONITOR[Monitor canary:<br/>Error rate? Latency?<br/>Business metrics?]
    C1 --> MONITOR

    MONITOR -- "Healthy" --> PROMOTE[Gradually increase<br/>5% → 25% → 50% → 100%]
    MONITOR -- "Unhealthy" --> ROLLBACK[Kill canary<br/>100% back to v1]
```

| Criterion | Assessment |
|-----------|-----------|
| **Zero downtime** | Yes |
| **Rollback speed** | Fast — kill canary instances, 100% back to stable |
| **Blast radius** | Minimal — only 5% of traffic affected by a bad release |
| **Compatibility** | v1 and v2 serve simultaneously — must be backward compatible |
| **Complexity** | Medium-High — needs traffic splitting, metrics comparison, promotion logic |
| **Best for** | High-traffic services; validating changes with real traffic before full rollout |

### Strategy D: Feature Flags (Dark Launch + Progressive Delivery)

```mermaid
graph TB
    subgraph "Single Deployed Version (v2)"
        FF{Feature Flag:<br/>new-checkout-flow}
        FF -- "ON (10% of users)" --> NEW[New Checkout Code Path]
        FF -- "OFF (90%)" --> OLD[Old Checkout Code Path]
    end

    MGMT[Feature Flag Service<br/>LaunchDarkly / Flagsmith / Unleash] --> FF
```

| Criterion | Assessment |
|-----------|-----------|
| **Decouples deploy from release** | Deploy code anytime; enable feature independently |
| **Rollback speed** | Instant — toggle flag off; no redeployment needed |
| **Targeting** | Granular — by user, region, percentage, account type |
| **Complexity** | Medium — flag management, cleanup of old flags, testing permutations |
| **Risk** | Flag sprawl; dead code if flags aren't cleaned up; testing complexity (2^N states) |
| **Best for** | Major feature launches; A/B testing; gradual rollout to specific cohorts |

---

## 3. Comparison

| Criterion | Rolling Update | Blue-Green | Canary | Feature Flags |
|-----------|---------------|------------|--------|--------------|
| **Rollback speed** | Minutes | Seconds | Seconds | Instant |
| **Blast radius** | Grows during rollout | All-or-nothing | Controlled (5-10%) | Controlled (per-user) |
| **Resource cost** | Low | 2× during deploy | Low (+1 canary) | None (same deploy) |
| **Mixed versions** | Yes (during rollout) | No | Yes | No (same binary) |
| **Complexity** | Low | Medium | Medium-High | Medium |
| **DB migration friendly** | Needs backward compat | Yes (separate envs) | Needs backward compat | Yes (code-level branching) |
| **Observability needed** | Basic health checks | Health check + smoke test | Advanced metrics comparison | Feature-level metrics |

---

## 4. Rollback Strategies

### The Rollback Decision Tree

```mermaid
graph TD
    DETECT[Problem Detected<br/>Automated alert or manual] --> SEVERITY

    SEVERITY{Severity?}
    SEVERITY -- "P1: Revenue/data impact" --> IMMEDIATE[Immediate Rollback<br/>Don't debug in production]
    SEVERITY -- "P2-P3: Degraded but functional" --> ASSESS

    ASSESS{Can you fix forward?}
    ASSESS -- "Yes, < 30 min" --> HOTFIX[Fix Forward<br/>Deploy hotfix]
    ASSESS -- "No, complex" --> ROLLBACK[Rollback to last known good]

    IMMEDIATE --> HOW{Rollback method?}
    ROLLBACK --> HOW

    HOW -- "Blue-Green" --> SWITCH[Switch LB to Blue]
    HOW -- "Canary" --> KILL[Kill canary, 100% stable]
    HOW -- "Feature Flag" --> TOGGLE[Toggle flag OFF]
    HOW -- "Rolling" --> REDEPLOY[Redeploy previous image tag]
    HOW -- "GitOps" --> REVERT[Revert Git commit → auto-deploy]
```

### Rollback: Code vs Data

```mermaid
graph TB
    subgraph "Easy: Code Rollback"
        CR[Redeploy previous<br/>container image]
    end

    subgraph "Hard: Data Rollback"
        DR[Database schema changed?<br/>Events published?<br/>External state modified?]
        DR --> Q1{Schema backward<br/>compatible?}
        Q1 -- "Yes" --> SAFE[Safe to roll back code<br/>Old code works with new schema]
        Q1 -- "No" --> DANGER[Cannot simply roll back<br/>Need data migration plan]
    end
```

| Rollback Target | Difficulty | Strategy |
|-----------------|-----------|----------|
| **Stateless code** | Easy | Redeploy previous image |
| **Config change** | Easy | Revert config in config server / env vars |
| **Feature flag** | Trivial | Toggle off |
| **Additive DB migration** (new column) | Easy | Old code ignores new column |
| **Destructive DB migration** (dropped column) | Hard | Must restore data; use expand-contract |
| **Published events** | Very Hard | Consumers may have already processed them; need compensating events |
| **Third-party state** (payments, emails) | Cannot rollback | Must compensate (refund, retract notification) |

---

## 5. Database Migration Strategy

The #1 cause of "can't rollback" is **database migrations that break backward compatibility**.

### The Expand-Contract Pattern for Zero-Downtime Migrations

```mermaid
sequenceDiagram
    participant V1 as Code v1
    participant DB as Database
    participant V2 as Code v2
    participant V3 as Code v3

    Note over V1,DB: Phase 1: Expand — Add new column
    V1->>DB: ALTER TABLE ADD COLUMN new_name VARCHAR
    Note over DB: Both old_name and new_name exist
    V1->>DB: Write to old_name (still works)
    V1->>DB: Backfill: UPDATE SET new_name = old_name

    Note over V2,DB: Phase 2: Migrate — Code reads/writes both
    V2->>DB: Write to BOTH old_name and new_name
    V2->>DB: Read from new_name (fallback to old_name)

    Note over V3,DB: Phase 3: Contract — Drop old column
    V3->>DB: Read/write new_name only
    V3->>DB: ALTER TABLE DROP COLUMN old_name
```

| Phase | Code Version | DB State | Rollback Safe? |
|-------|-------------|----------|---------------|
| **Expand** | v1 (unchanged) | Both columns exist | Yes — v1 ignores new column |
| **Migrate** | v2 (dual-write) | Both columns populated | Yes — roll back to v1, old column still correct |
| **Contract** | v3 (new only) | Old column dropped | **No** — cannot roll back to v1 after this |

**Rule:** Never combine code deployment and destructive migration in the same release.

---

## 6. CI/CD Pipeline Architecture

```mermaid
graph TB
    subgraph "Source"
        GIT[Git Push / PR Merge]
    end

    subgraph "CI Pipeline"
        BUILD[Build + Unit Tests]
        SAST[SAST / Security Scan]
        INTEGRATION["Integration Tests<br/>Contract Tests (Pact)"]
        IMAGE[Build Container Image<br/>Tag: git-sha]
        REGISTRY[Push to Container Registry]
    end

    subgraph "CD Pipeline"
        DEPLOY_DEV[Deploy to Dev<br/>Auto]
        TEST_DEV[Smoke Tests in Dev]
        DEPLOY_STAGING[Deploy to Staging<br/>Auto]
        TEST_STAGING[Integration + Performance Tests]
        CANARY_PROD[Canary Deploy to Prod<br/>5% traffic]
        MONITOR[Monitor Canary<br/>5 min — error rate, latency]
        PROMOTE{Healthy?}
        PROMOTE -- "Yes" --> FULL[Promote to 100%]
        PROMOTE -- "No" --> ABORT[Auto-Rollback Canary]
    end

    GIT --> BUILD --> SAST --> INTEGRATION --> IMAGE --> REGISTRY
    REGISTRY --> DEPLOY_DEV --> TEST_DEV --> DEPLOY_STAGING --> TEST_STAGING
    TEST_STAGING --> CANARY_PROD --> MONITOR --> PROMOTE
```

### GitOps Model (Declarative + Auditable)

```mermaid
graph LR
    subgraph "Developer"
        DEV[Developer pushes<br/>code change]
    end

    subgraph "CI"
        CI_BUILD[Build image<br/>Tag: sha-abc123]
        CI_PUSH[Push to registry]
    end

    subgraph "GitOps Repo"
        GITOPS[Update image tag<br/>in deployment manifest<br/>via PR]
    end

    subgraph "GitOps Controller"
        ARGO[ArgoCD / Flux<br/>Watches git repo<br/>Applies to cluster]
    end

    subgraph "Kubernetes"
        K8S[Cluster reconciles<br/>to desired state]
    end

    DEV --> CI_BUILD --> CI_PUSH --> GITOPS --> ARGO --> K8S

    ROLLBACK[Rollback = git revert<br/>on GitOps repo] --> ARGO
```

| Criterion | Imperative CD (Jenkins/GitHub Actions direct deploy) | GitOps (ArgoCD/Flux) |
|-----------|-----------------------------------------------------|---------------------|
| **Audit trail** | Build logs | Git history — every state change is a commit |
| **Rollback** | Retrigger pipeline with old tag | `git revert` — automatic reconciliation |
| **Drift detection** | Manual | Built-in — controller reconciles to git state |
| **Access control** | CD tool credentials | Git PR reviews — no direct cluster access needed |
| **Complexity** | Lower | Medium — need GitOps repo + controller |

---

## 7. Automated Rollback with Progressive Delivery

```mermaid
sequenceDiagram
    participant CD as CD Pipeline
    participant ARGO as Argo Rollouts / Flagger
    participant K8S as Kubernetes
    participant PROM as Prometheus
    
    CD->>ARGO: Deploy new ReplicaSet (v2)
    ARGO->>K8S: Create canary pods (5% traffic)
    
    loop Every 60 seconds for 5 minutes
        ARGO->>PROM: Query: error_rate{version="v2"} < 1%?
        ARGO->>PROM: Query: p99_latency{version="v2"} < 500ms?
        alt Metrics healthy
            ARGO->>K8S: Increase canary traffic (5% → 25% → 50% → 100%)
        else Metrics unhealthy
            ARGO->>K8S: Rollback — delete v2 pods, 100% to v1
            ARGO->>CD: Notify: rollback triggered
        end
    end
```

**Tools for progressive delivery:**

| Tool | Platform | Traffic Splitting | Auto-Rollback |
|------|----------|-------------------|---------------|
| **Argo Rollouts** | Kubernetes | Canary, Blue-Green, Traffic mirroring | Yes (Prometheus/Datadog analysis) |
| **Flagger** | Kubernetes + Istio/Linkerd | Canary, A/B, Blue-Green | Yes (built-in analysis) |
| **AWS CodeDeploy** | ECS, Lambda, EC2 | Canary, Linear, All-at-once | Yes (CloudWatch alarms) |
| **Istio** | Kubernetes | VirtualService weight-based routing | Manual (combine with Flagger for auto) |
| **LaunchDarkly / Flagsmith** | Any | Feature-flag based | Yes (flag kill switch) |

---

## 8. Multi-Service Deployment Coordination

When services have **cross-service dependencies**, you need deployment ordering.

### Strategy: Deploy in Dependency Order with Backward Compatibility

```mermaid
graph TB
    subgraph "Deploy Wave 1 (Providers first)"
        DB_MIG[Database Migration<br/>Additive only] --> PROVIDER[Deploy Provider Service<br/>Serves both v1 and v2 contracts]
    end

    subgraph "Deploy Wave 2 (Consumers)"
        CONSUMER[Deploy Consumer Service<br/>Uses v2 contract]
    end

    subgraph "Deploy Wave 3 (Cleanup)"
        CLEANUP[Remove v1 contract support<br/>from Provider]
    end

    PROVIDER --> CONSUMER --> CLEANUP
```

| Rule | Why |
|------|-----|
| **Providers deploy before consumers** | Consumer can't call v2 API if provider hasn't deployed it yet |
| **Provider supports N and N-1 contracts** | In case consumer rollback is needed while provider stays on v2 |
| **Never deploy consumer and provider simultaneously** | If both fail, you can't tell which caused the problem |
| **Database migrations deploy first** | Schema must be ready before code that depends on it |

---

## 9. Deployment Observability

| What to Track | Why | How |
|---------------|-----|-----|
| **Deploy frequency** | DORA metric — measure delivery velocity | CI/CD pipeline metrics |
| **Lead time for changes** | DORA metric — commit to production | Git timestamp → deploy timestamp |
| **Change failure rate** | DORA metric — % of deploys causing incidents | Incident count / deploy count |
| **Mean time to recovery** | DORA metric — how fast you fix failures | Incident timeline tracking |
| **Deploy annotations on dashboards** | Correlate performance changes with deploys | Grafana annotations from CD webhook |
| **Canary vs stable metrics comparison** | Real-time validation of new version | Side-by-side Grafana panels |

```mermaid
graph LR
    subgraph "Grafana Dashboard"
        TIMELINE["Error Rate Timeline<br/>────────●──────── ← Deploy marker<br/>         ↑ error spike starts here"]
    end
```

---

## 10. Anti-Patterns

| Anti-Pattern | Consequence |
|--------------|------------|
| **Manual deployments** | Inconsistent, error-prone, slow; can't do canary or auto-rollback |
| **Big-bang multi-service deploy** | All-or-nothing coordination negates independent deployment benefit |
| **Destructive DB migration + code deploy in one step** | Cannot rollback without data loss |
| **No readiness probe** | Traffic hits pods before they're ready; errors during every deploy |
| **Rollback by deploying "previous code"** | Build from source again — slow, not identical to what was running |
| **Rollback by image tag `latest`** | Mutable tags mean you don't know what's running; use immutable sha tags |
| **No deployment annotations** | "When did this start?" becomes a 20-minute investigation |
| **Canary without automated analysis** | Human watches dashboard for 10 minutes, gets distracted, promotes bad canary |
| **Feature flags never cleaned up** | Code becomes incomprehensible; testing matrix explodes |
| **Skipping staging** | "Works on my machine" meets production traffic, data, and concurrency |

---

## 11. Recommendation: Deployment Maturity Model

| Level | Strategy | Rollback | Automation | When |
|-------|----------|----------|-----------|------|
| **1 — Basic** | Rolling update + health checks | Redeploy previous image manually | CI builds image; manual CD | Starting out; < 10 services |
| **2 — Standard** | GitOps + rolling update | `git revert` → auto-reconcile | Full CI/CD pipeline; staging gate | 10-30 services; established team |
| **3 — Advanced** | Canary + automated analysis | Auto-rollback on metric breach | Progressive delivery (Argo Rollouts/Flagger) | 30+ services; high traffic; SLOs defined |
| **4 — Elite** | Canary + feature flags + chaos testing | Instant (flag toggle or auto-rollback) | Full progressive delivery + automated chaos | High-scale; mature SRE practice |

---

## 12. Practical Checklist

```
Pipeline:
[ ] Immutable container images tagged with git SHA (never :latest in prod)
[ ] CI: build → test → SAST scan → contract test → push image
[ ] CD: dev → staging → canary prod → full prod
[ ] GitOps repo for declarative cluster state (ArgoCD / Flux)

Deployment:
[ ] Zero-downtime: maxUnavailable: 0 in rolling update or use blue-green/canary
[ ] Readiness probe configured — no traffic until service is ready
[ ] Graceful shutdown: preStop hook + SIGTERM handling + connection drain
[ ] Deploy annotations on Grafana dashboards

Database:
[ ] Additive-only migrations in the same release as code
[ ] Expand-contract for breaking schema changes (separate releases)
[ ] Never drop columns in the same deploy as code that stops using them
[ ] Migration tool (Flyway/Liquibase) runs before app starts, not during

Rollback:
[ ] Previous image always available in registry (retention policy)
[ ] Rollback tested regularly — not just theoretically possible
[ ] Automated canary analysis with rollback trigger (error rate, latency)
[ ] Feature flags for major features — toggle off without redeploy
[ ] Runbook: "How to rollback Service X" documented and practiced

Observability:
[ ] DORA metrics tracked (deploy frequency, lead time, failure rate, MTTR)
[ ] Deploy events annotated on monitoring dashboards
[ ] Canary vs stable metrics side-by-side during rollout
[ ] Alerting on canary health during progressive delivery
```

---

## 13. Next Steps

1. **What's your current CI/CD tooling?** — GitHub Actions, GitLab CI, Jenkins, ArgoCD?
2. **Deployment platform?** — Kubernetes, ECS, Lambda? Determines which progressive delivery tools are available.
3. **How often do you deploy?** — Daily? Weekly? This determines where to invest first.
4. **Do you have database migrations in your pipeline?** — Flyway/Liquibase integrated?
5. **What's your current rollback process?** — Manual? Automated? Tested?
