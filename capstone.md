# Capstone 2: Stan's Robot Shop

## The Shop

- About a dozen small services
- Install it with the provided Helm chart
- Remove the Instana agent settings
- Use load-gen for steady traffic

### Focus on Four

**web**, **catalogue**, **cart**, and **payment**

The other services only need to run.

**Data stores:** MongoDB, Redis, MySQL, RabbitMQ

## Where It Runs

- Your Capstone 1 cluster
- Fork `instana/robot-shop` first
- Work only in your own fork

## Six Steps

1. **Deploy and watch** — Install Robot Shop on your Capstone 1 cluster and start load-gen. All pods are healthy.
2. **See the metrics** — Install kube-prometheus-stack with Helm and build one dashboard for requests, errors, and latency.
3. **Collect the logs** — Run Fluent Bit and send logs to Loki. You can search by service and error level.
4. **Follow one trace** — Use the instructor's OpenTelemetry config on one service and find one trace in Jaeger.
5. **Set one SLO** — Pick availability, write a one-page error budget policy, and send one alert to Slack.
6. **Run the incident drill** — The instructor breaks something. Mitigate first, find the root cause, then write a postmortem.

## Done Means

### Hand In

- Dashboard exported as code
- Logging and tracing config
- SLO document with error budget
- Blameless postmortem

### Acceptance Criteria

- The alert fires during the drill
- One request is traced
- Logs are searchable by service and level
- A runbook is linked from the alert

### Optional

- Burn rate alerts
- Trace to log links
- Litmus chaos test
