# Architecture

## Component Choices

| Component | Choice | Reason |
|---|---|---|
| Compute | EKS | Horizontal scaling, controlled rollouts, pod disruption controls |
| Ingress | AWS ALB | Managed L7 routing from checkout-facing services |
| Database | RDS PostgreSQL Multi-AZ | Strong consistency for discount rules and managed failover |
| DB pooling | RDS Proxy or PgBouncer | Protects PostgreSQL from pod-scaling connection storms |
| Cache | ElastiCache Redis | Low-latency cache for active rules and computed cart results |
| Metrics | Prometheus | Kubernetes-native scraping and histogram/SLO support |
| Dashboards | Grafana | On-call dashboards and alert visualization |
| Logs | Loki or CloudWatch Logs | JSON log search by request, trace, rule version, and outcome |
| Traces | OpenTelemetry with Tempo or X-Ray | Request-level visibility through cache, DB, and calculation spans |
| Delivery | GitHub Actions and Argo Rollouts | CI gates and progressive production rollout |
| Secrets | AWS Secrets Manager with External Secrets Operator | Rotation-friendly secret delivery to Kubernetes |

## High-Level Architecture

```mermaid
flowchart LR
    Customer[Customer Browser] --> Checkout[Checkout Service]
    Checkout --> ALB[AWS ALB]
    ALB --> K8sSvc[Kubernetes Service]
    K8sSvc --> Pods[Discount Service Pods]

    Pods --> Redis[(ElastiCache Redis)]
    Pods --> PgPool[RDS Proxy / PgBouncer]
    PgPool --> Postgres[(RDS PostgreSQL Multi-AZ)]

    Pods --> OTEL[OpenTelemetry Collector]
    OTEL --> Metrics[Prometheus]
    OTEL --> Traces[Tempo or X-Ray]
    Pods --> Logs[Loki or CloudWatch Logs]

    Metrics --> Grafana[Grafana Dashboards + Alerts]
    Traces --> Grafana
    Logs --> Grafana
```

## Request Flow

```mermaid
sequenceDiagram
    participant C as Checkout Service
    participant D as Discount Service
    participant R as Redis
    participant P as PostgreSQL

    C->>D: calculate_cart_discounts(cart, customer, payment)
    D->>D: Validate request and build lookup key
    D->>R: Get computed result / active rules
    alt cache hit
        R-->>D: Cached discount result or rules
    else cache miss
        D->>P: Query active discount rules
        P-->>D: Matching rules
        D->>D: Calculate final price deterministically
        D->>R: Cache result with TTL and rule version
    end
    D-->>C: DiscountedPrice
```

## Kubernetes Topology

```mermaid
flowchart TB
    subgraph AWS Region
      subgraph AZ1
        P1[Discount Pods]
      end
      subgraph AZ2
        P2[Discount Pods]
      end
      subgraph AZ3
        P3[Discount Pods]
      end

      SVC[Kubernetes Service] --> P1
      SVC --> P2
      SVC --> P3

      P1 --> Redis[(Redis Primary/Replica)]
      P2 --> Redis
      P3 --> Redis

      P1 --> DBPool[RDS Proxy / PgBouncer]
      P2 --> DBPool
      P3 --> DBPool
      DBPool --> RDS[(RDS PostgreSQL Multi-AZ)]
    end
```

## Failure Decision Flow

```mermaid
flowchart TD
    Alert[Latency or error alert] --> RecentChange{Recent app or rule change?}
    RecentChange -->|Yes| Rollback[Rollback app or deactivate rule version]
    RecentChange -->|No| RedisCheck{Redis latency or evictions high?}
    RedisCheck -->|Yes| RedisMitigate[Use stale/local rules, reduce writes, scale Redis]
    RedisCheck -->|No| DBCheck{DB query p99 or pool wait high?}
    DBCheck -->|Yes| DBMitigate[Identify query, disable bad rule, cap DB concurrency]
    DBCheck -->|No| AppCheck{CPU or in-flight requests saturated?}
    AppCheck -->|Yes| Scale[Increase replicas or CPU, check throttling]
    AppCheck -->|No| Trace[Inspect traces and checkout dependency timing]
```

