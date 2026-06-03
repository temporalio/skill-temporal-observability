---
name: temporal-observability
description: 'Instrument, collect, query, and alert on Temporal metrics across Temporal Cloud and self-hosted deployments. Use when the user mentions: "Temporal metrics", "Temporal monitoring", "Temporal observability", "Temporal alerting", "Datadog Temporal", "Grafana Temporal", "Cloud metrics endpoint", "scrape Temporal metrics", "PromQL migration".'
version: 0.1.0
---

# Skill: temporal-observability

## Overview

This skill helps users set up metrics collection, write monitoring queries, configure alerts, and diagnose performance issues using Temporal metrics. It operates in three modes:

- **Setup** — generating scrape configs, integration configs, and SDK metrics endpoint code. Always end setup with a validation step: prompt the user to confirm metrics are arriving and queryable in their platform.
- **Query** — writing PromQL/DQL queries for dashboards and alerts.
- **Diagnosis** — recommending which metrics to check for a given symptom and what the values mean.

You produce configs, queries, and code snippets. You do not have access to live metric data, dashboards, or alerting systems. When the user describes symptoms, provide the diagnostic queries and explain what to look for in the results. This skill owns the metrics collection and query layer. When the user needs to act on metric signals (scale workers, change tuning parameters, rotate credentials, administer namespaces), hand off to the appropriate sibling skill and provide the metric evidence gathered so far.

## How to Use This Skill

### Step 1: Determine the metric source

Before answering any metrics question, establish which of the three sources the user is asking about. This determines which reference files apply, which query functions are valid, and which auth/scrape patterns to use:

- **Cloud metrics** (`temporal_cloud_v1_*` only) — from `metrics.temporal.io`, API key auth. Never use `rate()`. Never recommend deprecated `temporal_cloud_v0_*` metrics.
- **SDK metrics** (`temporal_*` without `cloud`) — emitted by Workers/Clients, scraped from app endpoints. `rate()` is correct here.
- **Self-hosted cluster metrics** (`service_requests`, `persistence_latency`, etc.) — emitted by the Temporal Service itself. `rate()` is correct here.

If the source is ambiguous, ask before writing queries.

### Step 2: Determine the deployment context

Cloud, self-hosted, or dev server. This affects scrape configs, auth mechanisms, and which metrics are available.

### Step 3: Determine the observability platform

Prometheus, Datadog, Grafana Cloud, New Relic, ClickStack, or OTel Collector. This affects config format, query language, and available features.

### Step 4: For SDK metrics, determine the SDK language

Histogram units differ: milliseconds for Core-based SDKs (TypeScript, Python, .NET) vs seconds for Go and Java. Getting this wrong produces thresholds that are off by 1000x.

### Multi-file routing for common questions

Most real questions require 2-3 reference files:

- **"Set up monitoring for my Temporal Cloud namespace"** → `openmetrics-endpoint.md` (auth + scrape) + `integrations.md` (platform config) + `service-health-monitoring.md` (what to alert on) + `worker-health-monitoring.md` (worker fleet).
- **"My workers seem slow / tasks are backing up"** → `worker-health-monitoring.md` (backlog, sync match, poll success queries) + `troubleshooting.md` (diagnostic steps). If they need to act: hand off to `skill-temporal-workertuning`.
- **"Migrate from v0 to v1"** → `migration-v0-to-v1.md` (mapping table + auth changes). Only relevant for users already on v0 — never suggest v0 to new users. v0 is deprecated and unavailable to new accounts.
- **"What metrics should I watch?"** → `service-health-monitoring.md` + `worker-health-monitoring.md` for the essential observation set.

## Key Facts

- OpenMetrics endpoint: `metrics.temporal.io`, API key auth, 30s recommended scrape interval, 30k datapoints per scrape, 180 requests/hour rate limit, ~3-minute data latency.
- PromQL/v0 endpoint is deprecated and unavailable to new users. Sunset date October 5, 2026. Never recommend v0 metrics — always use v1.
- v1 metrics are pre-computed per-second rates with delta temporality. Never apply `rate()`, `increase()`, `irate()`, or `histogram_quantile()`.
- SDK histogram units: milliseconds (TypeScript, Python, .NET) vs seconds (Go, Java).
- v1 pre-calculated percentiles (p50, p95, p99) cannot be accurately aggregated across dimensions.
- `temporal_cloud_v1_approximate_backlog_count` is approximate — can overcount and resets to zero on idle queues.
- Self-hosted Frontend Service health checks: port 7233, not 8080.
- `worker_version` label is deprecated — use `temporal_worker_deployment_name` and `temporal_worker_build_id`.

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

### Never Recommend v0 Metrics
- The `temporal_cloud_v0_*` metrics and the PromQL endpoint are deprecated and no longer available to new users. **Never recommend v0 metrics for any purpose.** Do not include v0 metric names in configs, queries, dashboards, or examples. Always use the `temporal_cloud_v1_*` equivalents.
- The only context where v0 metrics should be mentioned is when a user explicitly asks about migrating away from them. In that case, consult `references/migration-v0-to-v1.md` and guide them to the v1 replacements.
- If a user references a v0 metric name (e.g., `temporal_cloud_v0_frontend_service_request_count`), respond with the v1 equivalent and note that v0 is deprecated and being shut down.

### v0 and v1 Names Are Not Interchangeable
- v0 and v1 names differ beyond the version prefix (e.g., `temporal_cloud_v0_frontend_service_request_count` → `temporal_cloud_v1_service_request_count` — note the dropped `frontend_`). When helping users migrate, consult the mapping table in `references/migration-v0-to-v1.md`; do not attempt to guess v1 names by swapping the version prefix.

### Post-Setup Validation
- Setup is not complete until metrics are confirmed arriving and queryable **in the user's destination platform**. Verifying that the source endpoint returns data is not sufficient — the data must be flowing end-to-end into the platform where the user will actually query it. After providing any scrape config, integration config, or SDK metrics setup, always end with a validation step.
- **Validate at the destination, not the source.** Tailor the check to where the data needs to land:
  - **Prometheus**: Query the Prometheus API to confirm data arrived — e.g., `curl -s "http://localhost:9090/api/v1/query?query=temporal_cloud_v1_service_request_count"` should return results with recent timestamps.
  - **Datadog, Grafana Cloud, New Relic, ClickStack**: These don't have local CLI query access — prompt the user to check the platform UI. Tell them exactly what to search for (e.g., "In Datadog Metrics Explorer, search for `temporal_cloud_v1` — do matching metrics appear with recent values?").
- For Cloud metrics, account for ~3-minute data latency — if the destination shows no data yet, wait a few minutes and check again.
- Do not assume setup succeeded just because the config looks correct. Verify at the destination or ask the user to verify.

### Authentication
- OpenMetrics endpoint: API key (Bearer token) with a service account having "Metrics Read-Only" Account Level Role.
- PromQL endpoint (deprecated): mTLS certificates with customer-specific endpoints (`<account-id>.tmprl.cloud/prometheus`).
- Never mix these authentication methods or endpoint URLs.
