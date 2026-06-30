# OpenTelemetry Demo Stack

## Purpose

This stack is for end-to-end testing of metrics:

1. Metric production (application or collector-generated metrics)
2. Metric collection and processing (OpenTelemetry Collector)
3. Metric storage and query (Prometheus)
4. Metric visualization (Grafana)

It also includes Jaeger for trace visualization and correlation.

## Components

- otelcol: receives OTLP data, generates span metrics, can scrape Prometheus endpoints, exports metrics for Prometheus scraping, and forwards traces to Jaeger (opentelemetry-collector-contrib 0.155.0)
- prometheus: scrapes metrics endpoints and stores time series (v3.12.0)
- grafana: visualizes data from Prometheus (grafana/grafana 13.0.3)
- jaeger: trace backend and UI (Jaeger v2, jaegertracing/jaeger 2.19.0)

## Data Flow

1. Your app sends telemetry to otelcol (OTLP gRPC 4317 or HTTP 4318), or exposes Prometheus metrics.
2. otelcol processes telemetry, exposes Prometheus-format metrics on port 9464, and forwards traces to jaeger over OTLP (jaeger:4317).
3. prometheus scrapes otelcol endpoints (9464 and 8888 by default).
4. grafana queries prometheus at http://prometheus:9090.
5. jaeger stores traces (in-memory) and reads span metrics from prometheus for its Monitor (SPM) tab.

## Files And What To Configure

### docker-compose.yaml

Defines containers, ports, and mounted config files.

Change this when:
- adding/removing services
- exposing additional host ports
- changing mounted config locations

Compose reference file:
- docker-compose.yaml

## Docker Compose Service Reference

This section maps the compose reference file and each service to its effective configuration.

### Compose Reference File

- source file: docker-compose.yaml
- service definitions: grafana, otelcol, prometheus, jaeger
- shared network name: opentelemetry-demo

### grafana

- defined in: docker-compose.yaml (service: grafana)
- mounted config file: grafana-datasource.yaml
- mount target in container: /etc/grafana/provisioning/datasources/defaults.yaml
- edit when changing datasource URL, default datasource, or provisioning behavior

### otelcol

- defined in: docker-compose.yaml (service: otelcol)
- mounted config files:
  - otelcol-config.yaml
  - otelcol-observability.yaml
- mount targets in container:
  - /etc/otelcol-config.yaml
  - /etc/otelcol-observability.yaml
- compose command loads both configs:
  - --config=/etc/otelcol-config.yaml
  - --config=/etc/otelcol-observability.yaml
- edit when changing receivers, processors, exporters, or pipeline wiring

### prometheus

- defined in: docker-compose.yaml (service: prometheus)
- mounted config file: prometheus-config.yaml
- mount target in container: /etc/prometheus/prometheus-config.yaml
- compose command flag using this file: --config.file=/etc/prometheus/prometheus-config.yaml
- edit when changing scrape jobs, targets, intervals, and retention-related behavior

### jaeger

- defined in: docker-compose.yaml (service: jaeger)
- mounted config file: jaeger-config.yaml
- mount target in container: /etc/jaeger/config.yaml
- compose command flag using this file: --config=/etc/jaeger/config.yaml
- Jaeger v2 is built on the OpenTelemetry Collector framework, so behavior is configured
  via this config file (extensions/receivers/exporters) instead of the v1 command flags and
  environment variables (COLLECTOR_OTLP_ENABLED, METRICS_STORAGE_TYPE, --prometheus.server-url)
- edit when changing trace storage limits, the OTLP receiver, or the Prometheus (SPM) backend URL

### otelcol-config.yaml

Main collector pipelines:
- receivers
- processors
- exporters
- service pipelines

Change this when:
- adding telemetry inputs (for example a Prometheus scrape receiver)
- changing filtering/batching behavior
- changing what pipelines receive or export

Current host scrape example:
- receiver: prometheus
- target: host.docker.internal:9321
- path: /metrics

Important:
- use host.docker.internal for host-local services from containers on macOS
- do not use localhost for host services from inside containers

### otelcol-observability.yaml

Collector observability exporters.

Change this when:
- changing where collector metrics are exposed
- adjusting Prometheus exporter settings

Current metric exposure endpoint:
- otelcol:9464

### prometheus-config.yaml

Prometheus scrape jobs (metric sources for Prometheus).

Change this when:
- adding/removing scrape targets
- tuning scrape/evaluation intervals
- setting new job labels

Default targets:
- otelcol:9464 (collector exported metrics)
- otelcol:8888 (collector internal metrics)

### grafana-datasource.yaml

Grafana datasource provisioning.

Change this when:
- switching datasource URL
- adding more datasources

Default datasource:
- Prometheus at http://prometheus:9090

## Common Change: Add A Local App Metrics Endpoint

If an app on your host exposes metrics at localhost:9321/metrics:

1. Add a Prometheus receiver in otelcol-config.yaml
2. Set target to host.docker.internal:9321
3. Include prometheus in metrics pipeline receivers
4. Recreate the collector container

Command:

docker compose up -d --force-recreate otelcol

## Quick Verification

- Prometheus targets page: http://localhost:9090/targets
- Grafana UI: http://localhost:3000
- Jaeger UI: http://localhost:16686 (Jaeger v2 serves the UI at the root path)

## Grafana Access

Grafana is available at:
- http://localhost:3000

Default admin credentials:
- username: admin
- password: admin

The Prometheus datasource is provisioned from grafana-datasource.yaml and points to:
- http://prometheus:9090

After login, you can verify datasource connectivity at:
- Home -> Connections -> Data sources -> Prometheus -> Save & test

If a target is down:
- verify endpoint reachable from container network
- verify target host/port/path
- check otelcol and prometheus logs

## Copy/Paste Snippets

### 1) One Host-Local App

Use this in otelcol-config.yaml under receivers:

```yaml
prometheus:
  config:
    scrape_configs:
      - job_name: host-app
        scrape_interval: 5s
        metrics_path: /metrics
        static_configs:
          - targets: ["host.docker.internal:9321"]
            labels:
              service: host-app
              env: local
```

Then include prometheus in the metrics pipeline receivers:

```yaml
service:
  pipelines:
    metrics:
      receivers: [otlp, spanmetrics, prometheus]
```

### 2) Multiple Host-Local Apps

```yaml
prometheus:
  config:
    scrape_configs:
      - job_name: host-apps
        scrape_interval: 5s
        metrics_path: /metrics
        static_configs:
          - targets: ["host.docker.internal:9321"]
            labels:
              service: api
              env: local
          - targets: ["host.docker.internal:9322"]
            labels:
              service: worker
              env: local
          - targets: ["host.docker.internal:9323"]
            labels:
              service: scheduler
              env: local
```

### 3) Label Conventions

Recommended labels for easier Grafana filtering:

- service: stable logical service name (api, worker, billing)
- env: local, dev, staging, prod
- team: optional ownership tag
- region: optional location tag if relevant

Example:

```yaml
labels:
  service: api
  env: local
  team: platform
```

### 4) Restart Command

After changing collector config:

```bash
docker compose up -d --force-recreate otelcol
```
