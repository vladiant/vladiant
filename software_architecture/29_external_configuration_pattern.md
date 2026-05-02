

## External Configuration Pattern in Microservices Architecture

### Context & Assumptions

The **External Configuration pattern** moves all environment-specific and operational configuration **out of the application code and deployment artifacts** into an external, centrally managed store. A single service binary runs unchanged across dev, staging, and production — behavior differences come entirely from **externally injected configuration**. This is fundamental to the Twelve-Factor App methodology (Factor III: Config) and essential in microservices where dozens of services, each with multiple instances across multiple environments, make embedded configuration unmanageable.

---

### The Problem: Embedded Configuration

```mermaid
graph TD
    subgraph "Without External Config"
        SRC[Source Code] --> BUILD_DEV[Build: dev artifact<br/>db.host=localhost<br/>log.level=DEBUG]
        SRC --> BUILD_STG[Build: staging artifact<br/>db.host=staging-db<br/>log.level=INFO]
        SRC --> BUILD_PROD[Build: prod artifact<br/>db.host=prod-db<br/>log.level=WARN]
    end

    style BUILD_DEV fill:#ef5350,stroke:#333,color:#fff
    style BUILD_STG fill:#ef5350,stroke:#333,color:#fff
    style BUILD_PROD fill:#ef5350,stroke:#333,color:#fff
```

| Problem | Impact |
|---|---|
| **Config baked into artifact** | Must rebuild for every environment; dev/staging/prod are different binaries |
| **Secrets in source code** | Database passwords, API keys in Git → security breach waiting to happen |
| **Config sprawl** | 50 services × 4 environments = 200 config variants to manage |
| **No runtime changes** | Changing a feature flag or log level requires redeployment |
| **No audit trail** | Who changed what config, when? |
| **Config drift** | Staging and prod slowly diverge → "works on staging, fails in prod" |

---

### The Solution: External Configuration

```mermaid
graph TD
    subgraph "With External Config"
        SRC[Source Code] --> BUILD[Build: ONE artifact<br/>No embedded config]
        BUILD --> DEV[Deploy to Dev]
        BUILD --> STG[Deploy to Staging]
        BUILD --> PROD[Deploy to Production]

        CS[(External Config Store<br/>Consul KV / Vault / etcd<br/>/ ConfigMap / Param Store)]

        CS -->|dev config| DEV
        CS -->|staging config| STG
        CS -->|prod config| PROD
    end

    style BUILD fill:#66bb6a,stroke:#333,color:#000
    style CS fill:#42a5f5,stroke:#333,color:#fff
```

**One immutable artifact** promoted through environments. Configuration is injected at **deploy time** or **runtime** from an external source.

---

### Configuration Hierarchy

```mermaid
graph TD
    subgraph "Configuration Precedence (highest wins)"
        L1[1. Runtime override<br/>API / hot-reload / feature flag] 
        L2[2. Environment variable<br/>injected at deploy]
        L3[3. External config store<br/>Consul / Vault / ConfigMap]
        L4[4. Environment-specific file<br/>application-prod.yml]
        L5[5. Default config file<br/>application.yml]
        L6[6. Code defaults<br/>Hardcoded fallbacks]
    end

    L1 --> L2 --> L3 --> L4 --> L5 --> L6

    style L1 fill:#ef5350,stroke:#333,color:#fff
    style L6 fill:#66bb6a,stroke:#333,color:#000
```

**Principle:** Defaults in code → overridden by files → overridden by external store → overridden by env vars → overridden by runtime changes. Higher layers have narrower scope but higher authority.

---

### What Belongs in External Config vs. Code

| Category | External Config ✅ | Code / Defaults ❌ → ✅ |
|---|---|---|
| **Database URLs** | `jdbc:postgresql://prod-db:5432/orders` | Connection pool defaults (`max=20`) |
| **Credentials / Secrets** | Vault / Secrets Manager | Never in code |
| **Feature flags** | `feature.new-checkout=true` | Flag evaluation logic |
| **Log level** | `log.level=WARN` | Log format, structured logging code |
| **Timeout values** | `payment.timeout-ms=3000` | Retry strategy logic |
| **Rate limits** | `api.rate-limit=100/s` | Rate limiting middleware code |
| **Service URLs** | `payment-service.url=http://...` | Service discovery logic |
| **Thread pool sizes** | `worker.pool-size=20` | Pool implementation |
| **Circuit breaker thresholds** | `cb.failure-rate=50` | Circuit breaker library |
| **Business rules** | ❌ Not usually | ✅ Business logic belongs in code |
| **Algorithms** | ❌ No | ✅ Core logic stays in code |

**Rule of thumb:** If it changes **between environments** or needs to change **without redeployment**, it's external config. If it defines **what the service does** (not how it's tuned), it's code.

---

### Configuration Injection Mechanisms

```mermaid
graph TD
    subgraph "Injection Methods"
        ENV[Environment Variables<br/>12-Factor standard]
        FILE[Mounted Config Files<br/>K8s ConfigMap volumes]
        API[Config Server API<br/>Spring Cloud Config, Consul]
        CLI[Command-Line Args<br/>--server.port=8080]
        DNS[DNS-Based<br/>SRV records, TXT records]
    end

    ENV --> APP[Application]
    FILE --> APP
    API --> APP
    CLI --> APP
    DNS --> APP

    style APP fill:#66bb6a,stroke:#333,color:#000
```

| Method | When Applied | Hot Reload? | Secret-Safe? | Best For |
|---|---|---|---|---|
| **Environment variables** | Container start | No (restart needed) | Moderate (visible in process list) | Simple key-value config, 12-Factor apps |
| **Mounted files (ConfigMap)** | Pod start (volume mount) | Yes (kubelet refresh) | No (plaintext files) | Structured config (YAML, JSON), certificates |
| **Config server API** | Runtime pull / push | Yes (watch / poll) | Yes (encrypted at rest) | Dynamic config, feature flags, centralized management |
| **Command-line args** | Process start | No | No | Override-specific values, debugging |
| **DNS/SRV** | Runtime lookup | Yes (TTL-based) | No | Service discovery endpoints |

---

### Technology Comparison

| Technology | Type | Hot Reload | Secrets | Versioning | Multi-Env | Best For |
|---|---|---|---|---|---|---|
| **Kubernetes ConfigMap** | Platform-native | Yes (volume) | No (use Secrets) | kubectl history | Namespace per env | K8s workloads, simple config |
| **Kubernetes Secrets** | Platform-native | Yes (volume) | Base64 (not encrypted by default) | kubectl history | Namespace per env | K8s secrets (with encryption at rest) |
| **HashiCorp Vault** | Dedicated secrets | Yes (agent sidecar) | Yes (encrypted, rotated) | Version history | Namespaces / paths | Secrets, dynamic credentials, PKI |
| **HashiCorp Consul KV** | Config + discovery | Yes (watch/blocking query) | No (use Vault) | No built-in | DC-aware KV paths | Key-value config, multi-DC |
| **Spring Cloud Config** | Config server | Yes (bus refresh) | Encrypt/decrypt support | Git-backed versioning | Profiles (dev, prod) | Spring Boot / JVM ecosystems |
| **AWS Systems Manager<br/>Parameter Store** | Managed KV | No (app polls) | Yes (KMS encryption) | Versioned parameters | Paths: /prod/order-svc/* | AWS-native simple config |
| **AWS Secrets Manager** | Managed secrets | Yes (rotation Lambda) | Yes (KMS, auto-rotation) | Version stages | Paths / tags | AWS-native secrets with rotation |
| **Azure App Configuration** | Managed config | Yes (SDK polling) | Key Vault references | Labels + snapshots | Labels per environment | Azure-native config + feature flags |
| **etcd** | Distributed KV | Yes (watch API) | TLS + RBAC | Revision history | Key prefixes | Low-level infrastructure config |
| **ConfigCat / LaunchDarkly** | Feature flags SaaS | Yes (streaming) | N/A | Full history | Environments built-in | Feature flags, gradual rollout |

---

### Architecture: Centralized Config in Microservices

```mermaid
graph TD
    subgraph "Configuration Infrastructure"
        GIT[Git Repository<br/>config-repo] -->|GitOps sync| CS[Config Server<br/>Spring Cloud Config<br/>/ Consul]
        VAULT[HashiCorp Vault<br/>Secrets] --> CS
        CS --> CACHE[(Local Cache<br/>per service)]
    end

    subgraph "Services"
        S1[Order Service] --> CACHE
        S2[Payment Service] --> CACHE
        S3[Inventory Service] --> CACHE
        S4[User Service] --> CACHE
    end

    subgraph "Runtime Override"
        ADMIN[Admin API / UI] -->|Change config| CS
        CS -->|Push notification<br/>or webhook| S1
        CS -->|Push notification| S2
        CS -->|Push notification| S3
    end

    style CS fill:#42a5f5,stroke:#333,color:#fff
    style VAULT fill:#ef5350,stroke:#333,color:#fff
    style GIT fill:#66bb6a,stroke:#333,color:#000
```

---

### Kubernetes ConfigMap + Secrets Flow

```mermaid
sequenceDiagram
    participant DEV as Developer
    participant GIT as Git Repo
    participant ARGO as ArgoCD / Flux
    participant K8S as Kubernetes API
    participant POD as Service Pod

    DEV->>GIT: Commit config change<br/>(ConfigMap YAML)
    GIT->>ARGO: Webhook / poll
    ARGO->>K8S: Apply ConfigMap
    K8S->>POD: Update volume mount<br/>(kubelet sync ~60s)

    Note over POD: Option A: File-watch<br/>detects change → reload

    Note over POD: Option B: Rolling restart<br/>triggered by annotation hash

    alt Secrets Change
        DEV->>GIT: Commit SealedSecret
        GIT->>ARGO: Sync
        ARGO->>K8S: Apply SealedSecret
        K8S->>K8S: Unseal → Kubernetes Secret
        K8S->>POD: Mount decrypted secret
    end
```

**Sealed Secrets / SOPS flow:** Secrets are encrypted in Git (using SealedSecrets or SOPS), decrypted only inside the cluster — secrets never appear in plaintext in source control.

---

### HashiCorp Vault Integration

```mermaid
graph TD
    subgraph "Vault Sidecar Injection"
        subgraph "Pod"
            INIT[Init Container:<br/>Vault Agent<br/>Authenticate + Fetch Secrets] --> VOL[(Shared Volume<br/>/vault/secrets)]
            APP[Application<br/>Reads /vault/secrets/db-password] --- VOL
            SIDECAR[Vault Agent Sidecar<br/>Renews lease, rotates secrets] --- VOL
        end

        VAULT[(HashiCorp Vault<br/>Dynamic Secrets Engine)]
        INIT -->|Auth: K8s ServiceAccount| VAULT
        SIDECAR -->|Renew lease every TTL| VAULT
    end

    style VAULT fill:#ef5350,stroke:#333,color:#fff
    style INIT fill:#42a5f5,stroke:#333,color:#fff
    style SIDECAR fill:#ab47bc,stroke:#333,color:#fff
```

| Vault Feature | Benefit |
|---|---|
| **Dynamic secrets** | Generate unique DB credentials per pod — auto-expire |
| **Lease management** | Credentials auto-rotate; revoke compromised creds instantly |
| **K8s auth method** | Pod authenticates via ServiceAccount — no static token |
| **Audit log** | Every secret access logged — who, when, what |
| **Transit engine** | Encrypt/decrypt without ever exposing keys to application |

---

### Hot Reload Strategies

```mermaid
graph TD
    subgraph "Hot Reload Approaches"
        A[Polling<br/>App checks config store<br/>every N seconds] -->|Simple, slight lag| APP1[Application]
        B[Watch / Streaming<br/>Config store pushes changes<br/>in real-time] -->|Low latency| APP2[Application]
        C[Webhook / Bus<br/>Config change triggers<br/>HTTP callback or event] -->|Event-driven| APP3[Application]
        D[File Watch<br/>Inotify on mounted<br/>ConfigMap volume] -->|K8s-native| APP4[Application]
        E[Rolling Restart<br/>Redeploy pod with<br/>new config hash] -->|Safest, full restart| APP5[Application]
    end

    style A fill:#66bb6a,stroke:#333,color:#000
    style B fill:#42a5f5,stroke:#333,color:#fff
    style C fill:#f9a825,stroke:#333,color:#000
    style D fill:#ff7043,stroke:#333,color:#fff
    style E fill:#ab47bc,stroke:#333,color:#fff
```

| Strategy | Latency | Complexity | Safety | Best For |
|---|---|---|---|---|
| **Polling** | Seconds (poll interval) | Low | High (controlled) | Most applications |
| **Watch / Stream** | Milliseconds | Medium | Medium | Feature flags, time-sensitive config |
| **Webhook / Bus** | Sub-second | Medium | Medium | Spring Cloud Bus, distributed refresh |
| **File watch (inotify)** | ~60s (kubelet sync) | Low | High | K8s ConfigMap-mounted config |
| **Rolling restart** | Minutes (deploy cycle) | Lowest | Highest | Critical config requiring full initialization |

**Safety concern with hot reload:** Changing a connection pool size or thread count mid-flight can cause transient errors. **Classify config as hot-reloadable vs. restart-required:**

| Hot-Reloadable ✅ | Restart Required ❌ |
|---|---|
| Log level | Database connection URL |
| Feature flags | Thread pool sizes |
| Rate limits | TLS certificates (sometimes) |
| Timeout values | Cache backend URL |
| UI text / messages | Listening port |
| Circuit breaker thresholds | JVM heap settings |

---

### Configuration Namespacing Strategy

```mermaid
graph TD
    subgraph "Config Key Hierarchy"
        ROOT["/config"]
        ROOT --> GLOBAL["/config/global<br/>shared across all services"]
        ROOT --> SVC["/config/services"]
        SVC --> ORD["/config/services/order-service"]
        ORD --> ORD_DEV["/config/services/order-service/dev"]
        ORD --> ORD_PROD["/config/services/order-service/prod"]
        SVC --> PAY["/config/services/payment-service"]
        PAY --> PAY_DEV["/config/services/payment-service/dev"]
        PAY --> PAY_PROD["/config/services/payment-service/prod"]
    end

    style GLOBAL fill:#f9a825,stroke:#333,color:#000
    style ORD_PROD fill:#42a5f5,stroke:#333,color:#fff
    style PAY_PROD fill:#42a5f5,stroke:#333,color:#fff
```

**Resolution order:** Service+env specific → service default → global → code default.

```
# Example: order-service in prod resolves "db.pool.max"
1. /config/services/order-service/prod/db.pool.max → 50   ✅ Found, use this
2. /config/services/order-service/db.pool.max → 20        (would use if #1 missing)
3. /config/global/db.pool.max → 10                         (would use if #2 missing)
4. Code default → 5                                         (would use if #3 missing)
```

---

### Secret Management Best Practices

```mermaid
graph TD
    subgraph "Secret Lifecycle"
        GEN[Generate<br/>Strong random / Vault dynamic] --> STORE[Store<br/>Vault / Secrets Manager<br/>Encrypted at rest]
        STORE --> INJECT[Inject<br/>Env var / Volume mount<br/>Sidecar / Init container]
        INJECT --> USE[Use<br/>Application reads at startup]
        USE --> ROTATE[Rotate<br/>Periodic / On-demand<br/>Zero-downtime rotation]
        ROTATE --> REVOKE[Revoke<br/>Expire / Invalidate<br/>Old credentials]
    end

    subgraph "NEVER"
        BAD1[❌ Git repository]
        BAD2[❌ Docker image layer]
        BAD3[❌ Environment variable<br/>in Dockerfile]
        BAD4[❌ Logging / error messages]
        BAD5[❌ Client-side code]
    end

    style GEN fill:#66bb6a,stroke:#333,color:#000
    style REVOKE fill:#ef5350,stroke:#333,color:#fff
    style BAD1 fill:#ef5350,stroke:#333,color:#fff
    style BAD2 fill:#ef5350,stroke:#333,color:#fff
    style BAD3 fill:#ef5350,stroke:#333,color:#fff
    style BAD4 fill:#ef5350,stroke:#333,color:#fff
    style BAD5 fill:#ef5350,stroke:#333,color:#fff
```

| Practice | Description |
|---|---|
| **Encrypt at rest** | Vault, AWS KMS, Azure Key Vault, K8s etcd encryption |
| **Encrypt in transit** | TLS between application and secret store |
| **Least privilege** | Each service accesses only its own secrets |
| **Short-lived credentials** | Vault dynamic secrets with TTL (e.g., 1-hour DB creds) |
| **Rotation without downtime** | Dual-credential overlap during rotation window |
| **Audit access** | Log every secret read — who, when, from where |
| **Sealed Secrets / SOPS in Git** | Encrypted secrets in Git; decrypted only in-cluster |
| **No secrets in env vars for sensitive workloads** | Prefer file mount — env vars visible in proc, crash dumps |

---

### Feature Flags as External Config

```mermaid
graph TD
    subgraph "Feature Flag Flow"
        ADMIN[Product Manager<br/>Dashboard] -->|Toggle flag| FF[Feature Flag Service<br/>LaunchDarkly / ConfigCat<br/>/ Unleash / Consul KV]
        FF -->|Stream / Poll| S1[Order Service<br/>if flag.new_checkout → v2 flow]
        FF -->|Stream / Poll| S2[Mobile BFF<br/>if flag.dark_mode → enable UI]
        FF -->|Stream / Poll| S3[Payment Service<br/>if flag.new_provider → Stripe v2]
    end

    style FF fill:#42a5f5,stroke:#333,color:#fff
```

Feature flags are **the most dynamic form of external configuration** — changing runtime behavior in seconds without deployment. They extend external config beyond operational tuning into **product release management**.

---

### Anti-Patterns

| Anti-Pattern | Problem | Remedy |
|---|---|---|
| **Secrets in Git** | Credentials exposed in version history forever | Use Vault, SealedSecrets, SOPS; scan with gitleaks/truffleHog |
| **Config in Docker image** | Must rebuild image per environment | Inject at runtime via env vars, volumes, or config server |
| **Shared config store, no namespacing** | Service A's change breaks Service B | Namespace by `service/environment`; RBAC per service |
| **No defaults in code** | App crashes if any config key missing | Sensible defaults for every setting; external config overrides |
| **Hot-reload everything** | Reloading DB pool config mid-transaction → errors | Classify hot-reloadable vs. restart-required config |
| **No validation on load** | Typo in config silently breaks behavior | Validate all config at startup; fail fast on invalid values |
| **Config changes without audit** | Can't determine who changed what, when | Git-backed config (GitOps) or Vault audit log |
| **Polling too frequently** | Config store overloaded by thousands of services polling every second | Use watch/streaming; poll at 30s-60s intervals; local cache |
| **Env var sprawl** | 100+ env vars per container → unmaintainable | Group into structured config files; use config server |
| **No config rollback** | Bad config deployed → manual reversal under pressure | GitOps: `git revert` → auto-apply; config server: version rollback |

---

### Config Change Safety

```mermaid
sequenceDiagram
    participant DEV as Developer
    participant GIT as Config Git Repo
    participant CI as CI/CD Pipeline
    participant STG as Staging
    participant PROD as Production

    DEV->>GIT: PR: Change payment.timeout-ms<br/>from 3000 to 5000

    GIT->>CI: Run config validation<br/>- Schema check<br/>- Range validation<br/>- Diff preview

    CI-->>GIT: Validation passed ✅

    Note over GIT: PR Review + Approve

    GIT->>CI: Merge → Apply to staging first
    CI->>STG: Deploy config to staging
    STG->>STG: Smoke tests pass ✅

    CI->>PROD: Progressive rollout<br/>10% → 50% → 100%

    Note over PROD: Monitor: error rate,<br/>latency, success rate

    alt Metrics degrade
        PROD->>CI: Auto-rollback config
    end
```

---

### Decision Framework

```mermaid
graph TD
    Q1{What type of config?} -->|Secrets / Credentials| VAULT[Vault / Secrets Manager<br/>Encrypted, audited, rotated]
    Q1 -->|Service operational config<br/>timeouts, pool sizes| Q2{Platform?}
    Q1 -->|Feature flags| FF[Feature Flag Service<br/>LaunchDarkly / Unleash / ConfigCat]
    Q1 -->|Slowly-changing reference data| DB[Database or<br/>static config file]

    Q2 -->|Kubernetes| CM[ConfigMap + Secrets<br/>+ SealedSecrets in Git]
    Q2 -->|VMs / Multi-platform| CONSUL[Consul KV / etcd<br/>+ Vault for secrets]
    Q2 -->|AWS| SSM[Parameter Store +<br/>Secrets Manager]
    Q2 -->|Spring Boot| SCC[Spring Cloud Config<br/>Git-backed + Vault]

    style VAULT fill:#ef5350,stroke:#333,color:#fff
    style FF fill:#ab47bc,stroke:#333,color:#fff
    style CM fill:#42a5f5,stroke:#333,color:#fff
    style CONSUL fill:#66bb6a,stroke:#333,color:#000
```

---

### Practical Checklist

- [ ] **Zero config in source code** — no connection strings, passwords, or API keys in Git
- [ ] **One artifact, all environments** — same Docker image across dev/staging/prod
- [ ] Build a **structured config hierarchy**: global → service → environment → instance
- [ ] **Validate config at startup** — fail fast with clear error messages on missing/invalid values
- [ ] **Sensible defaults in code** — external config overrides but app works without it
- [ ] **Separate secrets from config** — secrets in Vault/Secrets Manager, config in ConfigMap/Consul
- [ ] **Encrypt secrets at rest and in transit** — never plaintext in etcd, files, or Git
- [ ] **Classify hot-reloadable vs. restart-required** settings — document which is which
- [ ] **Audit all config changes** — Git history (GitOps) or Vault/Consul audit logs
- [ ] **GitOps for config** — config changes go through PR review, CI validation, progressive rollout
- [ ] **Monitor config freshness** — alert if a service is running with stale config
- [ ] **Secret rotation without downtime** — overlap window with dual credentials
- [ ] **Scan for secret leaks** — gitleaks, truffleHog in CI pipeline
- [ ] **RBAC on config store** — each service can read only its own namespace

---

### Recommendation

**On Kubernetes**, use **ConfigMaps** for non-sensitive config and **Sealed Secrets** (or external-secrets-operator syncing from Vault/AWS Secrets Manager) for credentials — all managed via **GitOps** (ArgoCD/Flux). Add **HashiCorp Vault** when you need dynamic secrets, automatic credential rotation, or multi-platform secret management beyond Kubernetes. For **feature flags**, use a dedicated service (Unleash for self-hosted, LaunchDarkly/ConfigCat for SaaS) — don't conflate feature flags with operational config. At minimum: every service should read config from **environment variables or mounted files**, have **validated defaults**, and **never contain secrets in source code or container images**.

---

### Next Steps to Explore

1. **GitOps for configuration** — ArgoCD/Flux managing ConfigMaps and secrets declaratively
2. **Dynamic secrets with Vault** — database credential rotation, PKI certificates
3. **Feature flag strategies** — canary rollout, percentage-based, user-targeting
4. **Config schema validation** — JSON Schema, CUE, or OPA for enforcing config contracts
5. **External Secrets Operator** — syncing secrets from Vault/AWS/Azure into Kubernetes Secrets