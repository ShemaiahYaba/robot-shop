# Robot Shop Runbooks

## RobotShopPaymentAvailabilitySLOBreached

**Alert:** Payment service is unready or missing in `robot-shop` (99.5% availability SLO / error budget burning).  
**Severity:** critical  
**SLO policy:** [docs/slo-error-budget-policy.md](docs/slo-error-budget-policy.md)

### Mitigate first

1. Confirm scope:
   ```bash
   kubectl -n robot-shop get pods -l service=payment -o wide
   kubectl -n robot-shop get deploy payment
   ```
2. If replicas are `0`, restore immediately:
   ```bash
   kubectl -n robot-shop scale deploy/payment --replicas=1
   ```
3. If the pod is CrashLoop / ImagePull / Pending, restart or reschedule:
   ```bash
   kubectl -n robot-shop rollout restart deploy/payment
   kubectl -n robot-shop rollout status deploy/payment --timeout=120s
   ```
4. Verify Ready:
   ```bash
   kubectl -n robot-shop get pods -l service=payment
   curl -sf http://payment.robot-shop.svc:8080/health || true
   ```

### Diagnose (after traffic is restored or while waiting)

```bash
kubectl -n robot-shop describe pod -l service=payment
kubectl -n robot-shop logs -l service=payment --tail=100
kubectl -n robot-shop get events --field-selector involvedObject.name=$(kubectl -n robot-shop get pod -l service=payment -o jsonpath='{.items[0].metadata.name}') --sort-by='.lastTimestamp'
```

Common causes: scaled to zero, OOM, bad image, node NotReady / Too many pods, dependency outage (RabbitMQ) if the process exits.

### Close out

1. Confirm Slack alert resolves (or silence after fix).
2. Record downtime minutes against the monthly **3.6h** error budget.
3. For Capstone drill: write a blameless postmortem (timeline, impact, contributing factors, action items).
