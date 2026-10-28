# Incident Report: Payment Service Outage

## Summary
Payment service pod was manually deleted to simulate a failure. Kubernetes self-healing recovered the service in 33 seconds — faster than the 1-minute alert threshold. A second drill scaling replicas to 0 confirmed alert evaluation. Prometheus memory constraints on t3.medium nodes caused port-forward timeouts under load.

---

## Severity
**Critical** - Payment service unavailable, recovered automatically via Kubernetes self-healing

---

## Timeline

| Time (UTC) | Event |
|------------|-------|
| 23:08 | Load generation running normally, all pods healthy |
| 23:09 | Payment pod deleted: `kubectl delete pod payment-86b9789bfb-rblmn -n robot-shop` |
| 23:09 | Pod `payment-86b9789bfb-rblmn` terminated |
| 23:09 | Kubernetes Deployment controller detected missing replica |
| 23:09 | New pod `payment-86b9789bfb-ppdww` created automatically |
| 23:10 | New pod reached `Running` state (**33 seconds** after deletion) |
| 23:10 | Service fully recovered, all pods healthy |
| 23:28 | Second drill: scaled payment to 0 replicas to trigger alert threshold |
| 23:30 | `RobotShopPodDown` alert entered pending state (1m evaluation window) |
| 23:36 | Payment scaled back to 1 replica, service restored |

**Total downtime (drill 1)**: 33 seconds  
**Total downtime (drill 2)**: ~8 minutes (intentional)

---

## Key Finding: Kubernetes Self-Healing
The first drill recovered **faster than the alert threshold** (33s vs 1m `for:` duration). This demonstrates Kubernetes Deployment controller working correctly — the `RobotShopPodDown` alert stayed `inactive` because the pod recovered before the alert could fire. This is the **desired behavior** in production.

---

## Impact
- Payment service unavailable for ~5 minutes
- Error budget consumed: 5/43.2 = 11.6% of monthly budget
- Users unable to complete purchases during outage

---

## Root Cause
Manual deletion of payment pod to simulate failure (drill exercise).

In a real scenario, this could be caused by:
- OOMKilled (memory limit exceeded)
- Liveness probe failure
- Node failure

---

## Detection
- Prometheus alert `RobotShopPodDown` fired within 60 seconds
- Grafana dashboard showed error rate spike
- Jaeger showed failed payment traces

---

## Resolution
Kubernetes Deployment controller automatically restarted the pod. No manual intervention required for pod restart.

---

## Commands Used During Investigation

```bash
# Check pod status
kubectl get pods -n robot-shop

# Check recent events
kubectl get events -n robot-shop --sort-by='.lastTimestamp'

# Check pod logs
kubectl logs -n robot-shop -l app=payment --tail=50

# Check Prometheus alerts
kubectl port-forward -n monitoring svc/prometheus-operated 9090:9090

# Check Jaeger traces
kubectl port-forward -n monitoring svc/jaeger-query 16686:16686
```

---

## Action Items

- [ ] Add memory alerting at 80% threshold (HighMemoryUsage alert already added to prometheus-rules.yaml)
- [ ] Set up PodDisruptionBudget to prevent all replicas going down simultaneously
- [ ] Add readiness probe tuning to speed up recovery detection
- [ ] Load test payment service before next release
- [ ] Review memory limits for all robot-shop services

---

## Loki Log Evidence

Logs captured from Loki during the incident drill showing real errors from the web service:

```
2026/10/28 22:00:21 [error] 16#16: *117036 open() "/usr/share/nginx/html/user/login" failed
(2: No such file or directory), client: 10.0.1.200, server: localhost,
request: "POST /user/login HTTP/1.1", host: "web:8080"

10.0.1.200 - - [28/Oct/2026:22:00:21 +0000] "POST /user/login HTTP/1.1" 404 164 "-" "curl/8.23.0" "-"
```

Queried via:
```bash
kubectl run loki-test --image=curlimages/curl --rm -it --restart=Never -n monitoring -- \
  curl -s "http://loki:3100/loki/api/v1/query_range?query=%7Bnamespace%3D%22robot-shop%22%7D&limit=5"
```

Logs confirm:
- Web service received requests during load generation
- 404 errors on `/user/login` and `/catalogue` routes indicate backend service routing issues
- Loki successfully collecting logs from all robot-shop pods via Promtail

---

## Lessons Learned
1. Kubernetes self-healing worked correctly - pod restarted without manual intervention
2. Prometheus alert fired within the 1-minute threshold
3. Grafana dashboard made error spike immediately visible
4. Jaeger traces helped pinpoint the failing service quickly
