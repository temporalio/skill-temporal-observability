---
name: temporal-observability
description: 'Instrument, collect, query, and alert on Temporal metrics across Temporal Cloud and self-hosted deployments. Covers the Cloud OpenMetrics endpoint (metrics.temporal.io), temporal_cloud_v1_* metric catalog, API key authentication, scrape configuration, and rate limits; third-party integrations with Datadog, Grafana Cloud, ClickStack, New Relic, Prometheus, and OpenTelemetry Collector; SDK metrics configuration, per-SDK scrape endpoints, temporal_workflow_* and temporal_activity_* metrics; self-hosted Temporal Service monitoring with Prometheus and Grafana dashboards; Cloud service health monitoring (availability, error rates, failure detection, replication lag, resource exhaustion); Worker fleet health monitoring (backlog, sync match rate, poll success rate, task slots, sticky cache); migration from deprecated PromQL v0 endpoint to OpenMetrics v1; and performance bottleneck troubleshooting via SDK metrics. Use when the user mentions: "Temporal metrics", "set up Datadog for Temporal", "Grafana Temporal dashboard", "Cloud metrics endpoint", "temporal_cloud_v1", "SDK metrics", "worker backlog", "approximate_backlog_count", "scrape Temporal metrics", "OpenMetrics endpoint", "PromQL migration", "v0 to v1", "rate() on Cloud metrics", "metrics authentication", "temporal_workflow_task_schedule_to_start_latency", "Temporal monitoring", "Temporal observability", "Temporal alerting", "metrics.temporal.io", "OpenMetrics v1", "temporal_cloud_v1_service_request_count", "worker fleet health", "sync match rate", "poll success rate", "task slots", "sticky cache metrics", "replication lag metrics", "Temporal Prometheus", "Temporal Grafana", "OTel Collector Temporal", "New Relic Temporal", "ClickStack Temporal", "SDK scrape endpoint", "temporal_activity_schedule_to_start_latency", "temporal_workflow_completed", "service health metrics". Does NOT cover writing SDK application code such as Workflows, Activities, or Workers (temporal-developer), Worker tuning parameters like slot suppliers, tuners, or cache sizing (temporal-workertuning), Cloud or self-hosted administration and diagnosis (temporal-ops), or Cloud connectivity, auth, and namespace configuration (temporal-cloud).'
version: 0.1.0
---

# Skill: temporal-observability

## Overview

Expert guidance for Temporal metrics collection, monitoring, and observability. Helps platform engineers and SREs instrument, collect, query, and alert on Temporal metrics across Temporal Cloud (OpenMetrics endpoint), self-hosted deployments (Prometheus + Grafana), and SDK-emitted metrics, with integrations for Datadog, Grafana Cloud, ClickStack, New Relic, and OpenTelemetry Collector.

## Intent Decision Table

| User intent | Action |
|---|---|
| Wants to know which Cloud metrics exist, their types, or labels | Consult `references/cloud-metrics-catalog.md` for the complete `temporal_cloud_v1_*` catalog |
| Needs to set up or configure the OpenMetrics endpoint (auth, scrape config, rate limits) | Consult `references/openmetrics-endpoint.md` |
| Wants to integrate metrics with Datadog, Grafana Cloud, ClickStack, New Relic, Prometheus, or OTel Collector | Consult `references/integrations.md` |
| Needs to configure SDK metrics scrape endpoints or understand SDK metric units | Consult `references/sdk-metrics.md` |
| Setting up monitoring for a self-hosted Temporal Service | Consult `references/self-hosted-monitoring.md` |
| Wants to monitor Temporal Cloud service health (latency, error rates, failures, limits) | Consult `references/service-health-monitoring.md` |
| Needs to monitor Worker fleet health (backlog, greedy workers, misconfigured workers, sticky cache) | Consult `references/worker-health-monitoring.md` |
| Migrating from PromQL/v0 to OpenMetrics/v1 | Consult `references/migration-v0-to-v1.md` |
| Troubleshooting performance bottlenecks using metrics | Consult `references/troubleshooting.md` |
| Wants to write Workflow/Activity/Worker SDK code (not metrics config) | Defer to `skill-temporal-developer` |
| Wants to tune Worker performance (slot suppliers, tuners, cache sizing) | Defer to `skill-temporal-workertuning` -- this skill owns collection/query layer only |
| Wants to administer or diagnose Temporal Cloud/self-hosted environments | Defer to `skill-temporal-ops` -- this skill provides metric queries for diagnosis |
| Wants Cloud connectivity, auth, or namespace config (not metrics) | Defer to `skill-temporal-cloud` |

## Critical Rules

### v0 vs v1 Metric Names
- The deprecated PromQL endpoint uses `temporal_cloud_v0_*` metrics. The OpenMetrics endpoint uses `temporal_cloud_v1_*` metrics. Never substitute one for the other. The naming patterns differ beyond the version prefix (e.g., `temporal_cloud_v0_frontend_service_request_count` became `temporal_cloud_v1_service_request_count` -- note the dropped `frontend_`).

### No `rate()` on v1 Metrics
- v1 metrics are pre-computed per-second rates with delta temporality. Do NOT apply `rate()`, `increase()`, `irate()`, or `histogram_quantile()` to them. Use `sum()`, `avg()`, `max()`, `min()` and similar aggregation functions directly.
- `rate()` IS correct for SDK metrics (e.g., `temporal_workflow_task_schedule_to_start_latency`) and self-hosted cluster metrics (e.g., `service_requests`), which are cumulative counters/histograms.

### Three Distinct Metric Sources
- **Cloud metrics** (`temporal_cloud_v1_*`): From the OpenMetrics endpoint at `metrics.temporal.io`, using API key auth.
- **SDK metrics** (`temporal_*` without `cloud`): Emitted by Workers and Clients, scraped from application-configured endpoints.
- **Self-hosted cluster metrics** (e.g., `service_requests`, `persistence_latency`): Emitted by the Temporal Service itself, different naming convention.
Never confuse these three sources.

### Authentication
- OpenMetrics endpoint: API key (Bearer token) authentication with a service account having "Metrics Read-Only" Account Level Role.
- PromQL endpoint (deprecated): mTLS certificates with customer-specific endpoints (`<account-id>.tmprl.cloud/prometheus`).
Never mix these authentication methods or endpoint URLs.

### Approximate Backlog Count
- `temporal_cloud_v1_approximate_backlog_count` is approximate. It can overcount (invalid/expired tasks) and reset to zero on idle task queues. Never describe it as exact.

### SDK Metric Unit Differences
- Core-based SDKs (TypeScript, Python, .NET): Histogram metrics default to milliseconds.
- Java and Go SDKs: Histogram metrics default to seconds.

### Percentile Metrics
- v1 provides pre-calculated percentiles (p50, p95, p99). These cannot be accurately aggregated across dimensions. Do not use `histogram_quantile()` on v1 metrics.

### Deprecated Labels
- `worker_version` is deprecated. Use `temporal_worker_deployment_name` and `temporal_worker_build_id` instead.

### Self-Hosted Health Checks
- Frontend Service health checks (TCP or gRPC) are on port 7233, not 8080 or any other port.
