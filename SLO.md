# Robot Shop SLO

## Service Level Objectives

### Availability SLO
- **SLI**: Successful HTTP requests (2xx) / Total HTTP requests
- **SLO**: 99.9% availability per month
- **Error Budget**: 0.1% per month = ~43.8 minutes downtime/month
- **Measurement window**: 30-day rolling

### Latency SLO
- **SLI**: HTTP requests completed in < 500ms / Total HTTP requests
- **SLO**: 95% of requests complete in < 500ms
- **Error Budget**: 5% of requests can exceed 500ms
- **Measurement window**: 5-minute rolling window

### Error Rate SLO
- **SLI**: Non-5xx responses / Total responses
- **SLO**: Error rate < 1%
- **Error Budget**: 1% of requests per 5-minute window

---

## Error Budget Math

### Monthly error budget (availability):
```
Total minutes/month = 30 * 24 * 60 = 43,200 minutes
Error budget = 43,200 * 0.001 = 43.2 minutes
```

### If we burn 10 min of downtime:
```
Budget remaining = 43.2 - 10 = 33.2 minutes (76.9% remaining)
```

---

## PromQL Tracking Queries

```promql
# Availability over last 5 minutes
sum(rate(http_requests_total{namespace="robot-shop",status=~"2.."}[5m]))
/
sum(rate(http_requests_total{namespace="robot-shop"}[5m]))

# p95 latency
histogram_quantile(0.95,
  rate(http_request_duration_seconds_bucket{namespace="robot-shop"}[5m])
)

# Error rate
sum(rate(http_requests_total{namespace="robot-shop",status=~"5.."}[5m]))
/
sum(rate(http_requests_total{namespace="robot-shop"}[5m]))

# Error budget remaining (monthly)
1 - (
  sum(increase(http_requests_total{namespace="robot-shop",status=~"5.."}[30d]))
  /
  sum(increase(http_requests_total{namespace="robot-shop"}[30d]))
) / 0.001
```

---

## Alerting Thresholds

| Alert | Condition | Severity | Action |
|-------|-----------|----------|--------|
| HighErrorRate | error_rate > 1% for 5m | critical | Page on-call |
| HighLatency | p95 > 500ms for 5m | warning | Investigate |
| RobotShopPodDown | pod not ready for 1m | critical | Page on-call |
| HighMemoryUsage | memory > 80% limit for 5m | warning | Monitor |
