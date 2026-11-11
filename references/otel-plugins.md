# OpenTelemetry Plugins (Tracing and Metrics)

Read for Temporal traces or OTel-exported SDK metrics in Go, Python, or Java (incl. Cloud Run workers). Prometheus scrape: `sdk-metrics.md`.

For every SDK: create the replay-safe provider and set it as the OTel global first, then register the plugin on the Client (Workers inherit it).

## Go (`go.temporal.io/sdk/contrib/opentelemetry-v2`, experimental)

```go
provider := temporalotel.NewReplaySafeTracerProvider(sdktrace.WithBatcher(exporter))
otel.SetTracerProvider(provider)
plugin, err := temporalotel.NewPlugin(temporalotel.PluginOptions{})
c, err := client.Dial(client.Options{Plugins: []client.Plugin{plugin}})
```

- No Temporal spans by default; set `PluginOptions.TracerOptions.AddTemporalSpans`.
- In workflow code use `temporalotel.Tracer("name").Start(ctx, ...)`; elsewhere `otel.Tracer(...)`.
- Metrics: set `PluginOptions.MetricsHandlerOptions`.

## Python (`temporalio.contrib.opentelemetry`, experimental)

```python
provider = create_tracer_provider()
provider.add_span_processor(SimpleSpanProcessor(ConsoleSpanExporter()))
opentelemetry.trace.set_tracer_provider(provider)
client = await Client.connect("localhost:7233", plugins=[OpenTelemetryPlugin()])
```

- Prefer `OpenTelemetryPlugin` over `TracingInterceptor` (zero-duration workflow spans); workflow spans appear only after the workflow completes.
- Global OTel meters recorded from workflow code re-record on replay: wrap with `ReplaySafeMeterProvider` and set it first (`set_meter_provider` works once per process). `workflow.metric_meter()` is already replay-safe.

## Cloud Run workers

- Go `go.temporal.io/sdk/contrib/gcp/cloudrun/otel`: `otel.NewPlugin(ctx, otel.PluginOptions{})`.
- Python `temporalio.contrib.gcp.cloud_run.opentelemetry.OpenTelemetryPlugin` (`temporalio[cloud-run-worker-otel]`).
- Java `temporal-gcp-cloud-run-opentelemetry`: `CloudRunOpenTelemetryPlugin.newBuilder().build()` via `WorkflowServiceStubsOptions.setPlugins`.
- Do not combine with another OTel metrics handler, tracing interceptor or generic OTel plugin on the same client.
- Exports OTLP to a collector (not Google Cloud directly): plugin option, then `OTEL_EXPORTER_OTLP_ENDPOINT`, then `http://localhost:4317`. Metrics every 60s.
- For Managed Service for Prometheus, no collector `batch` processor in the metrics pipeline.
- Flush on shutdown within Cloud Run's ~10s window (Go: `otelPlugin.Shutdown(ctx)`, ~8s). Java: `plugin.newFlushHook().run(Duration)` after `awaitTermination`.
