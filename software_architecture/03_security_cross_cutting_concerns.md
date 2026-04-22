# How to handle cross-cutting concerns, such as security, in a Microservices architecture?

Cross-cutting concerns — security, logging, tracing, rate limiting, configuration — are the hardest part of microservices because they **span every service** but shouldn't be **duplicated in every service**. The architectural challenge is: how do you enforce consistency without creating coupling?

---

## 1. The Cross-Cutting Concerns Landscape

| Concern | What It Covers |
|---------|---------------|
| **Authentication & Authorization** | Who is calling? Are they allowed? |
| **Observability** | Logging, distributed tracing, metrics |
| **Resilience** | Retries, circuit breakers, timeouts, bulkheads |
| **Traffic management** | Rate limiting, throttling, load balancing |
| **Configuration** | Centralized config, feature flags, secrets |
| **Data concerns** | Encryption in transit/at rest, data masking, audit trails |

---

## 2. Options Analysis

### Option A: Shared Library / SDK Approach

Each service includes a shared library that implements the concerns.

```mermaid
graph TB
    subgraph "Shared SDK"
        SDK[Cross-Cutting Library<br/>Auth · Logging · Tracing · Resilience]
    end

    S1[Service A<br/>Java] -- "imports" --> SDK
    S2[Service B<br/>.NET] -- "imports" --> SDK
    S3[Service C<br/>Go] -- "imports" --> SDK
```

| Criterion | Assessment |
|-----------|-----------|
| **Coupling** | High — all services depend on the library; version upgrades require redeployment |
| **Polyglot support** | Poor — must maintain one SDK per language |
| **Consistency** | Strong — if everyone uses the same version |
| **Operational cost** | Low infrastructure, high maintenance |
| **Flexibility** | Services can customize locally |

### Option B: API Gateway + Service Mesh (Infrastructure-Level)

Offload cross-cutting concerns to **infrastructure** — the application code stays clean.

```mermaid
graph TB
    Client[Client] --> GW[API Gateway<br/>Kong / Envoy / AWS API GW]

    subgraph "Service Mesh (Istio / Linkerd)"
        direction TB
        S1P[Sidecar Proxy] --- S1[Service A]
        S2P[Sidecar Proxy] --- S2[Service B]
        S3P[Sidecar Proxy] --- S3[Service C]
        S1P <--> S2P
        S2P <--> S3P
    end

    GW --> S1P
    GW --> S2P

    CP[Control Plane<br/>mTLS · Retries · Tracing · Rate Limits] -.-> S1P
    CP -.-> S2P
    CP -.-> S3P
```

| Criterion | Assessment |
|-----------|-----------|
| **Coupling** | Very low — concerns live in infrastructure, not code |
| **Polyglot support** | Excellent — sidecar is language-agnostic |
| **Consistency** | Enforced at infrastructure level — services can't bypass |
| **Operational cost** | High — must operate the mesh and gateway |
| **Flexibility** | Policy-driven configuration, no code changes |

### Option C: Hybrid (Gateway + Lightweight In-Process Middleware)

API Gateway handles **edge concerns**, in-process middleware handles **domain-aware concerns**.

```mermaid
graph TB
    Client[Client] --> GW[API Gateway]

    subgraph "Edge Concerns (Gateway)"
        GW --> Auth[Authentication<br/>JWT Validation]
        GW --> RL[Rate Limiting]
        GW --> CORS[CORS]
        GW --> TLS[TLS Termination]
    end

    subgraph "Service A"
        MW1[Middleware Pipeline] --> Authz1[Authorization<br/>Domain-level RBAC]
        MW1 --> Log1[Structured Logging]
        MW1 --> Trace1[Trace Propagation]
        MW1 --> BL1[Business Logic]
    end

    Auth --> MW1
```

| Criterion | Assessment |
|-----------|-----------|
| **Coupling** | Low — gateway owns edge; services own domain-specific policies |
| **Polyglot support** | Good — each runtime has its own middleware idiom |
| **Consistency** | Edge concerns enforced centrally; domain concerns need discipline per service |
| **Operational cost** | Medium — gateway is simpler than a full mesh |
| **Flexibility** | Best balance — coarse-grained at edge, fine-grained in-process |

---

## 3. Comparison

| Criterion | Shared Library | Gateway + Mesh | Hybrid |
|-----------|---------------|----------------|--------|
| **Coupling** | High | Very Low | Low |
| **Polyglot** | Poor | Excellent | Good |
| **Enforcement** | Opt-in | Mandatory | Edge: mandatory / Domain: opt-in |
| **Operational complexity** | Low | High | Medium |
| **Latency overhead** | None | Sidecar hop (~1ms) | Gateway hop only |
| **Team autonomy** | Low (forced upgrades) | High | High |
| **Cost** | Low | High (mesh infra) | Medium |

---

## 4. Recommendation: Hybrid Approach

Most production systems land here. Split concerns by **where they belong**:

| Layer | Concern | Implementation |
|-------|---------|---------------|
| **API Gateway** | Authentication (JWT/OAuth2 validation) | Kong, Envoy, AWS API GW, Azure APIM |
| **API Gateway** | Rate limiting, throttling | Gateway policies |
| **API Gateway** | TLS termination, CORS | Gateway config |
| **API Gateway** | Request ID injection | Custom header (e.g., `X-Request-Id`) |
| **In-Process Middleware** | Authorization (RBAC/ABAC) | Spring Security, ASP.NET Authorization |
| **In-Process Middleware** | Structured logging | SLF4J/Logback, Serilog — emit JSON with trace context |
| **In-Process Middleware** | Trace propagation | OpenTelemetry SDK (Java & .NET agents) |
| **Infrastructure** | mTLS (service-to-service encryption) | Service mesh OR cert-manager + sidecars |
| **Infrastructure** | Centralized config & secrets | Consul, Vault, AWS Secrets Manager, Azure Key Vault |
| **Infrastructure** | Metrics collection | Prometheus + OTel Collector |

---

## 5. Deep Dive: Security Specifically

Security is the most critical cross-cutting concern. Here's the layered model:

```mermaid
graph TB
    subgraph "Layer 1: Edge Security"
        GW[API Gateway] --> JWT[JWT Validation<br/>Token signature + expiry]
        GW --> RL[Rate Limiting<br/>DDoS protection]
        GW --> WAF[WAF Rules<br/>OWASP Top 10]
        GW --> TLS[TLS 1.3<br/>Encryption in transit]
    end

    subgraph "Layer 2: Service-Level Security"
        AUTH[Authorization Middleware<br/>RBAC / ABAC / Policy Engine] --> BL[Business Logic]
        BL --> VAL[Input Validation<br/>Schema-level + domain-level]
        BL --> AUDIT[Audit Logging<br/>Who did what, when]
    end

    subgraph "Layer 3: Data Security"
        ENC[Encryption at Rest<br/>DB-level + field-level]
        MASK[Data Masking<br/>PII in logs]
        SEC[Secrets Management<br/>Vault / KMS]
    end

    subgraph "Layer 4: Network Security"
        MTLS[mTLS Between Services]
        NP[Network Policies<br/>Namespace isolation]
        ZT[Zero Trust<br/>No implicit trust]
    end

    JWT --> AUTH
    AUTH --> ENC
    MTLS -.-> AUTH
```

### Token Propagation Pattern

```mermaid
sequenceDiagram
    participant C as Client
    participant GW as API Gateway
    participant A as Service A
    participant B as Service B

    C->>GW: Request + Bearer Token (JWT)
    GW->>GW: Validate JWT signature & expiry
    GW->>A: Request + JWT + X-Request-Id
    A->>A: Extract claims, check authorization
    A->>B: Internal call + JWT (or scoped token) + X-Request-Id
    B->>B: Validate JWT, check fine-grained permissions
    B-->>A: Response
    A-->>GW: Response
    GW-->>C: Response
```

**Key decisions:**
- **Propagate the original JWT** for simplicity, or **exchange for an internal token** (token exchange / RFC 8693) if you need to scope down permissions for internal calls.
- Use **OPA (Open Policy Agent)** or a similar policy engine if authorization rules are complex and shared across services — avoids duplicating policy logic.

---

## 6. Anti-Patterns

| Anti-Pattern | Why It's Dangerous |
|--------------|-------------------|
| Each service implements its own auth | Inconsistent enforcement, easy to miss a service |
| Passing plaintext credentials between services | One compromised service leaks everything |
| Logging PII without masking | Compliance violations (GDPR, HIPAA) |
| No distributed tracing | Impossible to debug cross-service security incidents |
| Shared secrets across services | Blast radius is the entire system |
| Trusting internal network implicitly | Lateral movement after any breach |

---

## 7. Next Steps

1. **What's your current gateway?** — Already have one, or greenfield?
2. **Identity provider** — Keycloak, Auth0, Azure AD, Okta?
3. **Are you on Kubernetes?** — A service mesh becomes much more practical there.
4. **Compliance requirements?** — GDPR, HIPAA, PCI-DSS each drive specific security patterns.
5. **How many services?** — Under ~10, a gateway + middleware is plenty; 50+, a service mesh starts paying for itself.
