# Blameless Postmortem — Payment Availability Drill

**Incident:** Capstone 2 Step 6 SRE failure drill  
**Service:** payment (`robot-shop`)  
**SLO:** 99.5% availability (Ready) — see [slo-error-budget-policy.md](slo-error-budget-policy.md)  
**Alert:** `RobotShopPaymentAvailabilitySLOBreached`  
**Date:** 2026-10-28  
**Severity:** critical (total loss of payment)  
**Status:** Resolved  

## Summary

During the Capstone incident drill, payment was scaled to zero replicas. Checkout could not complete while the deployment had no Ready pods. Prometheus fired the payment availability SLO alert; Alertmanager notified Slack via `slack-critical` with a runbook link. On-call mitigated by restoring `replicas=1` per [RUNBOOK.md](../RUNBOOK.md). Service returned to Ready in **123 seconds** of downtime.

This was an intentional fault injection to exercise detection, paging, mitigation, and error-budget accounting — not a production customer incident.

## Impact

| Area | Detail |
|------|--------|
| User impact | Payment unavailable; checkout fails for the duration of the outage |
| Scope | `deployment/payment` in namespace `robot-shop` (replicas 0) |
| Duration | **123 s** (2.05 min) wall-clock from inject to Ready restore |
| SLO / budget | Burns the 99.5% availability error budget (~**3.6 h / 30 days**) |

Load-gen continued against the shop; payment-dependent paths failed until restore.

## Timeline (UTC, measured)

| Time (UTC) | Event |
|------------|--------|
| 14:10:00 | Baseline: `payment` Ready `1/1`. Slack notification counter = **2**. |
| 14:10:12 | **Fault inject:** `kubectl -n robot-shop scale deploy/payment --replicas=0`. Deployment Ready `0/0`. |
| 14:11:33 | Prometheus rule enters **pending** (`activeAt` = 2026-10-28T14:11:33.433Z). |
| 14:12:03 | Alert **firing**. Alertmanager alert `startsAt` = 14:12:03.433Z; receiver **`slack-critical`**. Runbook URL uses branch **`master`**. |
| ~14:12:46 | Slack notification success: counter **2 → 3** (`alertmanager_notifications_failed_total` for slack = **0**). |
| 14:12:08 | **Mitigate start:** scale `payment` `--replicas=1` per runbook. |
| 14:12:15 | **Restored:** rollout complete; pod Ready `1/1` (`payment-64cfb4c6d8-qqhq8`). |
| ~14:13:25 | Slack resolve notification: counter **3 → 4** (failures still **0**). Prometheus no longer lists the alert as active. |

### Response metrics (this drill)

| Metric | Value |
|--------|--------|
| Time to pending (detect) | ~81 s after inject |
| Time to Alertmanager page | ~111 s after inject |
| Time from AM page to mitigate start | ~5 s |
| MTTR (inject → Ready) | **123 s** |

## Detection

- **Signal:** PrometheusRule `robot-shop-slo-alerts` — expression on missing/unready `kube_pod_status_ready` for `payment-*` in `robot-shop`, `for: 30s`.
- **Routing:** severity `critical` → Alertmanager receiver `slack-critical` (`api_url_file` secret `alertmanager-slack`).
- **Human channel:** Slack; annotation `runbook_url` →  
  `https://github.com/ShemaiahYaba/robot-shop/blob/master/RUNBOOK.md#robotshoppaymentavailabilityslobreached`

## Mitigation

Followed runbook “Mitigate first”:

1. Confirmed `deploy/payment` Ready `0/0`.
2. `kubectl -n robot-shop scale deploy/payment --replicas=1`
3. `kubectl -n robot-shop rollout status deploy/payment` — success within ~7 s
4. Verified pod Ready `1/1`

No node drain, image change, or dependency fix was required for this fault mode.

## Root cause

**Direct cause:** Deployment replica count set to **0** (fault injection for Capstone Step 6).

**Contributing factors (systems, not blame):**

- No PodDisruptionBudget or admission policy preventing `replicas=0` on a critical checkout dependency.
- Availability SLO correctly treats “no Ready payment pod” as total failure — expected and desirable for this page.
- Earlier Step 5 fire-test used the same inject path; Slack/runbook path was already proven, so MTTR was short once the page landed.

## Error budget consumed

| Field | Value |
|-------|--------|
| Downtime | **123 s ≈ 2.05 min** |
| Monthly budget (0.5% of 30 d) | **3.6 h = 216 min** |
| Budget consumed this incident | **2.05 / 216 ≈ 0.95%** of monthly budget |
| Policy band after drill | Still **> 50%** remaining → normal change velocity per policy |

Prior Step 5 verification outage (same day) also burned a small amount of budget; treat drill minutes as intentional reliability spend, not surprise customer loss.

## What went well

- Alert fired and routed to Slack with **zero** Slack notification failures.
- Runbook steps matched the actual fault; restore was immediate once scaled back.
- Runbook URL on the page used **`master`** (repo default branch), so the linked doc resolves on GitHub.
- Clear SLO policy made budget math straightforward.

## What to improve

- Guard critical services against accidental `scale --replicas=0` (e.g. document min replicas, or a policy/check in CI/CD).
- Consider a complementary alert on `kube_deployment_spec_replicas{deployment="payment"} == 0` for faster “someone scaled us down” signal (still keep readiness-based SLO).
- Shipping deployment was observed Ready **`0/1`** during the drill window — **out of scope** for this payment incident, but should be tracked as a separate reliability item so the shop is fully healthy before the next drill.

## Action items

| Owner | Action | Priority |
|-------|--------|----------|
| Capstone / platform | Keep `runbook_url` on `blob/master/...` in `observability/alerts/robot-shop-slo-rules.yaml` committed to the fork | P1 |
| Capstone / platform | Document “never scale payment to 0 outside drills” in RUNBOOK close-out | P2 |
| Capstone / platform | Investigate `shipping` Ready `0/1` separately | P2 |
| Capstone / platform | Optional: alert or check for `spec.replicas == 0` on payment | P3 |

## References

- [Error budget policy](slo-error-budget-policy.md)
- [RUNBOOK — RobotShopPaymentAvailabilitySLOBreached](../RUNBOOK.md#robotshoppaymentavailabilityslobreached)
- PrometheusRule: `observability/alerts/robot-shop-slo-rules.yaml`
