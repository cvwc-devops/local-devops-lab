# Loki Logging Stack for Docker Compose + Kubernetes

**One README, two runtimes, fewer false alarms.**

A practical Loki stack for both local development and Kubernetes, packaged in one layout with:

- **Loki** for log aggregation
- **Grafana** for dashboards and alerting
- **Grafana Alloy** for log collection
- **Docker Compose** for local development
- **Helm on Kubernetes** for cluster deployment
- **Quiet-by-default Loki alert tuning** to reduce noise

> This setup uses Docker Compose for development and Helm on Kubernetes for production, which matches Grafana’s recommended deployment approach for Loki. Grafana Alloy is used as the default collector for new deployments, while Promtail should be treated as legacy because it is deprecated and in its LTS/EOL phase. 

---

## Table of contents

- [Overview](#overview)
- [Repository layout](#repository-layout)
- [Architecture](#architecture)
- [Quick start with Docker Compose](#quick-start-with-docker-compose)
- [Docker Compose configuration](#docker-compose-configuration)
- [Kubernetes deployment](#kubernetes-deployment)
- [Label strategy](#label-strategy)
- [Logging format guidance](#logging-format-guidance)
- [Noise-reduction tuning for Loki alerts](#noise-reduction-tuning-for-loki-alerts)
- [Example Loki alert rules](#example-loki-alert-rules)
- [Notification policy defaults](#notification-policy-defaults)
- [Production notes](#production-notes)
- [Tag line](#tag-line)

---

## Overview

This repository gives you one clear path for logging in two places:

- **Docker Compose** for local testing and developer onboarding
- **Kubernetes** for real cluster deployment

The goal is not to make local and production identical. The goal is to make them **compatible**:

- same log labels,
- same dashboards,
- same alerting patterns,
- same troubleshooting workflow.

That way, you avoid the common problem where local logging is easy but useless in production, or production logging is powerful but too heavy for daily work.

A typical Loki stack consists of a collector/agent, Loki as the backend, and Grafana for querying and alerting. Grafana’s current documentation positions Alloy as the current collection path and Helm as the preferred Kubernetes install method. 

---

## Repository layout

```text
repo/
├─ README.md
├─ compose/
│  ├─ docker-compose.yml
│  ├─ loki-config.yaml
│  ├─ alloy/
│  │  └─ config.alloy
│  └─ grafana/
│     ├─ provisioning/
│     │  ├─ datasources/
│     │  │  └─ loki.yaml
│     │  └─ dashboards/
│     └─ dashboards/
├─ k8s/
│  ├─ namespaces.yaml
│  ├─ loki-values.yaml
│  ├─ grafana-values.yaml
│  └─ alloy-values.yaml
├─ alerts/
│  ├─ loki-rules.yaml
│  ├─ recording-rules.yaml
│  └─ notification-policy-notes.md
└─ examples/
   ├─ app-json-logs.md
   ├─ app-plain-logs.md
   └─ troubleshooting.md

---

## Why this layout works
- compose/ holds the full local stack.
- k8s/ holds the production deployment path.
- alerts/ keeps alert logic separate from deployment logic.
- examples/ documents how applications should log.

This separation makes it easier to promote rules, review changes, and reuse dashboards across both environments.

---

## Architecture
### Local development
- applications write logs to stdout
- Alloy discovers Docker containers and forwards logs to Loki
- Grafana queries Loki and evaluates alerts

### Kubernetes
- workloads write logs to stdout and stderr
- Alloy runs as an agent and forwards pod logs to Loki
- Loki runs via Helm
- Grafana queries Loki and manages alerting

### Design goals
- simple local startup
- stable labels across environments
- low-cardinality log streams
- alerts that stay useful after rollout day

---

### Quick start with Docker Compose

#### From the repository root:
- cd compose
- docker compose up -d

Open:
- Grafana: http://localhost:3000
- Loki: http://localhost:3100

Default Grafana login:
- username: admin
- password: admin

Grafana provides official Docker images for local setup, and Loki documents Docker/Compose as appropriate for evaluation, testing, and development rather than production.

### Docker Compose configuration
#### compose/docker-compose.yml
'''
version: "3.9"

services:
  loki:
    image: grafana/loki:3.5.0
    command: -config.file=/etc/loki/config.yaml
    ports:
      - "3100:3100"
    volumes:
      - ./loki-config.yaml:/etc/loki/config.yaml:ro
      - loki-data:/loki

  grafana:
    image: grafana/grafana:12.4.0
    ports:
      - "3000:3000"
    environment:
      GF_SECURITY_ADMIN_USER: admin
      GF_SECURITY_ADMIN_PASSWORD: admin
    volumes:
      - ./grafana/provisioning:/etc/grafana/provisioning:ro
      - ./grafana/dashboards:/var/lib/grafana/dashboards:ro
    depends_on:
      - loki

  alloy:
    image: grafana/alloy:latest
    command: run /etc/alloy/config.alloy
    volumes:
      - ./alloy/config.alloy:/etc/alloy/config.alloy:ro
      - /var/run/docker.sock:/var/run/docker.sock
    depends_on:
      - loki

volumes:
  loki-data:
compose/loki-config.yaml
auth_enabled: false

server:
  http_listen_port: 3100

common:
  path_prefix: /loki
  replication_factor: 1
  storage:
    filesystem:
      chunks_directory: /loki/chunks
      rules_directory: /loki/rules

schema_config:
  configs:
    - from: 2024-01-01
      store: tsdb
      object_store: filesystem
      schema: v13
      index:
        prefix: index_
        period: 24h

ruler:
  storage:
    type: local
    local:
      directory: /loki/rules
  rule_path: /tmp/loki/rules-temp
  alertmanager_url: http://grafana:3000

limits_config:
  reject_old_samples: true
  reject_old_samples_max_age: 168h
  allow_structured_metadata: true

analytics:
  reporting_enabled: false
compose/alloy/config.alloy
discovery.docker "containers" {
  host = "unix:///var/run/docker.sock"
}

discovery.relabel "docker_logs" {
  targets = discovery.docker.containers.targets

  rule {
    source_labels = ["__meta_docker_container_name"]
    target_label  = "container"
  }

  rule {
    source_labels = ["__meta_docker_container_label_com_docker_compose_service"]
    target_label  = "service"
  }

  rule {
    replacement  = "local"
    target_label = "cluster"
  }

  rule {
    replacement  = "dev"
    target_label = "environment"
  }
}

loki.source.docker "default" {
  host       = "unix:///var/run/docker.sock"
  targets    = discovery.relabel.docker_logs.output
  forward_to = [loki.process.default.receiver]
}

loki.process "default" {
  stage.static_labels {
    values = {
      job = "docker"
    }
  }

  forward_to = [loki.write.default.receiver]
}

loki.write "default" {
  endpoint {
    url = "http://loki:3100/loki/api/v1/push"
  }
}
'''

compose/grafana/provisioning/datasources/loki.yaml
'''
apiVersion: 1

datasources:
  - name: Loki
    type: loki
    access: proxy
    url: http://loki:3100
    isDefault: true
    editable: false
'''

Notes
- Pin exact image tags in real environments.
- Keep local Loki simple: single binary, filesystem storage, minimal moving parts.
- Use Alloy for new setups. Promtail remains documented, but it is deprecated and reaches EOL on March 2, 2026.

### Kubernetes deployment
Add Grafana Helm repo
'''
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
Create namespace
kubectl create namespace observability
Install Loki
helm upgrade --install loki grafana/loki \
  --namespace observability \
  -f k8s/loki-values.yaml
Install Grafana
helm upgrade --install grafana grafana/grafana \
  --namespace observability \
  -f k8s/grafana-values.yaml
'''

### Install Alloy
- Deploy Alloy with your Kubernetes values file and cluster-specific discovery settings.
- Grafana recommends the Loki Helm chart for Kubernetes and documents multiple deployment modes, including single binary and simple scalable.

### k8s/loki-values.yaml
'''
deploymentMode: SingleBinary

loki:
  auth_enabled: false

  commonConfig:
    replication_factor: 1

  schemaConfig:
    configs:
      - from: "2024-01-01"
        store: tsdb
        object_store: filesystem
        schema: v13
        index:
          prefix: index_
          period: 24h

singleBinary:
  replicas: 1

monitoring:
  dashboards:
    enabled: true
  rules:
    enabled: true
  serviceMonitor:
    enabled: true
'''

Kubernetes deployment guidance

Use single binary when you need:

a smaller internal cluster setup,

simpler operations,

lower resource overhead.

Move to simple scalable when you need:

clearer separation of read and write paths,

better scaling behavior,

more operational flexibility.

Grafana documents simple scalable mode as the path that separates read, write, and backend targets for larger production deployments.

Label strategy

A quiet Loki setup starts with label discipline.

Good labels

Use labels that are stable and low-cardinality:
- cluster
- environment
- namespace
- service
- app
- container
- level

Bad labels

Do not turn these into labels:
- request ID
- trace ID
- session ID
- user ID

raw URL with IDs

pod UID

exception text

Rule of thumb

If a value changes constantly, it should stay in the log line or parsed fields, not in the Loki label set.

Loki indexes labels rather than log bodies, so stable labels are essential for predictable ingestion and query performance.

Logging format guidance

Prefer structured JSON logs.

Example:
'''
{
  "ts": "2026-03-03T10:15:20Z",
  "level": "error",
  "service": "checkout-api",
  "message": "payment provider timeout",
  "route": "/api/orders/{id}/pay",
  "status": 504,
  "tenant": "internal"
}
'''

Why JSON logs help

easier parsing,

better filters,

cleaner alerts,

more reusable dashboards.

Loki metric queries such as rate, count_over_time, and absent_over_time become much more useful when the logs are structured enough to filter consistently.

Noise-reduction tuning for Loki alerts

The goal is not “more alerts.” The goal is fewer, better alerts.

### Best-default alert posture
## 1. Alert on rates and windows, not single log lines

Bad:
- one error line = one alert

Better:
- elevated error rate for 10 minutes

Use rate() or count_over_time() over a time range instead of raw matches.

## 2. Add a for: duration to nearly everything

Recommended defaults:
- warning: for: 10m
- critical: for: 5m to 10m
- absence: for: 5m on top of a larger absence window

This filters short spikes and rollout turbulence.

## 3. Use recovery thresholds for Grafana-managed alerts

Example:
- fire at > 5%
- recover at < 2%

Grafana documents recovery thresholds as a way to reduce alert flapping.

## 4. Treat “No Data” as a separate class of signal

Recommended defaults:
- do not page on generic datasource no-data by default
- use explicit absent_over_time() when missing logs are the real signal
- route DatasourceNoData and DatasourceError separately

Grafana documents these as special alert instances that can fire immediately, which makes them a common source of noise if not routed carefully.

## 5. Use recording rules for repeated expensive queries

If a query appears in multiple dashboards and alerts, precompute it.

Recording rules help reduce query cost and make alerts easier to reason about.

## 6. Keep alert dimensions small

### Good:
- one alert per service
- maybe per namespace + service

### Bad:
- one alert per pod
- one alert per container ID
- one alert per request path

## 7. Group notifications

### Recommended grouping:
- alertname
- cluster
- environment
- service

### Recommended defaults:
- group wait: 30s
- group interval: 5m
- repeat interval:
- warning: 4h
- critical: 1h

## 8. Separate warning from critical
- warnings go to chat/email
- critical alerts go to paging
- datasource and observability alerts go to platform or observability owners first

## Example Loki alert rules
### alerts/recording-rules.yaml
'''
groups:
  - name: loki-recording-rules
    interval: 1m
    rules:
      - record: service:log_errors_per_second:rate5m
        expr: |
          sum by (cluster, environment, namespace, service) (
            rate({job="docker"} | json | level="error"[5m])
          )

      - record: service:log_lines_per_second:rate5m
        expr: |
          sum by (cluster, environment, namespace, service) (
            rate({job="docker"}[5m])
          )
'''

### alerts/loki-rules.yaml
'''
groups:
  - name: loki-app-alerts
    interval: 1m
    rules:
      - alert: LokiHighErrorLogRate
        expr: |
          sum by (cluster, environment, namespace, service) (
            rate({job="docker"} | json | level="error"[5m])
          ) > 0.05
        for: 10m
        labels:
          severity: warning
          team: platform
        annotations:
          summary: "High error log rate for {{ $labels.service }}"
          description: "Error log rate has stayed above 0.05 lines/sec for 10m in {{ $labels.namespace }}."

      - alert: LokiHighErrorRatio
        expr: |
          (
            service:log_errors_per_second:rate5m
            /
            service:log_lines_per_second:rate5m
          ) > 0.05
        for: 10m
        labels:
          severity: critical
          team: platform
        annotations:
          summary: "High error ratio for {{ $labels.service }}"
          description: "More than 5% of log lines are errors for 10m."

      - alert: LokiServiceMissingLogs
        expr: |
          absent_over_time({namespace="prod", service="checkout-api"}[15m])
        for: 5m
        labels:
          severity: critical
          team: platform
        annotations:
          summary: "No logs from checkout-api"
          description: "No logs seen for checkout-api for 15m."

      - alert: LokiAuthFailuresBurst
        expr: |
          sum by (cluster, environment, namespace, service) (
            count_over_time({namespace="prod"} |= "authentication failed"[10m])
          ) > 50
        for: 10m
        labels:
          severity: warning
          team: security
        annotations:
          summary: "Authentication failures increased for {{ $labels.service }}"
          description: "More than 50 auth failure log lines in 10m."

      - alert: LokiCollectorPipelineErrors
        expr: |
          sum by (cluster, environment) (
            count_over_time({service="alloy"} |= "error"[10m])
          ) > 20
        for: 10m
        labels:
          severity: warning
          team: platform
        annotations:
          summary: "Alloy pipeline errors detected"
          description: "Collector errors are recurring and may affect log ingestion."
'''

### Why these examples are quieter
- they use 5–15 minute windows,
- they use for: delays,
- they group by service instead of pod,
- they alert on sustained symptoms,
- they separate app failures from pipeline failures.

Loki supports rate, count_over_time, and absent_over_time for this exact style of log-based alerting.

### Best-default values
- Use these values first, then tune by service.

### Rule evaluation
- every 1m

### Range windows
- rate alerts: 5m
- burst/count alerts: 10m to 15m
- absence alerts: 15m or more

Grafana’s Loki guidance on metric-query windows emphasizes that time range choice materially affects both performance and the usefulness of the resulting signal.

### Pending duration
- warning: 10m
- critical: 5m to 10m
- absence: 5m after a 15m absence query

### Notification grouping
- group by: alertname, cluster, environment, service
- group wait: 30s
- group interval: 5m
- repeat interval:
- warning: 4h
- critical: 1h

### Error-ratio thresholds

#### Recommended default:
- warning at > 2%
- critical at > 5%
- recover below 2% for critical rules where recovery thresholds are available

### No-data handling

Recommended default:
- do not page on generic no-data
- use explicit absence rules for truly critical services
- route datasource-generated alerts separately

## Notification policy defaults
- Use a policy like this:

### Warning channel

Send:
- severity=warning
- severity=info
- datasource no-data
- datasource error

Destination:
- Slack
- Teams
- email

### Critical channel

Send:

severity=critical

Destination:
- PagerDuty
- Opsgenie
- phone paging

### Grouping keys
- Group by:
- alertname
- cluster
- environment
- service

Do not group by:
- pod
- instance
- container ID
- request path

This keeps notifications aligned with things a team can actually own and fix.

Compose and Kubernetes parity rules

To keep local and production aligned, keep the same label model in both places.

Shared labels
- cluster
- environment
- namespace
- service
- container
- level

In Compose

Synthesize:
- cluster=local
- environment=dev

In Kubernetes

Derive:

cluster from cluster metadata
- environment from cluster or namespace conventions
- namespace, pod, container from Kubernetes discovery
- service from application labels

If the labels stay consistent, most dashboards and many alerts can move between local and cluster environments without being rewritten.

