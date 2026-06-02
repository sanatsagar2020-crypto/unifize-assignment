# Deployment And Resilience Design

## Kubernetes Deployment

Initial deployment:

- Minimum replicas: 24 pods.
- Planned peak replicas: 220-260 pods.
- Spread pods across 3 AZs using topology spread constraints.
- Pod request: 500 millicores CPU, 512 MiB memory.
- Pod limit: 1 CPU, 1 GiB memory.
- Per-pod expected throughput: 100 RPS at p99 under 100 ms, subject to load-test validation.

HPA:

- Scale on CPU and request concurrency.
- Target CPU: 60% of requested CPU.
- Target in-flight requests: 80 per pod.
- Scale-up stabilization: 30 seconds.
- Scale-down stabilization: 10 minutes.
- Pre-scale before known sale events to at least 180 pods.

Reasoning:

- Python services can hit event-loop saturation, DB pool wait, or CPU throttling before CPU reaches tutorial-style thresholds like 70-80%.
- In-flight requests are closer to customer-visible latency than CPU alone.
- Pre-scaling is required because a sale spike can arrive faster than reactive autoscaling.

Probes:

- Startup probe: app initialized, config loaded, telemetry exporter non-blocking.
- Readiness probe: service can accept requests and circuit breakers are not globally open.
- Liveness probe: detects stuck process/event loop only. It must not restart pods just because PostgreSQL is slow.

PDB:

- `minAvailable: 80%` during normal operations.
- Node upgrades must preserve AZ spread.

Connection pools:

- App DB pool: 5-10 connections per pod.
- RDS Proxy or PgBouncer protects PostgreSQL from total pod count.
- Redis pool is bounded with short timeouts and backpressure.

## Failure Mode 1: Slow Rule Query After New Offer

Scenario:

A new bank-card offer adds predicates on `bank_name`, `card_type`, `category`, and validity windows. PostgreSQL chooses a sequential scan on `discount_rules`. `db.query_discount_rules` p99 jumps from 15 ms to 120 ms, and cache misses breach the 150 ms service target.

Mitigations:

- Query metrics and traces tagged by `query_type`.
- `EXPLAIN ANALYZE` review for rule-query changes.
- Composite indexes for active status, validity window, discount type, brand/category/bank fields.
- Active rules cached by rule version and discount type.
- Canary new rules by rule version before full activation.
- Alert when `discount_db_query_duration_seconds` p99 exceeds 50 ms for 10 minutes.

## Failure Mode 2: Redis Hot Key Or Eviction Storm

Scenario:

Popular carts or sitewide discounts create hot Redis keys. Redis CPU rises, command latency increases, and computed-result keys are evicted. PostgreSQL receives amplified traffic from cache misses.

Mitigations:

- Separate namespaces and TTLs for active rules and computed cart results.
- Keep active-rule cache high priority with longer TTL and versioned keys.
- Add TTL jitter to computed-result keys.
- Use local in-process short-TTL cache for active rules.
- Monitor Redis evictions, CPU, command latency, and hit ratio.
- Circuit break Redis calls above 10-20 ms and use local/stale rules when safe.

## Failure Mode 3: Bad Discount Logic Rollout

Scenario:

A code change applies bank offers before category caps. Availability and latency remain normal, but average discount amount jumps by 40%, leaking revenue.

Mitigations:

- Golden tests using historical carts and expected outputs.
- Property tests for invariants: final price never negative, caps enforced, mutually exclusive rules not stacked.
- Shadow evaluation on production traffic before active rollout.
- Canary by pod percentage and rule version.
- Guardrail alerts on discount amount by rule version and discount type.
- Fast rollback of app version and independent deactivation of rule version.

## Dependency Degradation

PostgreSQL:

- If PostgreSQL is unavailable but Redis has current active rules, continue with cached rules for TTL plus a bounded stale window, for example 15 minutes.
- For generic discounts, return no optional promotion if rules cannot be loaded.
- For voucher validation, do not accept unknown voucher codes without authoritative validation.

Redis:

- Query PostgreSQL directly with strict pool limits and timeouts.
- Use local in-process active-rule cache briefly.
- Shed computed-result cache writes first.
- Alert immediately because Redis failure can overload PostgreSQL at peak.

Checkout timeout:

- Discount service internal timeouts must be shorter than checkout's deadline.
- Example budget: Redis timeout 20 ms, PostgreSQL timeout 75 ms, total service timeout below checkout's allowed budget.

Acceptable degraded behavior:

- Apply recently cached non-voucher rules for a bounded stale window.
- Return no optional generic promotion when rules are unavailable.
- Return retryable voucher validation failure if authoritative validation is unavailable.

Unacceptable degraded behavior:

- Silently accept unknown voucher codes.
- Return inconsistent prices for the same cart and rule version.

