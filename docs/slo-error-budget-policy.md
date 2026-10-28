# Error Budget Policy — Robot Shop Payment Availability

**Service:** payment (Stan's Robot Shop)  
**Owner:** Capstone 2 on-call (platform)  
**Last updated:** 2026-10-28

## SLO

| Field | Value |
|-------|--------|
| Indicator | Availability — payment pod Ready in namespace `robot-shop` |
| Objective | **99.5%** over a **30-day** rolling window |
| Error budget | **0.5%** ≈ **3.6 hours** unready/unavailable per 30 days |

Measurement uses Kubernetes readiness (`kube_pod_status_ready`). A Ready payment replica counts as successful; missing or unready payment burns budget.

## Why this SLO

Checkout cannot complete without payment. Burning this budget means customers fail at the highest-value step of the shop.

## Alerting

Prometheus alert **`RobotShopPaymentAvailabilitySLOBreached`** fires when payment is unready or absent for **30 seconds**. Alertmanager routes critical alerts to **Slack** with a link to the runbook.

This is a fast page for total loss of payment — treat it as rapid budget burn, not a slow multi-window burn-rate pair.

## Policy when budget is consumed

| Budget remaining | Policy |
|------------------|--------|
| **> 50%** | Normal change velocity. Ship features behind usual review. |
| **25–50%** | Prefer reliability work. Feature deploys need explicit risk review. |
| **< 25%** | **Feature freeze** for payment and its hard dependencies (cart, rabbitmq path) until budget recovers or a waiver is written. |
| **Budget exhausted** | Incident mode only: mitigate, restore availability, then postmortem before new features. |

## Response expectations

1. Acknowledge the Slack alert within **15 minutes** during the drill / on-call window.
2. Follow [RUNBOOK.md](../RUNBOOK.md#robotshoppaymentavailabilityslobreached): mitigate first, then diagnose.
3. After restore, note minutes of downtime against the 3.6h monthly budget.

## Explicit non-goals

- This policy does **not** define latency or correctness SLOs for payment.
- Synthetic load-gen intentional payment errors (`ERROR=1`) do **not** alone mark the pod unready; they are outside this availability SLO.
