# Runbook: Discount Calculation p99 Latency Breach

## Scenario

Alert: `calculate_cart_discounts` p99 latency has exceeded 150 ms for 10 minutes.

## Symptoms

- Discount service p99 latency above SLO.
- Checkout latency rising.
- Customers may report slow checkout or timeouts.
- Error rate may still be normal if requests are slow but eventually succeed.

## Confirm Impact

Check Grafana Discount Service dashboard:

1. p99 and p95 latency for `calculate_cart_discounts`.
2. Request rate by operation.
3. Error rate and fallback rate.
4. Redis command latency, hit ratio, and evictions.
5. PostgreSQL query p99 by query type.
6. DB pool usage and wait time.
7. Recent app deployments and rule-version changes.

Prometheus queries:

```promql
histogram_quantile(0.99, rate(discount_request_duration_seconds_bucket{operation="calculate_cart_discounts"}[5m]))
```

```promql
histogram_quantile(0.99, rate(discount_db_query_duration_seconds_bucket[5m]))
```

```promql
sum(rate(discount_fallback_total[5m])) by (fallback_type)
```

```promql
sum(rate(discount_requests_total{outcome!="success"}[5m]))
/
sum(rate(discount_requests_total[5m]))
```

## Likely Causes

1. Redis hit ratio dropped or Redis latency increased, causing PostgreSQL amplification.
2. A new discount rule or query path caused slow PostgreSQL scans.
3. Recent app deployment introduced inefficient logic or N+1 queries.
4. HPA did not scale fast enough for a traffic spike.
5. CPU throttling or node pressure.

## Mitigation

Step 1: Identify whether the bottleneck is Redis, PostgreSQL, or app capacity.

- Redis issue: high command latency, evictions, or falling hit ratio.
- PostgreSQL issue: high query p99, high pool wait, high connection count.
- App capacity issue: high in-flight requests, CPU throttling, dependency metrics normal.
- Change issue: latency increase aligns with deployment or rule activation.

Step 2: Redis mitigation.

- Enable bounded stale active-rule cache if not already active.
- Temporarily reduce computed-result cache writes if Redis CPU is saturated.
- Scale Redis if an approved scaling path exists.
- Watch PostgreSQL QPS because Redis mitigation can shift load.

Step 3: PostgreSQL mitigation.

- Identify slow `query_type` from dashboard and traces.
- Check recent rule changes for that query type.
- If tied to a new rule version, deactivate that rule version.
- If DB pool is saturated, reduce per-pod DB concurrency or enable stale-rule fallback.
- Do not add pods if DB is already the bottleneck.

Step 4: App scaling mitigation.

- Manually raise HPA minimum replicas if dependencies are healthy.
- Confirm pods are spread across AZs.
- Check CPU throttling. If high, increase CPU limits or reduce per-pod concurrency.

Step 5: Rollback.

- If latency started after app deployment, roll back to the previous stable image.
- If latency started after rule activation, deactivate the rule version and invalidate affected cache keys.
- Target app rollback time: under 10 minutes.
- Target rule rollback time: under 5 minutes.

## Escalation

- Page service owner if p99 remains above 150 ms after 15 minutes.
- Page database owner if PostgreSQL p99 remains above 50 ms or pool wait is saturated.
- Page platform owner if pods cannot scale or nodes are under pressure.
- Notify checkout/product channel if degraded behavior changes customer-visible discounts.

## Customer-Safe Degraded Mode

- Continue applying recently cached non-voucher rules for up to 15 minutes.
- Do not accept unknown voucher codes without authoritative validation.
- If voucher validation times out, return a retryable voucher validation message through checkout instead of blocking all checkout where possible.

## Post-Incident Actions

- Write timeline with deployment and rule-version correlation.
- Add missing alert if customers noticed before monitoring.
- Add golden test or query-plan check for the root cause.
- Review cache TTLs, pre-warming, and HPA minimums for sale events.
- Update this runbook with the actual fix.

