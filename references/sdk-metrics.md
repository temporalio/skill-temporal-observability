# SDK Metrics Overview

<!-- Sources: docs/references/sdk-metrics.mdx, docs/cloud/metrics/sdk-metrics-setup.mdx -->

This reference covers SDK metrics: how they differ from Cloud metrics, how to expose a scrape endpoint per SDK, Prometheus configuration for SDK scrape, Grafana data source setup, the key SDK metrics for observability, and metric unit differences across SDKs.

## SDK Metrics vs Cloud Metrics

SDK metrics are emitted by SDK Clients used to start Workers and to start, signal, or query Workflow Executions. Unlike Temporal Cloud metrics (exposed through the OpenMetrics HTTP API endpoint at `metrics.temporal.io`), SDK metrics require setting up a Prometheus scrape endpoint in application code for Prometheus to collect and aggregate. <!-- docs/cloud/metrics/sdk-metrics-setup.mdx:25-26 -->

All SDK metrics are prefixed with `temporal_` before being exported to their configured destination. <!-- docs/references/sdk-metrics.mdx:38 -->

SDK metrics are distinct from Cloud metrics (`temporal_cloud_v1_*`) and self-hosted cluster metrics (e.g., `service_requests`). They are emitted by the application-side SDK, not the Temporal Service.

## Metric Unit Differences Across SDKs
<!-- docs/references/sdk-metrics.mdx:55-65 -->

The unit of measurement for Histogram metrics varies by SDK:

- **Core-based SDKs** (TypeScript, Python, .NET): Histogram metrics are measured in **milliseconds** by default. This can be customized to use seconds. <!-- docs/references/sdk-metrics.mdx:59-61 -->
- **Java and Go SDKs**: Histogram metrics are measured in **seconds**. <!-- docs/references/sdk-metrics.mdx:63 -->

The Core SDK is a shared common core library used by TypeScript, Python, and .NET SDKs. <!-- docs/references/sdk-metrics.mdx:61 -->

## Exposing a Metrics Endpoint
<!-- docs/cloud/metrics/sdk-metrics-setup.mdx:46-66 -->

Each language SDK has its own way of configuring a Prometheus scrape endpoint. See the per-SDK guides:

- [Go SDK](/develop/go/platform/observability#metrics) <!-- docs/cloud/metrics/sdk-metrics-setup.mdx:51 -->
- [Java SDK](/develop/java/platform/observability#metrics) <!-- docs/cloud/metrics/sdk-metrics-setup.mdx:52 -->
- [TypeScript SDK](/develop/typescript/platform/observability#metrics) <!-- docs/cloud/metrics/sdk-metrics-setup.mdx:53 -->
- [Python SDK](/develop/python/platform/observability#metrics) <!-- docs/cloud/metrics/sdk-metrics-setup.mdx:54 -->
- [.NET SDK](/develop/dotnet/platform/observability#metrics) <!-- docs/cloud/metrics/sdk-metrics-setup.mdx:55 -->

Working samples: <!-- docs/cloud/metrics/sdk-metrics-setup.mdx:59-63 -->

- [Go SDK Samples](https://github.com/temporalio/samples-go/tree/main/metrics)
- [Java SDK Samples](https://github.com/temporalio/samples-java/tree/main/core/src/main/java/io/temporal/samples/metrics)
- [TypeScript SDK Samples](https://github.com/temporalio/samples-typescript/tree/main/interceptors-opentelemetry)
- [Python SDK Samples](https://github.com/temporalio/samples-python/tree/main/custom_metric)
- [.NET SDK Samples](https://github.com/temporalio/samples-dotnet/tree/main/src/OpenTelemetry/DotNetMetrics)

Some SDKs use OpenTelemetry to instrument metrics. A Prometheus exporter with OpenTelemetry can be used to expose metrics for scraping. <!-- docs/cloud/metrics/sdk-metrics-setup.mdx:65-66 -->

## Prometheus Configuration for SDK Scrape
<!-- docs/cloud/metrics/sdk-metrics-setup.mdx:68-91 -->

Prometheus must be configured to listen on the scrape endpoints exposed in application code. The following example assumes the scrape endpoint is on port 8077: <!-- docs/cloud/metrics/sdk-metrics-setup.mdx:70-74 -->

```yaml
global:
  scrape_interval: 30s

scrape_configs:
  - job_name: 'temporalsdkmetrics'
    metrics_path: /metrics
    scheme: http
    static_configs:
      - targets:
          - localhost:8077
```

To verify Prometheus is receiving SDK metrics, navigate to `http://localhost:9090` and check **Status > Targets** for the target endpoint status. <!-- docs/cloud/metrics/sdk-metrics-setup.mdx:95-96 -->

## Grafana Data Source Setup
<!-- docs/cloud/metrics/sdk-metrics-setup.mdx:98-120 -->

To add the SDK metrics Prometheus endpoint as a Grafana data source:

1. Go to **Configuration > Data sources**. <!-- docs/cloud/metrics/sdk-metrics-setup.mdx:106 -->
2. Select **Add data source > Prometheus**. <!-- docs/cloud/metrics/sdk-metrics-setup.mdx:107 -->
3. Enter a name (e.g., "Temporal SDK metrics"). <!-- docs/cloud/metrics/sdk-metrics-setup.mdx:108 -->
4. In the **HTTP** section, enter the Prometheus endpoint URL (e.g., `http://localhost:9090`). <!-- docs/cloud/metrics/sdk-metrics-setup.mdx:109-110 -->
5. Enable **Skip TLS Verify** in the **Auth** section (for local setups). <!-- docs/cloud/metrics/sdk-metrics-setup.mdx:111 -->
6. Click **Save and test** to verify. <!-- docs/cloud/metrics/sdk-metrics-setup.mdx:112 -->

Community-driven example dashboards for Temporal SDKs are available at: https://github.com/temporalio/dashboards/tree/master/sdk <!-- docs/cloud/metrics/sdk-metrics-setup.mdx:128-129 -->

## Common Tags on SDK Metrics
<!-- docs/references/sdk-metrics.mdx:67-84 -->

Each metric may have some combination of the following tags:

| Tag | Description |
|---|---|
| `task-queue` | Task Queue that the Worker Entity is polling |
| `namespace` | Namespace the Worker is bound to |
| `poller_type` | One of: `workflow_task`, `activity_task`, `nexus_task` (Go and Java only), `sticky_workflow_task` |
| `worker_type` | One of: `ActivityWorker`, `WorkflowWorker`, `LocalActivityWorker` (Go and Java only), `NexusWorker` (Go and Java only) |
| `activity_type` | The name of the Activity Function |
| `workflow_type` | The name of the Workflow Function |
| `operation` | RPC method name; available for metrics related to Temporal Client gRPC requests |

Some tags may not be available in every SDK, and Histogram metrics may have different buckets in each SDK. <!-- docs/references/sdk-metrics.mdx:85 -->

## Key SDK Metrics for Observability

The following are the most important SDK metrics for operational monitoring. The full list is in `docs/references/sdk-metrics.mdx`.

### Schedule-to-Start Latency

#### `temporal_workflow_task_schedule_to_start_latency`
<!-- docs/references/sdk-metrics.mdx:550-557 -->

The Schedule-To-Start time of a Workflow Task.

- Type: Histogram
- Available in: Core, Go, Java
- Tags: `namespace`, `task_queue`

#### `temporal_activity_schedule_to_start_latency`
<!-- docs/references/sdk-metrics.mdx:171-182 -->

The Schedule-To-Start time of an Activity Task in seconds. Useful for ensuring Activity Tasks are being processed from the queue in a timely manner. <!-- docs/references/sdk-metrics.mdx:173-177 -->

- Type: Histogram
- Available in: Core, Go, Java
- Tags: `namespace`, `task_queue`

### Execution Latency

#### `temporal_workflow_task_execution_latency`
<!-- docs/references/sdk-metrics.mdx:518-524 -->

Workflow Task Execution time.

- Type: Histogram
- Available in: Core, Go, Java
- Tags: `namespace`, `task_queue`, `workflow_type`

#### `temporal_activity_execution_latency`
<!-- docs/references/sdk-metrics.mdx:155-161 -->

Time to complete an Activity Execution, from the time the Activity Task is generated to the time the language SDK responded with a completion (failure or success).

- Type: Histogram
- Available in: Core, Go, Java
- Tags: `activity_type`, `namespace`, `task_queue`

#### `temporal_workflow_endtoend_latency`
<!-- docs/references/sdk-metrics.mdx:489-495 -->

Total Workflow Execution time from schedule to completion for a single Workflow Run. A retried Workflow Execution is a separate Run.

- Type: Histogram
- Available in: Core, Go, Java
- Tags: `namespace`, `task_queue`, `workflow_type`

### Replay Latency

#### `temporal_workflow_task_replay_latency`
<!-- docs/references/sdk-metrics.mdx:543-548 -->

Time to catch up on replaying a Workflow Task.

- Type: Histogram
- Available in: Core, Go, Java
- Tags: `namespace`, `task_queue`, `workflow_type`

### Task Slots

#### `temporal_worker_task_slots_available`
<!-- docs/references/sdk-metrics.mdx:437-445 -->

The total number of Workflow, Activity, Local Activity, or Nexus Task execution slots that are currently available. Use the `worker_type` tag to differentiate execution slots.

- Type: Gauge
- Available in: Core, Go, Java
- Tags: `namespace`, `task_queue`, `worker_type`

#### `temporal_worker_task_slots_used`
<!-- docs/references/sdk-metrics.mdx:448-456 -->

The total number of Workflow, Activity, Local Activity, or Nexus Task execution slots in current use. Use the `worker_type` tag to differentiate execution slots.

- Type: Gauge
- Available in: Core, Go, Java
- Tags: `namespace`, `task_queue`, `worker_type`

### Sticky Cache

#### `temporal_sticky_cache_hit`
<!-- docs/references/sdk-metrics.mdx:388-394 -->

A Workflow Task found a cached Workflow Execution to run against.

- Type: Counter
- Available in: Core, Go, Java
- Tags: `namespace`, `task_queue`

#### `temporal_sticky_cache_miss`
<!-- docs/references/sdk-metrics.mdx:396-402 -->

A Workflow Task did not find a cached Workflow Execution to run against.

- Type: Counter
- Available in: Core, Go, Java
- Tags: `namespace`, `task_queue`

#### `temporal_sticky_cache_size`
<!-- docs/references/sdk-metrics.mdx:404-410 -->

Current cache size, expressed in number of Workflow Executions.

- Type: Gauge
- Available in: Core, Go, Java
- Tags: `namespace` (TypeScript, Java), `task_queue` (TypeScript)

### Poll Metrics

#### `temporal_workflow_task_queue_poll_succeed`
<!-- docs/references/sdk-metrics.mdx:534-540 -->

A Workflow Worker polled a Task Queue and successfully picked up a Workflow Task.

- Type: Counter
- Available in: Core, Go, Java
- Tags: `namespace`, `task_queue`

#### `temporal_workflow_task_queue_poll_empty`
<!-- docs/references/sdk-metrics.mdx:526-532 -->

A Workflow Worker polled a Task Queue and timed out without picking up a Workflow Task.

- Type: Counter
- Available in: Core, Go, Java
- Tags: `namespace`, `task_queue`

#### `temporal_activity_poll_no_task`
<!-- docs/references/sdk-metrics.mdx:163-169 -->

An Activity Worker poll for an Activity Task timed out, and no Activity Task is available.

- Type: Counter
- Available in: Core, Go, Java
- Tags: `namespace`, `task_queue`

### Failure Metrics

#### `temporal_workflow_task_execution_failed`
<!-- docs/references/sdk-metrics.mdx:505-517 -->

A Workflow Task Execution failed.

- Type: Counter
- Available in: Core, Go, Java
- Tags: `namespace`, `task_queue`, `workflow_type`, `failure_reason`

Valid values for `failure_reason`: <!-- docs/references/sdk-metrics.mdx:513-516 -->
- `NonDeterminismError`: The Workflow Task failed due to a non-determinism error.
- `WorkflowError`: The Workflow Task failed for any other reason.

#### `temporal_activity_execution_failed`
<!-- docs/references/sdk-metrics.mdx:148-153 -->

An Activity Execution failed. Does not include local Activity Failures in Go and Java SDKs.

- Type: Counter
- Available in: Core, Go, Java
- Tags: `activity_type`, `namespace`, `task_queue`

## Complete Metric List

For the full table of all SDK metrics, their types, and SDK availability, see `docs/references/sdk-metrics.mdx:87-137`. <!-- docs/references/sdk-metrics.mdx:87-137 -->

SDK metric definitions are maintained in the following source locations: <!-- docs/references/sdk-metrics.mdx:49-53 -->
- [Core SDK Worker metrics](https://github.com/temporalio/sdk-core/blob/master/crates/sdk-core/src/telemetry/metrics.rs)
- [Core SDK Client metrics](https://github.com/temporalio/sdk-core/blob/master/crates/client/src/metrics.rs)
- [Java SDK Worker metrics](https://github.com/temporalio/sdk-java/blob/master/temporal-sdk/src/main/java/io/temporal/worker/MetricsType.java)
- [Java SDK Client metrics](https://github.com/temporalio/sdk-java/blob/master/temporal-serviceclient/src/main/java/io/temporal/serviceclient/MetricsType.java)
- [Go SDK Worker and Client metrics](https://github.com/temporalio/sdk-go/blob/c32b04729cc7691f80c16f80eed7f323ee5ce24f/internal/common/metrics/constants.go)
