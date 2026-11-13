# SDK Distributed Tracing (.NET, Go, Java, TypeScript, Rust)

This reference covers the OpenTelemetry and OpenTracing tracing interceptors for non-Python SDKs, which propagate trace context in Temporal headers across clients, Workflows, and Activities. For SDK metrics see `sdk-metrics.md`.

## .NET

Add `TracingInterceptor.NexusSource.Name` to the tracer provider with `AddSource`.

Starting a standalone Activity from a client creates a `StartActivity:<activity type>` span, tagged with the Activity ID when `TagNameActivityId` is set.

## Go

The client-side interceptor creates a `StartActivity` span, tagged with the Activity ID, when a standalone Activity is started.

## Java

For standalone Activities, register `OpenTracingActivityClientInterceptor` with `ActivityClientOptions.newBuilder().setInterceptors(...)`.

`updateWithStart` is traced with the `UpdateWithStartWorkflow` operation type.

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
