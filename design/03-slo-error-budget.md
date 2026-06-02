# SLI, SLO, And Error Budget Policy

## SLI 1: Availability

Measurement:

- Event: every completed request to `calculate_cart_discounts` or `validate_discount_code`.
- Success: valid application response within 500 ms and no internal error outcome.
- Failure: 5xx, timeout, dependency failure without valid fallback, malformed response.
- Window: rolling 30 days, with 5-minute burn-rate alerting.

SLO:

- `calculate_cart_discounts`: 99.95% monthly availability.
- `validate_discount_code`: 99.9% monthly availability.

Justification:

- Checkout depends synchronously on the service.
- Voucher validation can have a slightly lower SLO because customers can sometimes checkout without a voucher, but accepting invalid voucher codes is not an acceptable fail-open path.

## SLI 2: Latency

Measurement:

- Event: every successful `calculate_cart_discounts` request.
- Success: p99 duration under 150 ms.
- Alert window: rolling 5 minutes.
- Reporting window: rolling 30 days.

Prometheus:

```promql
histogram_quantile(
  0.99,
  rate(discount_request_duration_seconds_bucket{operation="calculate_cart_discounts"}[5m])
)
```

SLO:

- p99 under 150 ms for 99% of 5-minute windows over 30 days.

Justification:

- Checkout conversion is sensitive to tail latency, not average latency.
- Five-minute windows catch sale-event degradation quickly without paging on isolated spikes.

## SLI 3: Discount Correctness Proxy

Measurement:

- Event: every request where a rule version is applied.
- Success:
  - No calculation exception.
  - No invalid rule state detected.
  - Voucher rejection rate stays within expected baseline.
  - Discount amount distribution by rule version stays within configured guardrails.
- Window: 30 minutes for anomaly detection, 30 days for reporting.

SLO:

- 99.99% of requests do not hit a known correctness failure.

Justification:

- Correctness is business-critical and cannot be fully captured by latency/availability.
- A bad rule can leak revenue or overcharge customers while infrastructure dashboards remain green.

## Error Budget

For 99.95% monthly availability:

```text
30 days = 43,200 minutes
Allowed unavailability = 0.05% = 21.6 minutes/month
```

| Budget State | Operational Policy |
|---|---|
| 0-50% consumed | Normal delivery. Review weekly SLO report. |
| 50-75% consumed | Service owner reviews active risks. Risky changes require explicit approval. |
| 75-100% consumed | Freeze non-critical changes. Prioritize reliability fixes and rollback risky releases. |
| 100% consumed | Stop feature rollout until service owner and business owner approve an exception. |

Decision owners:

- Service owner: technical mitigation and rollback.
- Incident commander: live-incident coordination.
- Product/business owner: customer-facing degraded behavior decisions.

