# OpenTelemetry for Serverless Workers (AWS Lambda, Google Cloud Run)

Covered here: Python and Go.

## AWS Lambda

Call the helper inside the `run_worker` / `lambdaworker.RunWorker` configure callback. It sets up Core metrics export and tracing for the ADOT (AWS Distro for OpenTelemetry) Lambda layer.

**Python** (`temporalio.contrib.aws.lambda_worker.otel`; install the `lambda-worker-otel` extra):

```python
from temporalio.contrib.aws.lambda_worker.otel import apply_defaults, OtelOptions

def configure(config):
    ...
    apply_defaults(config, OtelOptions())
```

- Collector endpoint defaults to `http://localhost:4317`, as expected by the ADOT layer.
- Service name falls back to `OTEL_SERVICE_NAME`, then `AWS_LAMBDA_FUNCTION_NAME`, then `temporal-lambda-worker`; endpoint falls back to `OTEL_EXPORTER_OTLP_ENDPOINT`.
- `metric_periodicity` defaults to 10 seconds. There is no explicit metrics flush API, so set it shorter than the Lambda timeout to get at least one export per invocation.
- Traces are force-flushed by a per-invocation shutdown hook.
- `configure` runs on every invocation, but the global tracer provider is installed once per process and reused on warm invocations. Before 1.36.0, each warm invocation built a new tracer provider and span processor that the global provider ignored, leaking them; 1.36.0 fixed this.
- For only one half: `apply_tracing(config)` configures tracing only, and `build_metrics_telemetry_config()` builds a `TelemetryConfig` for your own `Runtime`.

**Go** (`go.temporal.io/sdk/contrib/aws/lambdaworker/otel`, a separately released module):

```go
otel.ApplyDefaults(options, &options.ClientOptions, otel.Options{})
```

It configures OTLP gRPC exporters, X-Ray-compatible trace IDs and per-invocation flushing. `MetricExportInterval` defaults to 10 seconds.

## Google Cloud Run

Stage: the Python `OpenTelemetryPlugin` is Experimental.

The plugin exports metrics and traces over OTLP gRPC to `localhost:4317`, not directly to Google Cloud, so a collector sidecar (Google-Built OpenTelemetry Collector) is required.

**Python** (`temporalio.contrib.gcp.cloud_run.opentelemetry`; install the `cloud-run-worker-otel` extra): pass one `OpenTelemetryPlugin()` in `Client.connect(plugins=[...])`. Do not also install the generic OpenTelemetry plugin on that client.

**Go** (`go.temporal.io/sdk/contrib/gcp/cloudrun/otel`): `otel.NewPlugin(ctx, otel.PluginOptions{})`, passed in `client.Options.Plugins`.

- Metrics export every 60 seconds by default.
- Service name falls back to `OTEL_SERVICE_NAME`, `CLOUD_RUN_WORKER_POOL`, `K_SERVICE`, then `temporal-worker`.
- Custom `Runtime`: `OpenTelemetryPlugin(runtime=rt)` leaves Core metrics config alone, and combining `runtime` with `metric_periodicity` raises `ValueError` at construction. Set the period in `build_metrics_telemetry_config(*, endpoint=None, service_name=None, metric_periodicity=None)`, a keyword-only helper in the same module returning a `TelemetryConfig` for `Runtime(telemetry=...)`. Pass the runtime to the plugin, not also a different one to `Client.connect` (that raises `ValueError`).
- Collector: do not put a `batch` processor on the metrics pipeline to Google Managed Prometheus (it can fail with `Duplicate TimeSeries`); keep batching on the traces pipeline.
- Shutdown: stop every worker, then flush before Cloud Run sends SIGKILL: Python `plugin.shutdown(timeout)`; Go close the client, then `otelPlugin.Shutdown(ctx)` (README example uses 8s).
