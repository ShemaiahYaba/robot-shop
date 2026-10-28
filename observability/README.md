# Capstone 2 — Observability

## Stack

- `kube-prometheus-stack` in namespace `monitoring` (Prometheus, Grafana, Alertmanager)
- `prometheus-blackbox-exporter` — HTTP probes for web / catalogue / cart / payment
- `nginx-exporter` — scrapes `web` `/nginx_status` for request rate
- ServiceMonitors for `cart` and `payment` `/metrics`
- **Loki** (SingleBinary) + **Fluent Bit** DaemonSet — Robot Shop logs searchable by `service` and level
- **Jaeger** + **OpenTelemetry Collector** — synthetic checkout trace (web → catalogue → cart → payment)

## Dashboard (as code)

[`grafana/robot-shop-red.json`](grafana/robot-shop-red.json) — requests, errors, latency (RED).

Provisioned via ConfigMap `robot-shop-red-dashboard` (`grafana_dashboard=1`).

## Access Grafana

```bash
kubectl -n monitoring port-forward svc/kube-prometheus-stack-grafana 3000:80
```

Login: `admin` / password from Helm values (`prom-operator` unless changed).

Open dashboard **Robot Shop RED**.

### Logs (Explore → Loki)

```logql
{service="cart"}
{service="payment"} |= "ERROR"
{service="cart"} | json | level=~"(?i)error|warn|info"
{service=~"web|catalogue|cart|payment"}
```

## Reinstall / upgrade

```bash
helm upgrade --install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  -n monitoring -f kube-prometheus-stack-values.yaml

helm upgrade --install prometheus-blackbox-exporter prometheus-community/prometheus-blackbox-exporter \
  -n monitoring -f blackbox/values.yaml

kubectl apply -f nginx-exporter/deployment.yaml
kubectl apply -f servicemonitors/
kubectl apply -f blackbox/probes.yaml

helm upgrade --install loki grafana/loki -n monitoring -f logging/loki-values.yaml
helm upgrade --install fluent-bit fluent/fluent-bit -n monitoring -f logging/fluent-bit-values.yaml
kubectl apply -f logging/loki-datasource.yaml

# Tracing (Step 4)
kubectl apply -f tracing/jaeger.yaml
kubectl apply -f tracing/otel-collector.yaml
kubectl apply -f tracing/trace-workflow-configmap.yaml
kubectl -n monitoring delete job trace-verification-job --ignore-not-found
kubectl apply -f tracing/trace-job.yaml
kubectl -n monitoring logs job/trace-verification-job -f
kubectl -n monitoring get svc jaeger
```

### Traces (Jaeger)

Open the Jaeger LoadBalancer URL (port **16686**), search service **web**, or paste the Trace ID printed by the Job.
