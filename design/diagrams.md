# Diagrams

This file collects the main system, request, deployment, observability, CI/CD, and incident-flow diagrams for the discount calculation service.

## Picture Diagrams

These SVG diagrams render as images in GitHub:

![System Architecture](assets/system-architecture.svg)

![Checkout Request Flow](assets/request-flow.svg)

![Multi-AZ Deployment Topology](assets/deployment-topology.svg)

![Latency Incident Triage](assets/incident-triage.svg)

## 1. System Architecture

```mermaid
flowchart LR
    Browser[Customer Browser] --> Checkout[Checkout Service]
    Checkout --> LB[AWS ALB / Internal Load Balancer]
    LB --> K8sSvc[Kubernetes Service]
    K8sSvc --> Pods[Discount Service Pods]

    Pods --> Redis[(ElastiCache Redis)]
    Pods --> DBProxy[RDS Proxy / PgBouncer]
    DBProxy --> Postgres[(RDS PostgreSQL Multi-AZ)]

    Pods --> OTEL[OpenTelemetry Collector]
    OTEL --> Prom[Prometheus]
    OTEL --> TraceStore[Tempo or AWS X-Ray]
    Pods --> LogStore[Loki or CloudWatch Logs]

    Prom --> Grafana[Grafana Dashboards and Alerts]
    TraceStore --> Grafana
    LogStore --> Grafana
```

## 2. Checkout Request Flow

```mermaid
sequenceDiagram
    autonumber
    participant Checkout as Checkout Service
    participant Discount as Discount Service
    participant Redis as Redis Cache
    participant DB as PostgreSQL

    Checkout->>Discount: calculate_cart_discounts(cart, customer, payment)
    Discount->>Discount: Validate request and normalize inputs
    Discount->>Redis: Lookup computed result by cart/customer/payment/rule_version

    alt Computed result cache hit
        Redis-->>Discount: DiscountedPrice
        Discount-->>Checkout: Return cached DiscountedPrice
    else Computed result cache miss
        Discount->>Redis: Lookup active rules by rule_version
        alt Active rules cache hit
            Redis-->>Discount: Active rules
        else Active rules cache miss
            Discount->>DB: Query active rules by brand/category/bank/voucher
            DB-->>Discount: Matching rules
            Discount->>Redis: Cache active rules with versioned key
        end
        Discount->>Discount: Apply rules and caps deterministically
        Discount->>Redis: Cache computed result with short TTL
        Discount-->>Checkout: Return DiscountedPrice
    end
```

## 3. Multi-AZ Kubernetes Deployment

```mermaid
flowchart TB
    subgraph Region[AWS Region]
        subgraph AZ1[Availability Zone 1]
            NodeA[EKS Nodes]
            PodA[Discount Pods]
            NodeA --> PodA
        end

        subgraph AZ2[Availability Zone 2]
            NodeB[EKS Nodes]
            PodB[Discount Pods]
            NodeB --> PodB
        end

        subgraph AZ3[Availability Zone 3]
            NodeC[EKS Nodes]
            PodC[Discount Pods]
            NodeC --> PodC
        end

        HPA[HPA: CPU + In-flight Requests] --> PodA
        HPA --> PodB
        HPA --> PodC

        PDB[PodDisruptionBudget minAvailable 80%] --> PodA
        PDB --> PodB
        PDB --> PodC

        PodA --> Redis[(Redis Primary + Replica)]
        PodB --> Redis
        PodC --> Redis

        PodA --> Pool[RDS Proxy / PgBouncer]
        PodB --> Pool
        PodC --> Pool
        Pool --> RDS[(RDS PostgreSQL Multi-AZ)]
    end
```

## 4. Observability Data Flow

```mermaid
flowchart LR
    App[Discount Service Pods] --> Logs[Structured JSON Logs]
    App --> Metrics[Prometheus Metrics]
    App --> Traces[OpenTelemetry Traces]

    Logs --> Loki[Loki / CloudWatch Logs]
    Metrics --> Prometheus[Prometheus]
    Traces --> Collector[OpenTelemetry Collector]
    Collector --> Tempo[Tempo / X-Ray]

    Loki --> Grafana[Grafana]
    Prometheus --> Grafana
    Tempo --> Grafana

    Grafana --> Alerts[Alertmanager / PagerDuty]
    Alerts --> OnCall[On-call Engineer]
```

## 5. Grafana Dashboard Layout

```mermaid
flowchart TB
    subgraph Row1[Top Row: User-Visible Health]
        L1[p99 / p95 / p50 Latency]
        L2[Request Rate]
        L3[Error Rate]
        L4[SLO Burn Rate]
    end

    subgraph Row2[Middle Row: Dependencies]
        D1[Redis Hit Ratio]
        D2[Redis Command Latency]
        D3[PostgreSQL Query p99]
        D4[DB Pool Saturation]
    end

    subgraph Row3[Bottom Row: Business Correctness]
        B1[Voucher Accept / Reject Rate]
        B2[Discount Amount by Rule Version]
        B3[Top Applied Rules]
        B4[Fallback Rate]
    end

    Row1 --> Row2
    Row2 --> Row3
```

## 6. CI/CD And Progressive Delivery

```mermaid
flowchart LR
    Dev[Developer PR] --> Checks[Lint, Type Checks, Unit Tests]
    Checks --> Security[Secret Scan, SAST, Dependency Scan]
    Security --> Logic[Golden Discount Logic Tests]
    Logic --> Build[Build Signed Container Image]
    Build --> Stage[Deploy to Staging]
    Stage --> Smoke[Smoke and Integration Tests]
    Smoke --> Canary1[Canary 1%]
    Canary1 --> Canary5[Canary 5%]
    Canary5 --> Canary25[Canary 25%]
    Canary25 --> Full[100% Rollout]

    Canary1 --> Guardrails{Latency, Error, Fallback, Discount Guardrails}
    Canary5 --> Guardrails
    Canary25 --> Guardrails
    Guardrails -->|Fail| Rollback[Rollback App or Rule Version]
    Guardrails -->|Pass| Full
```

## 7. Latency Incident Triage

```mermaid
flowchart TD
    Alert[p99 latency > 150 ms for 10 minutes] --> RecentChange{Recent app deploy or rule activation?}

    RecentChange -->|Yes| ChangeType{App or rule?}
    ChangeType -->|App| AppRollback[Rollback app image]
    ChangeType -->|Rule| RuleRollback[Deactivate rule version and invalidate cache]

    RecentChange -->|No| RedisCheck{Redis latency high or hit ratio falling?}
    RedisCheck -->|Yes| RedisMitigation[Use stale/local rules, reduce cache writes, scale Redis]

    RedisCheck -->|No| DBCheck{PostgreSQL query p99 or pool wait high?}
    DBCheck -->|Yes| DBMitigation[Find slow query, disable bad rule, cap DB concurrency]

    DBCheck -->|No| AppCapacity{CPU throttling or in-flight requests high?}
    AppCapacity -->|Yes| ScalePods[Increase HPA min replicas or CPU limit]
    AppCapacity -->|No| TraceDrilldown[Inspect traces and checkout dependency timing]

    RedisMitigation --> Watch[Watch checkout latency and fallback rate]
    DBMitigation --> Watch
    ScalePods --> Watch
    AppRollback --> Watch
    RuleRollback --> Watch
```

## 8. Dependency Degradation Behavior

```mermaid
flowchart TD
    Request[Discount Request] --> RedisAvailable{Redis available?}

    RedisAvailable -->|Yes| CacheHit{Cache hit?}
    CacheHit -->|Yes| ReturnCached[Return cached result]
    CacheHit -->|No| DBAvailable{PostgreSQL available and fast?}

    RedisAvailable -->|No| LocalCache{Local active-rule cache available?}
    LocalCache -->|Yes| UseLocal[Use local active rules for bounded TTL]
    LocalCache -->|No| DBAvailable

    DBAvailable -->|Yes| QueryDB[Query active rules and compute result]
    QueryDB --> CacheResult[Cache result and return]

    DBAvailable -->|No| VoucherPresent{Voucher validation required?}
    VoucherPresent -->|Yes| VoucherRetry[Return retryable voucher validation failure]
    VoucherPresent -->|No| GenericFallback[Return price with no optional promotion or stale non-voucher rules]

    UseLocal --> Compute[Compute with local active rules]
    Compute --> ReturnDegraded[Return response with fallback_applied=true]
```
