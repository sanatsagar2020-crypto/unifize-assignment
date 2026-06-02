# Discount Service Production Design

## Context

The discount calculation service is a Python service on the synchronous checkout path for a fashion e-commerce platform. It calculates brand discounts, bank card offers, category deals, and voucher codes. It reads discount rules from PostgreSQL and uses Redis for hot rules and computed-result caching.

## Production Goals

- Keep checkout available during dependency degradation.
- Keep p99 latency below 150 ms during steady and sale-event traffic.
- Make discount correctness failures visible, not only infrastructure failures.
- Roll out discount logic and rule changes safely because mistakes directly affect revenue and customer trust.

## Assumptions

| Area | Assumption |
|---|---|
| Cloud | AWS, single region, multi-AZ |
| Compute | EKS |
| Database | RDS PostgreSQL Multi-AZ |
| Cache | ElastiCache Redis |
| Observability | OpenTelemetry, Prometheus, Grafana, Loki or CloudWatch Logs, Tempo or X-Ray |
| Steady traffic | 2,000 RPS |
| Peak traffic | 20,000 RPS for 4-6 hours |
| Latency target | p99 under 150 ms |
| Availability model | Single region, multi-AZ for v1 |

## Design Priorities

1. Availability on checkout path.
2. Low tail latency under bursty sale traffic.
3. Guardrails for discount correctness and voucher validation.
4. Fast rollback for app logic and rule versions.
5. Explicit capacity math and known bottlenecks.

## Out Of Scope For V1

- Active-active multi-region.
- Full Terraform/IaC implementation.
- Exact database schema and index DDL.
- Fraud controls for voucher-code enumeration.
- Implementation code.

## Files

| File | Purpose |
|---|---|
| [diagrams.md](diagrams.md) | Central diagram pack for architecture, request flow, deployment, observability, CI/CD, and incident response |
| [01-architecture.md](01-architecture.md) | System components, request flow, and topology diagrams |
| [02-observability.md](02-observability.md) | Logs, metrics, tracing, and dashboard plan |
| [03-slo-error-budget.md](03-slo-error-budget.md) | SLIs, SLOs, and error budget policy |
| [04-deployment-resilience.md](04-deployment-resilience.md) | Kubernetes deployment design and failure modes |
| [05-cicd-change-safety.md](05-cicd-change-safety.md) | Pipeline, rollout, rollback, and supply chain controls |
| [06-capacity-cost.md](06-capacity-cost.md) | Resource estimates, bottlenecks, and cost optimization |
| [07-runbook-latency.md](07-runbook-latency.md) | 3 AM runbook for p99 latency breach |
