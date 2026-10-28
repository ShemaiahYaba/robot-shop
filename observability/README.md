# Capstone 2 — Metrics (Step 2)

## Stack

- `kube-prometheus-stack` in namespace `monitoring` (Prometheus, Grafana, Alertmanager)
- `prometheus-blackbox-exporter` — HTTP probes for web / catalogue / cart / payment
- `nginx-exporter` — scrapes `web` `/nginx_status` for request rate
- ServiceMonitors for `cart` and `payment` `/metrics`

## Dashboard (as code)

[`grafana/robot-shop-red.json`](grafana/robot-shop-red.json) — requests, errors, latency (RED).

Provisioned via ConfigMap `robot-shop-red-dashboard` (`grafana_dashboard=1`).

## Access Grafana

```bash
kubectl -n monitoring port-forward svc/kube-prometheus-stack-grafana 3000:80
```

Login: `admin` / password from Helm values (`prom-operator` unless changed).

Open dashboard **Robot Shop RED**.

## Reinstall / upgrade

```bash
helm upgrade --install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  -n monitoring -f kube-prometheus-stack-values.yaml

helm upgrade --install prometheus-blackbox-exporter prometheus-community/prometheus-blackbox-exporter \
  -n monitoring -f blackbox/values.yaml

kubectl apply -f nginx-exporter/deployment.yaml
kubectl apply -f servicemonitors/
kubectl apply -f blackbox/probes.yaml
```
