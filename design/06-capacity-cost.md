# Capacity And Cost

## Traffic Inputs

| Scenario | RPS |
|---|---:|
| Steady state | 2,000 |
| Peak sale event | 20,000 |

Target p99 latency: under 150 ms.

## Pod Estimate

Planning assumption:

- One pod with 500 millicores CPU request and 1 CPU limit handles about 100 RPS while keeping p99 below 100 ms.
- This must be validated by load testing.

```text
steady pods = 2,000 RPS / 100 RPS per pod = 20 pods
steady with 20% headroom = 24 pods

peak pods = 20,000 RPS / 100 RPS per pod = 200 pods
peak with 20% headroom = 240 pods
```

CPU and memory:

```text
steady CPU requested = 24 pods * 0.5 vCPU = 12 vCPU
steady memory requested = 24 pods * 512 MiB = 12 GiB

peak CPU requested = 240 pods * 0.5 vCPU = 120 vCPU
peak memory requested = 240 pods * 512 MiB = 120 GiB
```

## PostgreSQL Estimate

Assumptions:

- Active-rule cache hit ratio: 95% steady.
- Pre-warmed sale cache hit ratio: 98%.
- Cache miss requires 1-2 PostgreSQL queries.

```text
steady DB read QPS = 2,000 * 5% * 2 = 200 QPS
peak DB read QPS = 20,000 * 2% * 2 = 800 QPS
```

Degraded Redis scenario:

```text
peak degraded DB read QPS = 20,000 * 20% * 2 = 8,000 QPS
```

Sizing:

- Start with RDS PostgreSQL around 8-16 vCPU, then validate using load test and Performance Insights.
- Discount rule storage is likely small; IOPS and query plans are the main constraints.
- Example storage estimate: 1 million historical rule rows at 1 KiB plus indexes and audit overhead is roughly 5-10 GiB.

## Redis Estimate

Assumptions:

- Active rules: 100,000 rules * 2 KiB serialized = 200 MiB.
- Computed cart cache at peak: 2 million hot keys * 1 KiB = 2 GiB.
- Overhead and fragmentation: 2x.

```text
Redis memory = (200 MiB + 2 GiB) * 2 = about 4.4 GiB
```

Initial recommendation:

- Use at least 8-16 GiB usable Redis memory.
- Separate active-rule and computed-result keyspaces.
- Pre-warm active rules before sale events.

## First Bottleneck

The first likely bottleneck at peak is dependency amplification from cache misses:

1. Redis latency or evictions reduce cache hit ratio.
2. PostgreSQL read QPS rises sharply.
3. DB pool wait time increases.
4. Service p99 rises.
5. Checkout starts timing out.

Mitigation:

- Pre-warm active rules before sales.
- Use local in-process active-rule cache.
- Cap DB concurrency per pod.
- Use bounded stale active rules instead of hammering PostgreSQL.
- Add read replicas only after confirming consistency tolerance.

## Cost Optimization

Proposal:

- Keep active rules in local in-process cache with short TTL and rule-version checks.
- Keep Redis as the cross-pod cache and invalidation/version source.

Estimated impact:

- At 20,000 RPS, avoiding one Redis active-rule lookup per request saves up to 20,000 Redis ops/sec.
- A 60-second local TTL with rule-version invalidation can reduce active-rule Redis lookups by 80-95%.
- This can delay Redis vertical scaling and improve p99 latency.

Trade-off:

- Local cache adds consistency complexity. Rule-versioned keys and emergency invalidation are required for rollback.

