# Go OpenTelemetry: v1 and v2

Read when a Go user asks how to export SDK metrics or traces with OpenTelemetry, or which OTel module to use.

| | v1 `go.temporal.io/sdk/contrib/opentelemetry` | v2 `go.temporal.io/sdk/contrib/opentelemetry-v2` |
|---|---|---|
| Stage | Latest release v0.8.1 | Experimental: APIs may change; the docs call it Pre-release |

## v2 setup

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

## v1 monotonic counters

`MetricsHandlerOptions.UseMonotonicCounters` defaults to `false`, so SDK counters are OTel `Int64UpDownCounter`. Set it to `true` to use `Int64Counter` so exporters classify them as monotonic sums; `Inc` values must then be non-negative.

v1 contrib/opentelemetry v0.8.1 requires Go SDK v1.46.0 (v0.8.0 declared a stale v1.12.0 minimum).
