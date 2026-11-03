# OpenTelemetry conformance for Dapr traces and metrics

* Author(s): @MyMirelHub
* State: Ready for Implementation
* Updated: 2026-11-03

## Overview

This proposal brings daprd's telemetry in line with the OpenTelemetry specification and semantic conventions. It affects the runtime (`pkg/diagnostics`, the HTTP and gRPC API servers, the app channels, pub/sub delivery, bindings, the metrics exporter and tracing configuration) and the workflow engine (`dapr/durabletask-go`). No API, component or SDK changes are needed.

It proposes to:

1. Emit spans with the span kinds, names, attributes and status the semantic conventions define, and add the missing spans for calls to components, to other sidecars and to the app.
2. Propagate trace context the way W3C Trace Context and Baggage require.
3. Configure the SDK (resource, OTLP exporter, sampler, propagators) as the OpenTelemetry environment variable specification defines.
4. Replace OpenCensus with the OpenTelemetry metrics SDK, keeping today's Prometheus metrics unchanged and adding OTLP export, exemplars and the semantic convention duration metrics.
5. Remove the workflow engine's `unsafe` changes to span internals and align workflow spans with the specification.

## Background

Dapr exports traces with the OpenTelemetry Go SDK, but what it puts in those traces does not follow the OpenTelemetry conventions. Observability backends build service maps, error rates, database and messaging views and dashboards from those conventions, so the gaps show up to users as broken or misleading telemetry rather than as missing features:

* **Service maps don't connect.** Every inbound API call is recorded as a CLIENT span, and no CLIENT span is recorded for calls to other sidecars or to the app. Tools that build service graphs from CLIENT/SERVER and PRODUCER/CONSUMER pairs (Tempo, Jaeger SPM, Application Insights, Datadog) cannot draw edges for Dapr hops.
* **Error rates are inflated.** Every HTTP 4xx and every non-OK gRPC code marks a server span as failed, so a `NotFound` from a state get counts as an error.
* **Attributes are misleading.** `db.system` is set to the building block (`state`, `secrets`, `bindings`) instead of the database, `messaging.system` is always `pubsub`, `db.connection_string` holds the building block name, `db.statement` holds the request path including state keys, `rpc.system` holds the Conversation component name, and `server.address` holds the client's address. Several of these attributes have been removed from the conventions.
* **Span names have unbounded cardinality.** HTTP spans are named after the request path, and service invocation spans embed the user supplied method.
* **Propagation sends invalid or no context.** Without a configured exporter, daprd forwards an all-zero `traceparent`, which W3C forbids. Invalid `baggage` fails the request with a 400 or `InvalidArgument`. An oversized `tracestate` is dropped instead of truncated. The OpenCensus `grpc-trace-bin` header takes precedence over `traceparent`.
* **Pub/sub traces are incomplete.** Subscriber spans are CLIENT spans named `pubsub/<topic>`, bulk delivery creates one span per message instead of one span with links, and publishing has no producer span.
* **SDK configuration deviates from the specification.** The resource lacks the required `telemetry.sdk.*` attributes and `service.instance.id`. `OTEL_EXPORTER_OTLP_ENDPOINT=http://...` still uses TLS, any path in the endpoint is ignored, and `OTEL_TRACES_SAMPLER`, `OTEL_PROPAGATORS` and `OTEL_SDK_DISABLED` have no effect.
* **Metrics depend on OpenCensus**, which was archived in 2023. There is no OTLP metrics export and no exemplars to link a latency spike to a trace.
* **Workflow tracing changes span internals with `unsafe`.** To keep one span across replays, durabletask-go overwrites the SDK span's private span context through reflection and clears the sampled flag of intermediate spans. This depends on otel-go internals and silently does nothing with other tracer providers.

Users have reported many of these problems separately over the last three years, and most of those issues were closed as stale without a fix (see Related issues). Addressing them one at a time has not worked because they share a cause: the instrumentation was built before the conventions it should follow existed or stabilised. This proposal treats them as one change so the result is consistent and can be tested as a whole.

## Related Items

### Related proposals

* [20240517-R-http-metrics-path-matching](./20240517-R-http-metrics-path-matching.md): the HTTP metrics path matching it introduced is kept and also provides `http.route` for the new duration metrics.

### Related issues

Open:

* [dapr/dapr#10773](https://github.com/dapr/dapr/issues/10773) Emit tracing spans for state store operations
* [dapr/dapr#10462](https://github.com/dapr/dapr/issues/10462) gRPC service-to-service invocation traces not propagated
* [dapr/dapr#10979](https://github.com/dapr/dapr/issues/10979) API logging output carries no trace or span ID (out of scope, see below)
* [dapr/dapr#3994](https://github.com/dapr/dapr/issues/3994) Add more detail for tracing
* [dapr/dapr#5230](https://github.com/dapr/dapr/issues/5230) Correlate actor reminder creation and execution events
* [dapr/dapr#4708](https://github.com/dapr/dapr/issues/4708) Issues with `go.opentelemetry.io/otel` dependency

Closed without a fix:

* [dapr/dapr#7764](https://github.com/dapr/dapr/issues/7764) Replace sunsetted OpenCensus for metrics/traces with OTel equivalent
* [dapr/dapr#7225](https://github.com/dapr/dapr/issues/7225) Add support for metrics and logging in OpenTelemetry integration
* [dapr/dapr#9032](https://github.com/dapr/dapr/issues/9032) Align Pub/Sub tracing with OpenTelemetry semantic conventions
* [dapr/dapr#9471](https://github.com/dapr/dapr/issues/9471) Improve OTel resource information
* [dapr/dapr#9308](https://github.com/dapr/dapr/issues/9308) Conversation API tracing improvements
* [dapr/dapr#8877](https://github.com/dapr/dapr/issues/8877) Missing messaging span attribute in pub/sub tracing
* [dapr/dapr#7505](https://github.com/dapr/dapr/issues/7505) Propagate traces to component implementations
* [dapr/dapr#9451](https://github.com/dapr/dapr/issues/9451), [dapr/dapr#8340](https://github.com/dapr/dapr/issues/8340) Sidecar and control plane telemetry over OTLP

Previously fixed in part:

* [dapr/dapr#5225](https://github.com/dapr/dapr/issues/5225), [dapr/dapr#7703](https://github.com/dapr/dapr/issues/7703), [dapr/dapr#9990](https://github.com/dapr/dapr/issues/9990), [dapr/dapr#6176](https://github.com/dapr/dapr/issues/6176)

## Expectations and alternatives

### In scope

* Span kinds, names, attributes, status and `error.type` for every span daprd emits, following semantic conventions v1.43.
* New CLIENT spans for component operations, calls to other sidecars and calls to the app, and PRODUCER and CONSUMER spans for pub/sub.
* W3C Trace Context and Baggage behaviour, plus the Level 2 `traceresponse` header.
* Resource detection, `OTEL_*` environment variables and the global propagator.
* OpenCensus to OpenTelemetry metrics, with identical Prometheus output, OTLP export, exemplars and semantic convention duration metrics.
* Workflow span lifecycle in durabletask-go.

### Not in scope

* Trace and span IDs in daprd logs and OTLP log export ([#10979](https://github.com/dapr/dapr/issues/10979)). They need their own design because logging lives in `dapr/kit`.
* Tracing for the control plane services.
* Spans created inside components (database driver spans). Component spans in this proposal are created by the runtime around each component call.
* Renaming existing Prometheus metrics. Their names and units stay as they are; the semantic convention metrics are added next to them.

### Alternatives considered

* **Fix issues one at a time.** This is what has happened so far. Each change was small but left the rest inconsistent, and most issues were closed as stale.
* **Dual-emit old and new attributes behind `OTEL_SEMCONV_STABILITY_OPT_IN`**, the migration pattern OpenTelemetry uses for its own instrumentation. This avoids breaking existing trace queries for one release but doubles attribute volume, and span kinds and names cannot be dual-emitted. It remains an option if maintainers prefer a softer rollout for attributes.
* **Use the contrib `otelhttp` and `otelgrpc` instrumentation** for the API servers. They do not know about Dapr's routes, components or invocation targets, and Dapr's propagation has to keep `grpc-trace-bin` for older sidecars, so most of the custom logic would remain.
* **Keep OpenCensus metrics and add OTLP through a bridge.** The OpenCensus bridge is itself deprecated and keeps the archived dependency.

### Trade-offs

* **Breaking change for trace consumers.** Dashboards, alerts and queries that use the current span names, span kinds or attributes need updating. The Prometheus metrics are not affected.
* **More spans per request.** Each request gains a component or outgoing call span. They are sampled with the request, so the cost scales with the sampling rate.
* **Histogram bucket boundaries.** OpenCensus counted a value equal to a bucket bound in the next bucket, contrary to the `le` label; OpenTelemetry counts it in that bucket. Values exactly on a bound move down one bucket.

## Implementation Details

### Design

#### Spans

| Operation | Kind | Name | Main attributes |
|---|---|---|---|
| HTTP API call | SERVER | `{method} {http.route}` | `http.request.method`, `http.route`, `http.response.status_code`, `url.path`, `url.scheme`, `client.address`, `server.address` |
| gRPC API call, sidecar to sidecar call, proxied call | SERVER | `{rpc.method}` | `rpc.system.name=grpc`, `rpc.method`, `rpc.response.status_code` |
| Call to another sidecar or to the app (gRPC) | CLIENT | `{rpc.method}` | as above, plus `server.address`, `server.port` |
| Call to the app or an HTTP endpoint (HTTP) | CLIENT | `{method}` | `url.full` (credentials and signature parameters redacted), `http.response.status_code` |
| State and configuration store operation | CLIENT | `{db.operation.name}` | `db.system.name` mapped from the component type (`state.redis` to `redis`), `db.operation.name` |
| Conversation operation | CLIENT | `chat` | `gen_ai.provider.name`, `gen_ai.operation.name` |
| Other component operation | CLIENT | `{operation} {component}` | `dapr.component.*` only |
| Publish | PRODUCER | `publish {topic}` | `messaging.system` mapped from the component type, `messaging.destination.name`, `messaging.operation.type=send`, `messaging.message.id`, `cloudevents.*` |
| Bulk publish | PRODUCER per message, CLIENT for the batch | `create {topic}`, `publish {topic}` | the batch span links to each message span and sets `messaging.batch.message_count` |
| Delivery to the app | CONSUMER | `process {topic}` | child of the producer context; a new trace when the message has none |
| Bulk delivery | CONSUMER | `process {topic}` | one span linked to each message's producer context |
| Input binding event | CONSUMER | `process {binding}` | `dapr.component.*` |

All component spans carry `dapr.component.name`, `dapr.component.type` and `dapr.component.operation`. Dapr specific API attributes keep the `dapr.` namespace (`dapr.api`, `dapr.app_id`, `dapr.invoke_method`, `dapr.actor.type`, `dapr.actor.id`). The removed or misused attributes (`db.system` with a building block value, `db.connection_string`, `db.statement`, `net.peer.name`, `rpc.service`, `dapr.status_code`) are no longer set.

Component spans need the component type. The runtime registers a resolver from component name to type when tracing is set up, so call sites only pass the component name.

Outgoing CLIENT spans are created by gRPC client interceptors on the connections to the app and to other sidecars, and by a wrapper around the HTTP app channel. They are only created when the call is part of a trace, so health checks and subscription listing do not start traces. Stream spans end when the stream completes.

#### Status

* HTTP SERVER spans fail on 5xx only. CLIENT and CONSUMER spans fail on 4xx and 5xx.
* gRPC SERVER spans fail only on `UNKNOWN`, `DEADLINE_EXCEEDED`, `UNIMPLEMENTED`, `INTERNAL`, `UNAVAILABLE` and `DATA_LOSS`. CLIENT spans fail on every code other than `OK`.
* Failed spans set `error.type`.

#### Propagation

* `traceparent` is preferred over `grpc-trace-bin` on incoming calls. Both are still sent to other sidecars so mixed-version clusters keep working during an upgrade.
* Parsing follows W3C strictly (lowercase hex, exact field lengths for version `00`, future versions accepted). Extracted contexts are marked remote.
* Invalid or all-zero contexts are never propagated.
* `tracestate` is truncated as W3C specifies: long entries first, then entries from the end.
* Invalid baggage is dropped, not rejected.
* Responses carry `traceresponse`. `traceparent` is still returned for existing clients.
* A global propagator is registered from `OTEL_PROPAGATORS`, defaulting to `tracecontext,baggage`, so libraries inside daprd that use the OpenTelemetry API propagate as well.

#### SDK configuration

* The resource includes `telemetry.sdk.*`, `telemetry.distro.name=dapr`, `telemetry.distro.version`, `host.name` and `service.instance.id`. On Kubernetes it also includes `k8s.pod.name` and `k8s.namespace.name`, and `service.instance.id` is the UUIDv5 the conventions recommend. `OTEL_RESOURCE_ATTRIBUTES` and `OTEL_SERVICE_NAME` take precedence.
* `OTEL_EXPORTER_OTLP_*` follow the exporter specification: the endpoint scheme decides TLS, the generic endpoint gets `/v1/traces` appended for HTTP, `_TRACES_INSECURE` is read, and `http/json` logs a warning. gRPC stays the default protocol for backwards compatibility, which the specification allows. The Configuration CRD also accepts a URL as the endpoint.
* `OTEL_TRACES_SAMPLER` and `OTEL_TRACES_SAMPLER_ARG` set the rate when the Configuration does not. Dapr always respects the parent's decision, so non-parent-based samplers are applied as their parent-based variants.
* `OTEL_SDK_DISABLED=true` disables trace export.
* Using the Zipkin exporter logs a deprecation warning, following OpenTelemetry's deprecation of the Zipkin exporter specification.

#### Metrics

* All instruments move to the OpenTelemetry metrics SDK. The Prometheus exporter is configured without unit or `_total` suffixes, scope labels or `target_info`, so the exposition stays byte-identical to today's. A golden test generated from the current OpenCensus output enforces this.
* Path matching, `increasedCardinality`, `excludeVerbs`, custom latency buckets, metric rules and `recordErrorCodes` keep working. The SDK's default cardinality cap is disabled to match OpenCensus unless `OTEL_GO_X_CARDINALITY_LIMIT` is set.
* New: OTLP push when `OTEL_METRICS_EXPORTER` includes `otlp`; exemplars from sampled spans; and `http.server.request.duration`, `http.client.request.duration`, `rpc.server.call.duration` and `rpc.client.call.duration` in seconds with the recommended buckets.

#### Workflows (dapr/durabletask-go)

There are no OpenTelemetry semantic conventions for durable workflow engines, so this part aligns with the specification and the general naming rules:

* Remove `UnsafeSetSpanContext`, `ChangeSpanID` and `CancelSpan`. Intermediate replays do not start a recording span; activities are parented to a span context built from the persisted workflow span ID. On completion or continue-as-new, the workflow span is emitted once with its original start time. Its persisted span ID is supplied through an `sdktrace.IDGenerator` registered by daprd that reads a requested ID from the context, which is a supported SDK extension point.
* Propagate unsampled contexts so parent-based sampling holds for activities and child workflows.
* Set `error.type` from the failure details on failed workflows and activities.
* Name spans `orchestration {name}`, `activity {name}` and `timer`, with the version as an attribute instead of part of the name.
* Namespace span event attributes (`durabletask.event.name` instead of `name`).
* Link each continue-as-new generation's root span to the previous generation.
* Keep the `durabletask.*` attribute namespace, which is shared with the other Durable Task SDKs.

### Feature lifecycle outline

* **Expectations:** the trace changes ship together in one minor release, with release notes that list every renamed span and attribute and how to update queries. Metrics need no user action.
* **Compatibility:** Prometheus metric names, labels, types and buckets are unchanged apart from the bucket boundary behaviour above. `grpc-trace-bin` and the `traceparent` response header are kept. The old trace attributes are removed.
* **Feature flags:** none proposed. If maintainers prefer a transition period, the attribute changes can be put behind `OTEL_SEMCONV_STABILITY_OPT_IN` for one release (see Alternatives).

### Acceptance Criteria

* Unit tests for each convention: span kinds, names, attributes and status for HTTP and gRPC, outgoing client spans, component, messaging and workflow spans, W3C parsing and truncation, resource and `OTEL_*` handling.
* An integration test that runs two sidecars against an OTLP collector and checks every span kind, the CLIENT/SERVER parent link between the sidecars, and the resource.
* The golden Prometheus exposition test passes unchanged.
* No measurable increase in API latency at the default sampling rate. HTTP middleware allocations are compared with the existing benchmarks, which were also the motivation for [#7764](https://github.com/dapr/dapr/issues/7764).

## Completion Checklist

* Runtime: diagnostics, API servers, app channels, pub/sub, bindings, tracing setup and metrics exporter.
* durabletask-go: workflow span lifecycle and names; daprd registers the span ID generator.
* Tests: unit, integration and the golden metrics test.
* Documentation: tracing and metrics pages in dapr/docs, a migration note for trace queries, and the `OTEL_*` environment variables.
* SDKs: no changes required.
