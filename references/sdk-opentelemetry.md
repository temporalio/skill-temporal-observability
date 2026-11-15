# SDK OpenTelemetry and Tracing

Choose the SDK below for OpenTelemetry metrics export, tracing interceptors or plugins, and replay-safe instrumentation. Java's OpenTracing setup is included in its SDK section.

| SDK | Guidance |
|---|---|
| [.NET](#net) | Nexus trace source and standalone Activity spans |
| [Go](#go) | v1/v2 modules, plugin setup, metrics and standalone Activity spans |
| [Java](#java) | Standalone Activity interceptor and Update-with-Start spans |
| [Python](#python) | Plugin, replay-safe providers and metrics exporter |
| [TypeScript](#typescript) | v1/v2 packages, compatibility and imports |
| [Rust](#rust) | Experimental plugin, context propagation and replay-safe spans |

For metric definitions, units and Prometheus scraping, read [SDK metrics](sdk-metrics.md). For AWS Lambda or Google Cloud Run deployment, read [serverless OpenTelemetry](serverless-otel.md).

## .NET

Add `TracingInterceptor.NexusSource.Name` to the tracer provider with `AddSource`.

Starting a standalone Activity from a client creates a `StartActivity:<activity type>` span, tagged with the Activity ID when `TagNameActivityId` is set.

## Go

| | v1 `go.temporal.io/sdk/contrib/opentelemetry` | v2 `go.temporal.io/sdk/contrib/opentelemetry-v2` |
|---|---|---|
| Version / stage | v0.8.1 | Experimental: APIs may change; the docs call it Pre-release |

### v2 Plugin Setup

Install with `go get go.temporal.io/sdk/contrib/opentelemetry-v2@latest`.

Create a replay-safe tracer provider, set it as the OTel global, then register the plugin. `NewPlugin` requires the global provider to come from `NewReplaySafeTracerProvider`.

```go
provider := temporalotel.NewReplaySafeTracerProvider(sdktrace.WithBatcher(exporter))
otel.SetTracerProvider(provider)
defer provider.Shutdown(ctx)

plugin, err := temporalotel.NewPlugin(temporalotel.PluginOptions{})
c, err := client.Dial(client.Options{Plugins: []client.Plugin{plugin}})
w := worker.New(c, "my-task-queue", worker.Options{}) // workers get the plugin automatically
```

- By default the plugin creates no spans of its own; it only propagates trace context. Set `PluginOptions.TracerOptions.AddTemporalSpans` to emit a span per Temporal operation (`StartWorkflow`, `RunWorkflow`, `RunActivity`, `ContinueAsNew`).
- Metrics: set `PluginOptions.MetricsHandlerOptions` (non-nil) and the plugin installs an OTel metrics handler on the client.
- Meter Provider: OTel's default global Meter Provider is a no-op, so `MetricsHandlerOptions` records nothing unless a configured one is set with `otel.SetMeterProvider` before creating the plugin, or a meter is passed in `MetricsHandlerOptions.Meter`.
- If `Meter` is unset, the handler uses the global meter provider's meter named `temporal-sdk-go`.
- `UseMonotonicCounters` is also on v2's `MetricsHandlerOptions`; example enabling both spans and metrics:

```go
plugin, err := temporalotel.NewPlugin(temporalotel.PluginOptions{
	TracerOptions:         tracing.TracerOptions{AddTemporalSpans: true},
	MetricsHandlerOptions: &temporalotel.MetricsHandlerOptions{UseMonotonicCounters: true},
})
```
- In workflow code build the tracer with `temporalotel.Tracer("name")` and call `tracer.Start(workflowCtx, "span")`; in client and activity code use `otel.Tracer(...)`.

### v1 Monotonic Counters

`MetricsHandlerOptions.UseMonotonicCounters` defaults to `false`, so SDK counters are OTel `Int64UpDownCounter`. Set it to `true` to use `Int64Counter` so exporters classify them as monotonic sums; `Inc` values must then be non-negative.

v1 contrib/opentelemetry v0.8.1 requires Go SDK v1.46.0 (v0.8.0 declared a stale v1.12.0 minimum).

### Standalone Activity Tracing

The client-side interceptor creates a `StartActivity` span, tagged with the Activity ID, when a standalone Activity is started.

## Java

For standalone Activities, register `OpenTracingActivityClientInterceptor` with `ActivityClientOptions.newBuilder().setInterceptors(...)`.

`updateWithStart` is traced with the `UpdateWithStartWorkflow` operation type.

## Python

### Tracing and Replay Safety

- `OpenTelemetryPlugin(add_temporal_spans=False)` expects your tracer provider to be created with `create_tracer_provider()`; `add_temporal_spans` adds Temporal-specific spans such as StartWorkflow and RunWorkflow.
- `ReplaySafeMeterProvider` and `ReplaySafeLoggerProvider` drop recordings made from workflow code only while replaying; queries and update validators are not dropped.
- `ReplaySafeMeterProvider` wraps counters, up-down counters, histograms and gauges; observable instruments pass through unwrapped.
- Spans are exported for rejected updates and failed queries. Replay-safe spans end when running read-only (queries and update validators), even when holding a non-failure exception.

### Metrics Export

- `MetricsExporter` (Experimental; requires Python SDK 1.33.0+) drains a `temporalio.runtime.MetricBuffer` into an OpenTelemetry `MeterProvider` every `poll_interval` (default 1 second).
- Wire the buffer into a `Runtime`, for example `Runtime(telemetry=TelemetryConfig(metrics=MetricBuffer(10_000)))` (10,000 is an example size). A full buffer drops updates, and the running exporter is what drains it.
- Start it with `async with MetricsExporter(buffer, meter_provider):` or `run()` once the `Runtime` using the buffer exists; `shutdown()` stops it after a final drain. `run()` raises `RuntimeError` if the buffer was never attached to a `Runtime`.
- `on_error` is called, in addition to logging, when draining or applying one update fails; if it raises, that is logged and ignored.
- Counters are added, histograms recorded, and gauges exported as an observable gauge holding the latest value per attribute set.

## TypeScript

Two packages exist: `@temporalio/interceptors-opentelemetry` for OpenTelemetry JS SDK v1, and `@temporalio/interceptors-opentelemetry-v2` for OpenTelemetry JS SDK v2, which is Experimental.

The v2 package produces histories incompatible with the v1 package. Replaying a Workflow first run with the v1 package can cause a non-determinism error.

In both packages, import Workflow interceptors from `<package>/workflow` and use the package root for the plugin, Client interceptors, and Worker interceptors.

## Rust

`OpenTelemetryPlugin` is Experimental. Enable both the `experimental` and `opentelemetry` crate features.

Add it with `ClientOptions::new("default").plugin(OpenTelemetryPlugin::builder().build())`. Trace context travels in the `_tracer-data` header; W3C Trace Context and W3C Baggage are the default propagators.

`OpenTelemetryPluginBuilder::encode_client_headers(true)` applies the client's codec to the plugin's tracing header only. It defaults to false.

Use `WorkflowIdGenerator` in the tracer provider for replay-safe Workflow spans, and wrap span processors in `WorkflowSpanProcessor` if application code creates Workflow spans.

The Rust interceptor API does not support inbound Nexus handler interception.
