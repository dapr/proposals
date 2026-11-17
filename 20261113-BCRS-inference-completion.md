# Proposal: Inference Completion Building Block - Replacing Conversation

- Author(s): Whit Waldo (@whitwaldo)
- State: Draft
- Updated: 2026-11-16

Media input shape proposed by @joshvanl and adapted/integrated into this proposal.

## Overview

This proposal replaces the preview **Conversation** building block with **Inference Completion**: a deliberately breaking, provider-neutral API for generating model responses. It replaces OpenAI-shaped messages/choices with ordered text and multimodal content, instructions, tool calls/results, and provider continuation state. Autoregressive, diffusion, and hybrid models share one request and an always-streamed response: callers receive incremental content or, when explicitly requested, a reference to a stored completion transcript. A model may emit generated-media references mid-stream without the caller having declared up front that media was expected. Applications supply their own history; see [Non-goals](#non-goals) for what the series deliberately leaves out.

This is not a short proposal because it builds on what we already have and expands even further (superseding [#108](https://github.com/dapr/proposals/pull/108) and [113](https://github.com/dapr/proposals/pull/113)): it defines the HTTP/gRPC APIs, typed streaming and artifact-read protocols, provider mappings, `completion.<provider>` components, SDK examples, acceptance criteria, and migration from `ConverseAlpha1` / `ConverseAlpha2`. It also establishes reusable runtime conventions: component model defaults and override controls, application-scoped model-access policies, resiliency, PII processing, observability, provider-native idempotency where supported, and cancellation propagated to provider work. Shared caching conventions support small final unary responses in adopting sibling APIs through general state or the proposed Centralized Cache integration; Completion itself has no cacheable operation and exposes no cache fields.

Completion anchors a coordinated but independently approvable AI/Inference series. Decisions, Embeddings, Reranking, Speech, Transcription, Image Generation, and Video Generation have separate proposals, provider components, configuration, and APIs; they adopt applicable runtime conventions by reference. This proposal does not introduce a universal Inference component or require approval or delivery of those other building blocks; they are split into separate proposals to be evaluated separately atop this foundational proposal.

Completion also supports provider-native asynchronous batches for independent, non-interactive requests such as evaluations, classification, and bulk content generation. Separate submission, status, cancellation, and streamed-result APIs preserve providers' batch execution and pricing opportunities without turning ordinary parallel completion calls into a purported discounted batch. Prices, supported models, and turnaround windows remain provider-defined.

Component-owned per-model routing settings let infrastructure constrain supported execution choices without accepting request-level routing filters. Purpose-specific discovery APIs let applications list models and query provider-reported pricing, capabilities, and deployment metadata through Dapr rather than installing provider SDKs. Each adapter defines its supported settings and discovery coverage; neither quantization filtering nor complete metadata is assumed across providers.

## Contents

| Section | What it covers |
| --- | --- |
| [Proposal scope and context](#proposal-scope-and-context) | Motivation, building-block boundaries, [non-goals](#non-goals), and independent approval |
| [Runtime API and execution](#runtime-api-and-execution) | Endpoints, delivery choices, [component metadata](#component-metadata-reference), [error codes](#error-code-reference), and provider integration |
| [Completion protocol reference](#completion-protocol-reference) | Wire schemas, content semantics, event ordering, transport, and transcript publication |
| [Provider-native completion batches](#provider-native-completion-batches) | Deferred execution, submission, per-item results, cancellation, and provider mappings |
| [Shared runtime extras](#shared-runtime-extras) | Model selection, access policies, resiliency, cancellation, artifacts, and cache conventions |
| [Model routing](#component-owned-model-routing) and [discovery](#model-discovery-and-pricing) | Per-model operator settings, model listing, pricing, and metadata retrieval |
| [Migration and delivery](#migration-and-delivery) | Conversation retirement, runtime acceptance criteria, and broad SDK work |
| [Illustrative SDK appendix](#illustrative-sdk-appendix) | Optional language examples for consuming the runtime contract |
| [Sibling operation reference](#sibling-operation-reference) | Job semantics adopted by image/video proposals, not Completion endpoints |
| [Related items](#related-items) | Prior proposals and supporting references |

The main reading path is runtime-focused: scope, execution, protocol, shared behavior, and delivery. Language examples are supplementary and collapsed by language; the sibling operation reference is only needed when reviewing an adopting job-based proposal.

## Proposal scope and context

### Coordinated proposal series

These colocated proposals are introduced together but reviewed, approved, implemented, and stabilized independently:

| Proposal | Purpose / component family | HTTP namespace | Relationship to this proposal |
| --- | --- | --- | --- |
| This proposal: Completion | Conversation replacement; `completion.<provider>` | `completioninference` | Foundational completion contract and reusable runtime extras |
| [Decisions](./20261113-BCRS-inference-decisions.md) | Typed probability/choice/score evaluation; `decisions.<provider>` | `decisioninference` | Independently adopts applicable extras |
| [Embeddings](./20261113-BCRS-inference-embeddings.md) | Text to dense vectors; `embeddings.<provider>` | `embeddinginference` | Independently adopts applicable extras |
| [Reranking](./20261113-BCRS-inference-reranking.md) | Query/document relevance; `reranking.<provider>` | `rerankinginference` | Independently adopts applicable extras |
| [Image Generation](./20261113-BCRS-inference-image-generation.md) | Generated images and their operations; `imagegeneration.<provider>` | `imagegenerationinference` | Independently adopts artifact and operation conventions |
| [Video Generation](./20261113-BCRS-inference-video-generation.md) | Generated video and its operations; `videogeneration.<provider>` | `videogenerationinference` | Independently adopts artifact and operation conventions |

HTTP namespaces concatenate the singular purpose name with `inference`, following the compound-name style of `binarystore`: `/v1.0-alpha1/{purpose}inference/{name}/...`. This makes the family visible in each route without introducing a shared `/inference` dispatcher. Component types, RPC names, SDK names, and access-policy operation names remain purpose-specific as defined in each proposal.

Approval of Completion does not approve any sibling building block, provider integration, or delivery commitment. Siblings reference the reusable extras here; image and video generation do not depend on one another. Shared schema/helper work is a technical dependency where adopted, not an all-or-nothing approval boundary. AI/Inference is a documentation grouping that can accept further purposes without reorganizing a universal API.

### Background and architectural boundary

Conversation's OpenAI-shaped messages/choices contract is too narrow even for multi-provider completion: providers differ in instruction handling, ordered content blocks, tool envelopes, signed continuation state, and incremental delivery. I propose replacing this contract while Conversation is still preview rather than adding provider-specific escape hatches to a stable API.

**Building-block boundaries follow application purpose, not provider identity or model architecture.** A completion model and an embedding model may be sold by the same provider without sharing a public contract or component configuration. Conversely, autoregressive and diffusion completion belong to the same building block because callers request the same purpose.

Implementations may reuse authentication, transports, provider libraries, or an internal dispatcher for omni-models. Such sharing must not leak into public protos, require an umbrella component, mutate another component's model/configuration, or couple cancellation and credentials across component instances.

#### Scope and alternatives

This proposal covers Completion's replacement API, provider mappings, request-bound streamed completion, provider-native asynchronous completion batches, generated-content artifact reads, per-model component settings, model discovery/pricing, runtime extras, SDKs, and Conversation migration.

- Keeping Conversation unchanged would preserve the provider-shaped contract and miss the preview migration window.
- A single Inference component with task-specific endpoints would reduce repeated configuration but require cross-purpose capability checks and bundle unrelated settings and approval decisions. The coordinated purpose-specific series is preferred.
- Endpoints divided by model architecture would expose implementation mechanics rather than application purpose.

Separate components may repeat credentials/endpoint references and use separate clients. Shared secret references and internal libraries can reduce duplication; explicit configuration isolation is an intentional tradeoff. Each proposal's provider list is illustrative, not a requirement to implement every provider before approval.

#### Non-goals

The following are deliberately excluded. Later sections rely on this list rather than restating each exclusion; where a section does mention one, it is because a reader is likely to assume otherwise at that point.

| Area | Excluded from this proposal | Where the need is met instead |
| --- | --- | --- |
| Sibling purposes | Decisions, embeddings, reranking, dedicated image/video generation, vector storage, and training | Separate proposals in the series |
| Universal API | A shared `/inference` dispatcher, umbrella Inference component, public `task` discriminator, or family-wide capability manifest | Purpose-specific routes, registries, and component types |
| Conversation state | Runtime-owned conversation history, session creation/resumption, cache-backed session state, or provider-held `context_id` sessions | Callers supply full history per invocation |
| Unary completion | A public unary `Complete` endpoint or a `stream` query selector | Always-streamed `CompleteAlpha1`; SDK fold helpers materialize locally on request |
| Response caching for Completion | Cache reads/writes for any Completion operation, cache-only buffers, cached transcripts, or cache-backed job tracking | Shared cache contract applies to eligible sibling unary operations only |
| Runtime tool execution | The sidecar invoking a tool on the application's behalf, running an agent loop, or scheduling the dependent follow-up turn. Tool *calling* is fully in scope and matches Conversation Alpha2: tool declarations, `tool_choice`, streamed `tool_call` blocks, `tool_result` history blocks, and the `TOOL_RESULTS_REQUIRED` outcome | As in Conversation Alpha2, the application executes the call the model emits and submits the result as a `tool_result` block in the next request |
| Job orchestration | A durable Dapr scheduler, background poller, or application worker for batches | Provider-owned native batch jobs; applications or Workflows poll |
| Interactive batch emulation | Implementing batches by fanning out ordinary completion calls | Provider-native batch endpoints only; unsupported integrations fail explicitly |
| Model inventory | A Dapr-maintained or release-bound model/capability catalog used for preflight | Provider validation at invocation; optional operator allowlists and adapter-documented restrictions |
| Request-level routing | Caller-supplied routing filters, quantization preferences, or provider parameter injection via `metadata` | Component-owned `modelRouting` |
| Artifact lifecycle | Artifact expiry/TTL fields, expiry errors, retention guarantees, or a Binary Store expiry feature | Operator-owned storage policy |
| Resumption | Stream resumption via `sequence`/`Last-Event-ID`, byte-range reads, or automatic reconnect | Re-issue the request or read |
| Cross-app invocation | Cross-app inference routing, App ID header trust, or Workflow-style self-call bypass in access policies | Separate future design if needed |
| Real-time media | Bidirectional live audio/video sessions | Out of scope for the series |
| Guarantees Dapr cannot make | Deterministic output, exactly-once execution/billing, zero-cost cancellation, batch discounts, or price ceilings | Provider-defined; documented per adapter |

## Runtime API and execution

This section introduces what the runtime exposes and how it connects to providers. The complete message definitions and transport invariants are collected in the [protocol reference](#completion-protocol-reference).

### Completion API

| HTTP endpoint | gRPC RPC | Result |
| --- | --- | --- |
| `POST /v1.0-alpha1/completioninference/{name}/complete` | `CompleteAlpha1` | Typed event stream, including a terminal transcript reference when requested |
| `GET /v1.0-alpha1/completioninference/{name}/artifacts/{artifactId}` | `ReadCompletionArtifactAlpha1` | Incremental artifact bytes |
| `POST /v1.0-alpha1/completioninference/{name}/batches` | `SubmitCompletionBatchAlpha1` | Accepted provider-native batch handle and status |
| `GET /v1.0-alpha1/completioninference/{name}/batches/{batchId}` | `GetCompletionBatchAlpha1` | Current batch status and available counts |
| `DELETE /v1.0-alpha1/completioninference/{name}/batches/{batchId}` | `CancelCompletionBatchAlpha1` | Native cancellation request acknowledgment |
| `GET /v1.0-alpha1/completioninference/{name}/batches/{batchId}/results` | `ReadCompletionBatchResultsAlpha1` | Correlated final item results as an event stream |
| `GET /v1.0-alpha1/completioninference/{name}/models` | `ListCompletionModelsAlpha1` | Authorized model catalog page |
| `GET /v1.0-alpha1/completioninference/{name}/models/metadata?model={model}` | `GetCompletionModelMetadataAlpha1` | Model pricing, capabilities, and deployment metadata |

The route/RPC selects the Completion registry; `name` resolves a `completion.*` component only. A component of a different building-block type is rejected locally with `ERR_INFERENCE_COMPONENT_NOT_FOUND`, never routed by provider identity or inferred model capability.

```protobuf
// In dapr.proto; completion.proto and inference_common.proto supply the types.
rpc CompleteAlpha1(InferenceCompleteRequestAlpha1) returns (stream InferenceCompleteStreamEventAlpha1) {}
rpc ReadCompletionArtifactAlpha1(InferenceReadArtifactRequestAlpha1) returns (stream InferenceReadArtifactResponseAlpha1) {}
rpc SubmitCompletionBatchAlpha1(stream InferenceSubmitCompletionBatchRequestAlpha1) returns (InferenceCompletionBatchAlpha1) {}
rpc GetCompletionBatchAlpha1(InferenceCompletionBatchRequestAlpha1) returns (InferenceCompletionBatchAlpha1) {}
rpc CancelCompletionBatchAlpha1(InferenceCompletionBatchRequestAlpha1) returns (google.protobuf.Empty) {}
rpc ReadCompletionBatchResultsAlpha1(InferenceCompletionBatchRequestAlpha1) returns (stream InferenceCompletionBatchResultEventAlpha1) {}
rpc ListCompletionModelsAlpha1(InferenceListModelsRequestAlpha1) returns (InferenceListModelsResponseAlpha1) {}
rpc GetCompletionModelMetadataAlpha1(InferenceGetModelMetadataRequestAlpha1) returns (InferenceModelMetadataAlpha1) {}
```

### Error code reference

Completion introduces a new `ERR_INFERENCE_*` code family and does not reuse Conversation's `ERR_CONVERSATION_*` codes; this is part of the intentional break. Codes are shared with the sibling proposals so that SDK error handling is uniform across the series. Every code is returned as a standard Dapr error (HTTP body `errorCode`/`message`, gRPC status plus `ErrorInfo` reason) before a stream opens, or through the post-open stream error path described under [HTTP and runtime behavior](#http-and-runtime-behavior).

Status mapping follows two rules:

- **Operation-level unsupported → `501` / `Unimplemented`.** The integration lacks the entire operation (batches, discovery, remote cancellation). The caller cannot change the request to succeed.
- **Option-level unsupported → `400` / `InvalidArgument`.** The operation exists but a specific request option (idempotency key, structured output, media input, a tool, a model override) is unsupported by this component, configuration, or model. The caller can succeed by changing the request.

Upstream data that the runtime cannot trust maps to `502` / `Internal` rather than `DataLoss`: `Internal` signals a broken invariant in a dependency, whereas `DataLoss` conventionally signals unrecoverable corruption of the caller's own data.

| Code | HTTP | gRPC | Raised when |
| --- | --- | --- | --- |
| `ERR_INFERENCE_INVALID_REQUEST` | 400 | `InvalidArgument` | Request shape/validation failure not covered by a more specific code: malformed history, duplicate block/call IDs, unmatched tool results, schema supplied with a non-schema mode, unknown enum values, conflicting path/body `name`, empty/whitespace model, `stream` query parameter supplied |
| `ERR_INFERENCE_FEATURE_UNSUPPORTED` | 400 | `InvalidArgument` | A locally known unsupported option for this component/model: tools, structured output, media input/output, partial tool calls, `ARTIFACT` delivery without a configured `artifactStore`, provider-rejected unsupported option mapped by the adapter |
| `ERR_INFERENCE_IDEMPOTENCY_UNSUPPORTED` | 400 | `InvalidArgument` | `idempotency_key` supplied to an integration/operation that cannot forward it |
| `ERR_INFERENCE_MODEL_OVERRIDE_DISABLED` | 400 | `InvalidArgument` | Request model differs from `defaultModel` while `disableModelOverride = true` |
| `ERR_INFERENCE_MODEL_NOT_CONFIGURED` | 400 | `InvalidArgument` | Effective model is not a key in a present `modelRouting` object |
| `ERR_INFERENCE_PROVIDER_STATE_INCOMPATIBLE` | 400 | `InvalidArgument` | Supplied `provider_state`/`opaque_content` is bound to another provider/format or is rejected by the provider as invalid continuation data |
| `ERR_INFERENCE_DISCOVERY_PAGE_TOKEN_INVALID` | 400 | `InvalidArgument` | Expired, tampered, or cross-scope discovery `page_token` |
| `ERR_INFERENCE_ACCESS_DENIED` | 403 | `PermissionDenied` | An applicable access policy does not grant the component/operation/model combination, including artifact inputs |
| `ERR_INFERENCE_COMPONENT_NOT_FOUND` | 404 | `NotFound` | `name` does not resolve to a component of the invoking building block's type (`completion.*` here; a component of another building block type is also reported this way) |
| `ERR_INFERENCE_MODEL_NOT_FOUND` | 404 | `NotFound` | Provider reports that the requested model does not exist (invocation or metadata lookup) |
| `ERR_INFERENCE_ARTIFACT_NOT_FOUND` | 404 | `NotFound` | Unknown, unauthorized-to-reveal, or no-longer-available artifact ID |
| `ERR_INFERENCE_BATCH_NOT_FOUND` | 404 | `NotFound` | Unknown batch ID, or a batch bound to another application/component |
| `ERR_INFERENCE_OPERATION_NOT_FOUND` | 404 | `NotFound` | Unknown operation ID, an operation bound to another application/component, or any lookup against a synchronous-only provider that issues no handles (media siblings) |
| `ERR_INFERENCE_BATCH_RESULTS_UNAVAILABLE` | 404 | `NotFound` | Terminal batch whose results the provider no longer retains |
| `ERR_INFERENCE_BATCH_RESULTS_NOT_READY` | 409 | `FailedPrecondition` | Results requested for a non-terminal batch |
| `ERR_INFERENCE_BATCH_NOT_CANCELABLE` | 409 | `FailedPrecondition` | Cancellation requested for an already-terminal batch |
| `ERR_INFERENCE_OPERATION_NOT_CANCELABLE` | 409 | `FailedPrecondition` | Cancellation requested for an already-terminal operation (media siblings) |
| `ERR_INFERENCE_IDEMPOTENCY_CONFLICT` | 409 | `Aborted` | Provider reports the idempotency key was reused with a different payload |
| `ERR_INFERENCE_RATE_LIMITED` | 429 | `ResourceExhausted` | Provider throttling/quota, including batch quota, after any configured retries |
| `ERR_INFERENCE_CANCELED` | 499 | `Canceled` | Caller cancellation observed before successful completion (gRPC only in practice; HTTP clients see a closed connection) |
| `ERR_INFERENCE_PROVIDER_ERROR` | 500 | `Internal` | Provider returned an error the adapter cannot map more specifically; message carries the sanitized provider reason |
| `ERR_INFERENCE_PROTOCOL_VIOLATION` | 500 | `Internal` | Adapter/component emitted an invalid event sequence (undeclared revision, missing finalization, mismatched final value, unknown stop reason, malformed final tool arguments) |
| `ERR_INFERENCE_STORAGE_FAILED` | 500 | `Internal` | Artifact/transcript/staging/manifest store write or read failed |
| `ERR_INFERENCE_BATCH_UNSUPPORTED` | 501 | `Unimplemented` | Component does not implement the native batch interface |
| `ERR_INFERENCE_DISCOVERY_UNSUPPORTED` | 501 | `Unimplemented` | Component does not implement model listing or metadata lookup |
| `ERR_INFERENCE_CANCELLATION_UNSUPPORTED` | 501 | `Unimplemented` | Provider offers no remote job-cancellation mechanism |
| `ERR_INFERENCE_METADATA_INVALID` | 502 | `Internal` | Provider catalog/metadata record fails validation (missing IDs, duplicate IDs, invalid price/decimal, non-positive context window) |
| `ERR_INFERENCE_RESPONSE_INVALID` | 502 | `Internal` | A unary provider response fails the building block's result validation (sibling proposals: probabilities out of range or not summing to one, missing/duplicate vector or result indices, dimension mismatch, descriptor without an artifact ID). Completion's streaming equivalent is `ERR_INFERENCE_PROTOCOL_VIOLATION` |
| `ERR_INFERENCE_BATCH_RESULTS_INCOMPLETE` | 502 | `Internal` | Provider result set lacks an authoritative disposition for a submitted item, or result framing is corrupt |
| `ERR_INFERENCE_PROVIDER_UNAVAILABLE` | 503 | `Unavailable` | Provider unreachable/timed out after any configured retries |
| `ERR_INFERENCE_DEADLINE_EXCEEDED` | 504 | `DeadlineExceeded` | Request deadline or outbound timeout elapsed before `completed` |

Configuration errors (invalid `modelRouting`, missing required store references, cache settings on Completion, malformed access policies) are surfaced at component/policy load time through existing Dapr configuration error reporting, not through these runtime codes. Adapters MUST map native errors to this table and MUST NOT invent additional public codes; an unmapped native error uses `ERR_INFERENCE_PROVIDER_ERROR` with the provider's sanitized reason in `message`.

### Completion contract

`CompleteAlpha1` generates the next model turn from instructions and ordered history. This is an intentionally breaking contract, not a rename of `AIMessage`, `ConversationTools`, or `outputs[].choices[]`. One invocation produces one generation containing ordered turns and content blocks. Each `CompleteAlpha1` call returns a stream through the same RPC/HTTP route. The request selects content-event delivery or explicit persistence of the entire completion transcript; it does not need to predict whether the model will generate media. Independent deferred requests use the explicit [batch APIs](#provider-native-completion-batches).

| Request delivery | Stream result | Whole-response assembly |
| --- | --- | --- |
| `EVENTS` (default) | Lifecycle/content events, including generated-media references where produced | None on the native incremental path; SDK folding is opt-in |
| `ARTIFACT` | Lifecycle events ending in a published transcript reference in `completed` | None on the native incremental path; writes each content event to configured storage |

The event pipeline is the primary execution path for streaming-capable providers. Neither runtime nor SDK may construct a complete response merely to serve a stream or publish an artifact. A caller may explicitly fold content events into a local result. Full-response processing exceptions and unary-only provider fallback are specified below; a streamed transport is not an unconditional constant-memory guarantee.

A reference-only result still uses the stream envelope, not a different return type: `started`, optional `usage`, then `completed` carrying the transcript reference. The reference is effectively its single result payload; lifecycle events and successful transport termination remain mandatory. In default `EVENTS` delivery, text, tool calls, and generated-media references may be mixed and are announced as they become known. No initial event promises a complete artifact inventory or a separate live-stream URL.

The [protocol reference](#completion-protocol-reference) defines the wire contract. Completion-specific types live in `completion.proto`; reusable value types have the ownership specified under [shared wire types](#shared-wire-types). The `Inference` prefix identifies the AI/Inference family for both purpose-specific and shared wire types.

### Runtime execution path

```text
Request -> resolve component/model/options and authorize -> provider adapter
        -> normalize and process content events
        -> EVENTS: stream content events to the caller
           ARTIFACT: write transcript, then stream its published reference
        -> terminal completion and successful transport termination
```

Configuration and known restrictions are checked before provider work. Both delivery paths share validation, cancellation, and failure handling.

### Completion component and configuration

Each `completion.<provider>` component has its own credentials, endpoint, default model, resiliency, access policy, and optional artifact-store binding. Neither interactive completion nor batch control/results use response caching. It cannot be invoked as an embedding, decision, reranking, image-generation, or video-generation component even if its internal provider client supports those operations. Here's an example in Go:

```go
package completion

type Component interface {
    metadata.ComponentWithMetadata
    Init(ctx context.Context, meta Metadata) error
    io.Closer
}

type Completer interface {
    Complete(ctx context.Context, req *CompleteRequest) (*CompleteResponse, error)
}

type CompletionStreamer interface {
    CompleteStream(ctx context.Context, req *CompleteRequest, opts StreamOptions, out chan<- StreamChunk) error
}

type StreamOptions struct {
    ContentMode  ContentMode
    IncludeUsage bool
}

// StreamChunk has exactly one payload; indices address the typed response.
type StreamChunk struct {
    TurnIndex      uint32
    BlockIndex     uint32
    TurnStart      *TurnStart
    BlockStart     *BlockStart
    BlockDelta     *BlockDelta
    BlockCompleted *BlockCompleted
    TurnCompleted  *TurnCompleted
    Usage          *Usage
    Finish         *CompletionFinish
}
```

The Go request/result/content types mirror the common proto's semantics without importing runtime-generated proto types into components-contrib. Model-invoking component requests carry the resolved model, operator-owned routing snapshot, resolved `scrub_pii` decision, forwardable request metadata (see [`forwardRequestMetadata`](#component-metadata-reference)), and optional native idempotency key. The routing snapshot is internal, not an application request field. Component `CompleteRequest` also excludes `delivery` and transport `stream_options`; the runtime selects the event sink, resolves `StreamOptions`, and owns `partial_tool_calls` buffering. `TurnStart` supplies the role, `BlockStart` the stable ID/kind and source `Revisable` declaration, `BlockDelta` the text update or argument fragment, and `CompletionFinish` the outcome/model. `BlockCompleted` mirrors the marker/state/final-value proto and MUST NOT contain accumulated text or an entire `ContentBlock`. Adapters must not accumulate final text merely to manufacture completion markers; any native final-text field is not copied into the event. The runtime assigns response IDs and public sequence numbers and owns global `started` / `completed` events, including delivery metadata, the effective model, and transcript references. Unary-provider fallback receives the same runtime response-ID treatment and emits the same public stream contract.

`CompleteStream` MUST NOT close `out`; the runtime owns it. Every send must respect cancellation, and the component must bind the same invocation cancellation to the provider's HTTP/gRPC/SDK operation, not just its output channel. If the provider requires explicit job cancellation, the adapter also invokes that cancellation path under the finite cleanup context specified by [end-to-end cancellation](#end-to-end-cancellation). Successful return requires all blocks/turns to have completed and one `Finish`; a non-nil error or missing finalization fails the stream even if chunks were produced. The runtime does not emit public `completed` until the component returns successfully, the queue drains, and response processing succeeds; artifact delivery additionally requires successful publication. Provider-specific events must be normalized by the adapter, not parsed in HTTP/gRPC handlers. Non-text final values, state, and optional usage reuse normalized result types without forcing text assembly. Live transport and the incremental artifact writer are runtime-owned consumers of this same pipeline; SDK folding is a separate, explicit client operation.

Each Completion component implements at least one of `Completer`, `CompletionStreamer`, or the [native batch interface](#batch-component-boundary). These distinguish execution modes within one purpose; a batch-only component rejects interactive calls with `ERR_INFERENCE_FEATURE_UNSUPPORTED`. The runtime replays a unary-only provider response as `BUFFERED` events through either interactive delivery mode. Optional `ModelLister` and `ModelMetadataGetter` component interfaces implement [model discovery](#model-discovery-and-pricing) within this registry. Inference remains usable when discovery is unsupported.

The runtime validates request shape, component configuration, model-override policy, model-access authorization, and invariants it actually knows. Adapters may reject known unsupported operations/options before a network call; unknown per-model restrictions are evaluated by the provider at invocation time and returned as explicit mapped Dapr errors. A locally known unsupported feature uses `ERR_INFERENCE_FEATURE_UNSUPPORTED` (HTTP `400` / gRPC `InvalidArgument`). Do not silently drop options, infer tool support from a provider name, or emulate a strict schema with a prompt. Native `complete_stream` remains distinct from unary fallback; neither model name nor a Completion interface proves that a particular model accepts tools or structured output.

Cancellation behavior is documented per adapter/provider integration and actual outcomes are recorded at runtime, without requiring a per-model manifest. Distinguish provider-confirmed cancellation, documented cancel-on-disconnect behavior, and best-effort transport abort; a cancellable local SDK method is not proof of stopped computation. Likewise, artifact-backed completion requires the configured store and runtime reader, not a new provider interface or model catalog. Native prompt-cache controls and provider-specific tuning remain documented component configuration.

#### Completion configuration example

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: chat
spec:
  type: completion.openai
  version: v1
  metadata:
    - name: key
      secretKeyRef:
        name: openai-secrets
        key: apiKey
    - name: defaultModel
      value: "gpt-4o"
    - name: disableModelOverride
      value: "false"
    - name: modelRouting
      value: '{"gpt-4o": {}, "gpt-4o-mini": {}}'
    - name: artifactStore
      value: "transcripts"
    - name: batchStore
      value: "completion-batches"
```

Interactive `EVENTS` delivery needs only provider credentials. `ARTIFACT` delivery additionally requires `artifactStore`; native batches additionally require `batchStore`. The shared cache settings (`cacheStore`, `cacheEnabled`, `cacheNamespace`) belong to adopting siblings; on a `completion.*` component they are configuration errors. There are no per-task defaults such as `embedModel`: each component already has one purpose.

#### Component metadata reference

Metadata keys are either **runtime-owned** (identical name, type, and semantics on every `completion.*` component, validated by the runtime) or **provider-defined** (declared by each adapter, passed through to its client, and documented in that component's reference page). Runtime-owned keys are never forwarded to providers. Adapters MUST NOT declare provider-defined keys that collide with runtime-owned names.

| Key | Owner | Required | Type / shape | Meaning |
| --- | --- | --- | --- | --- |
| `defaultModel` | Runtime | Optional (required when `disableModelOverride = true`) | Provider model ID, exact case | Model used when a request omits `model` |
| `disableModelOverride` | Runtime | Optional, default `false` | `bool` | Reject request models other than `defaultModel` |
| `modelRouting` | Runtime key, provider-defined values | Optional | JSON object keyed by exact model ID; each value is an adapter-defined JSON object | Restricts selectable models and supplies per-model routing settings; see [component-owned model routing](#component-owned-model-routing) |
| `artifactStore` | Runtime | Required for `ARTIFACT` delivery and for providers returning inline generated media | Name of a Binary Store (or other supported artifact-capable) component in the same namespace | Backing store for completion transcripts and persisted generated media |
| `batchStore` | Runtime | Required for native batches | Name of a state store component | Durable batch bindings and submission manifests; also used for input staging unless `batchStagingStore` is set |
| `batchStagingStore` | Runtime | Optional | Name of a Binary Store component | Separate store for staged normalized batch input when the provider requires file upload and `batchStore` is unsuitable for large values |
| `scrubPii` | Runtime | Optional, default `false` | `bool` | Default for requests that omit `scrub_pii`; an explicit request value overrides it |
| `forwardRequestMetadata` | Runtime | Optional, default `false` | `bool` | When `true`, request `metadata` is passed to the adapter for mapping onto the provider's native tagging field (for example OpenAI `metadata`/`user`); when `false`, it is used only in Dapr tracing attributes |
| `keepAliveInterval` | Runtime | Optional, default `15s` | Go duration | SSE comment interval while a response is open |
| `cleanupTimeout` | Runtime | Optional, default `30s` | Go duration | Deadline for the supervised cleanup context used by [end-to-end cancellation](#end-to-end-cancellation) for native cancellation and staging cleanup |
| `cacheStore`, `cacheEnabled`, `cacheNamespace`, `responseCacheTTL` | Runtime | Rejected | — | Configuration error on `completion.*`; the first three apply to sibling components only, the last is the legacy Conversation key |
| `key`, `endpoint`, `apiVersion`, `region`, `organization`, `project`, `timeout`, native prompt-cache or safety settings, and similar | Provider-defined | Per adapter | Per adapter | Credentials, service location, and provider tuning. Shape and defaults differ between adapters; secrets use `secretKeyRef` |

Provider-defined keys may use any names not reserved above. Adapters document which runtime-owned optional features they support (`ARTIFACT` delivery, native batches, discovery, idempotency forwarding, metadata forwarding, remote cancellation) so operators can tell before deployment which store references are meaningful.

The optional `CompletionAccessPolicy` resource below governs which models and operations applications may use on `chat`. It is enforced by the runtime before this component is invoked, not forwarded as a provider parameter. The example's `modelRouting` restricts new calls/discovery to two exact model IDs with no provider-specific filters. A component's model boundary, override lock, and access policy are independent restrictions; none overrides another. Omit `modelRouting` to retain unrestricted dynamic selection subject to the other controls.

Initial candidate adapters include OpenAI, Anthropic, Google AI, Bedrock, Ollama, Cohere, and a conformance/test provider. The [provider translation table](#provider-translation-and-diffusion-adapter-boundary) describes intended mappings, not a model catalog or a promise to ship all adapters together.

### Provider translation and diffusion adapter boundary

| Source | Common mapping | Adapter responsibility |
| --- | --- | --- |
| OpenAI Chat Completions | Assistant text and calls become ordered model blocks; text/argument fragments append | Translate native role/tool envelopes; do not synthesize multiple candidates when only one generation is requested |
| OpenAI Responses | Message, function-call, and reasoning items become ordered blocks/turns | Preserve item order and opaque continuation state; map native completion/failure explicitly |
| Anthropic Messages | Content-block starts/deltas/stops map to block events; `tool_use` becomes a call | Put instructions in `system`, tool results in native user content, preserve signed thinking state, and combine usage updates once |
| Gemini / Bedrock / Ollama | Native parts or content blocks become typed blocks, with complete calls where fragments are unavailable | Adapt each endpoint's native envelopes, signatures, and finish/usage events; do not require OpenAI compatibility |
| Proposed text-diffusion adapter | Full decoded text states become `REPLACE` updates for a stable text block, followed by the final decoded text | Explicitly declare revisability, exclude the prompt, retain unresolved-position representation in previews, and declare only actually supported features |
| Unary-only provider | Final response replayed as `BUFFERED` blocks | Supply the same typed result, without pretending it streamed natively |

The [LLaDA inference code](https://github.com/ML-GSAI/LLaDA/blob/main/generate.py), as discussed in dapr/proposals#113, motivates the snapshot adapter contract. The [demo](https://github.com/ML-GSAI/LLaDA/blob/main/app.py) in that repository collects denoising states before replaying them; it is not evidence of a hosted live-streaming API. Implementing an adapter that emits states during inference is separate work. Neither tool support nor token usage is assumed for that model. Until such an adapter exists, synthetic component fixtures exercise the same boundary.

Illustrative updates for one text block (SDK enum abbreviations, not literal SSE JSON):

```text
APPEND source:    APPEND("Dapr is ") -> APPEND("a runtime") -> block_completed()
Snapshot source:  REPLACE("Dapr __ a ____") -> REPLACE("Dapr is a runtime") -> block_completed()
Append client
with snapshots:  [buffer latest snapshot] -> APPEND("Dapr is a runtime") -> block_completed()
Empty snapshot:  REPLACE("draft") -> REPLACE("") -> block_completed()
```

SDKs expose one typed async `Complete` event iterator for both delivery modes, plus explicit artifact-read and fold helpers. Folding an `EVENTS` stream returns `InferenceCompleteResponseAlpha1`; an `ARTIFACT` stream yields a reference that the application explicitly reads before folding its content events. Iteration retains no completed text/block history unless the caller chooses to retain it. Applications render updates by turn/block, appending or replacing as directed, and never execute tools from previews. The same request, stream envelope, and content renderer work with autoregressive, diffusion, and hybrid serving implementations.

## Completion protocol reference

This section is the detailed runtime/adapter contract. Start with the [request and content model](#common-request-and-result), then [stream negotiation](#streaming-and-delivery-negotiation), [event grammar](#event-grammar-and-folding-normative), [HTTP behavior](#http-and-runtime-behavior), and [artifact-backed completion](#artifact-backed-completion).

### Common request and result

```protobuf
enum InferenceCompleteDeliveryModeAlpha1 {
  INFERENCE_COMPLETE_DELIVERY_MODE_ALPHA1_UNSPECIFIED = 0;
  INFERENCE_COMPLETE_DELIVERY_MODE_ALPHA1_EVENTS = 1;
  INFERENCE_COMPLETE_DELIVERY_MODE_ALPHA1_ARTIFACT = 2;
}

message InferenceCompleteRequestAlpha1 {
  string name = 1;
  repeated string instructions = 2;
  repeated InferenceTurnAlpha1 history = 3;
  repeated InferenceToolAlpha1 tools = 4;
  InferenceToolChoiceAlpha1 tool_choice = 5;
  InferenceOutputConstraintAlpha1 output_constraint = 6;
  optional double temperature = 7;
  optional int64 max_output_tokens = 8;
  optional bool scrub_pii = 9;
  map<string, string> metadata = 10;
  InferenceCompleteStreamOptionsAlpha1 stream_options = 11;
  optional string model = 12;
  optional string idempotency_key = 13;
  InferenceCompleteDeliveryModeAlpha1 delivery = 14;
}

message InferenceTurnAlpha1 {
  enum TurnRole {
    TURN_ROLE_UNSPECIFIED = 0;
    USER = 1;
    MODEL = 2;
    TOOL = 3;
  }
  TurnRole role = 1;
  repeated InferenceContentBlockAlpha1 content = 2;
  repeated InferenceProviderStateAlpha1 provider_state = 3;
}

message InferenceContentBlockAlpha1 {
  string id = 1;
  oneof kind {
    string text = 2;
    InferenceMediaAlpha1 media = 3;
    InferenceToolCallAlpha1 tool_call = 4;
    InferenceToolResultAlpha1 tool_result = 5;
    string reasoning_summary = 6;
    InferenceProviderStateAlpha1 opaque_content = 7;
  }
  repeated InferenceProviderStateAlpha1 provider_state = 8;
}

message InferenceMediaAlpha1 {
  string mime_type = 1;
  oneof source {
    bytes data = 2;
    string uri = 3;
    InferenceProviderStateAlpha1 provider_reference = 4;
    InferenceArtifactReferenceAlpha1 artifact_reference = 5;
  }
}

message InferenceToolAlpha1 {
  string name = 1;
  optional string description = 2;
  google.protobuf.Struct input_schema = 3;
}

message InferenceToolChoiceAlpha1 {
  enum Mode {
    MODE_UNSPECIFIED = 0;
    AUTO = 1;
    NONE = 2;
    REQUIRED = 3;
  }
  oneof selection {
    Mode mode = 1;
    string named_tool = 2;
  }
}

message InferenceToolCallAlpha1 {
  string call_id = 1;
  string name = 2;
  google.protobuf.Struct arguments = 3;
}

message InferenceToolResultAlpha1 {
  string call_id = 1;
  repeated InferenceToolResultContentAlpha1 content = 2;
  bool is_error = 3;
}

message InferenceToolResultContentAlpha1 {
  oneof kind {
    string text = 1;
    google.protobuf.Value json = 2;
    InferenceMediaAlpha1 media = 3;
  }
}

message InferenceProviderStateAlpha1 {
  string provider = 1;
  string format = 2;
  string compatibility_key = 3;
  bytes payload = 4;
}

message InferenceOutputConstraintAlpha1 {
  enum Mode {
    MODE_UNSPECIFIED = 0;
    TEXT = 1;
    JSON = 2;
    JSON_SCHEMA = 3;
  }
  Mode mode = 1;
  google.protobuf.Struct schema = 2;
}

// Canonical shape produced by SDK fold helpers and used internally for
// unary-only provider fallback. Defined here so every SDK folds to the same
// structure; it is never the request or response type of a public RPC.
message InferenceCompleteResponseAlpha1 {
  string id = 1;
  repeated InferenceTurnAlpha1 output = 2;
  InferenceCompletionOutcomeAlpha1 outcome = 3;
  optional string model = 4;
  InferenceUsageAlpha1 usage = 5;
}

message InferenceCompleteArtifactAlpha1 {
  enum Format {
    FORMAT_UNSPECIFIED = 0;
    COMPLETION_EVENTS_JSONL_V1ALPHA1 = 1;
  }
  InferenceArtifactAlpha1 artifact = 1;
  Format format = 2;
}

enum InferenceCompletionOutcomeAlpha1 {
  INFERENCE_COMPLETION_OUTCOME_ALPHA1_UNSPECIFIED = 0;
  INFERENCE_COMPLETION_OUTCOME_ALPHA1_COMPLETED = 1;
  INFERENCE_COMPLETION_OUTCOME_ALPHA1_TOOL_RESULTS_REQUIRED = 2;
  INFERENCE_COMPLETION_OUTCOME_ALPHA1_OUTPUT_LIMIT = 3;
  INFERENCE_COMPLETION_OUTCOME_ALPHA1_REFUSED = 4;
  INFERENCE_COMPLETION_OUTCOME_ALPHA1_FILTERED = 5;
}

message InferenceUsageAlpha1 {
  optional uint64 input_tokens = 1;
  optional uint64 output_tokens = 2;
  optional uint64 total_tokens = 3;
  optional uint64 cache_read_tokens = 4;
  optional uint64 cache_write_tokens = 5;
  optional uint64 reasoning_tokens = 6;
}
```

Instructions have one application-level scope and preserve their listed order. Adapters map them into native instruction fields; a separate system/developer hierarchy is not silently collapsed into this scope. `role` describes a turn's participant. User turns carry user content, model turns can interleave text and calls, and tool turns contain correlated results. Adapters wrap tool results in the native role required by their provider without changing their meaning. Output turns use `MODEL`.

Block IDs are nonempty and unique within a turn. Output IDs are adapter-assigned when the provider supplies none. Call IDs identify calls across the history and results, independently of block IDs; duplicate calls, unmatched results, and unsupported role/content combinations are validation errors. Completed output can be appended to history unchanged. Provider adapters preserve ordering and group adjacent items only where native turn semantics permit it; they must not flatten mixed content into a text prompt.

Tool calling works the same way as in Conversation Alpha2 and is a supported part of the contract. `tools` declares the callable functions (`ConversationTools` / `ConversationToolsFunction`), `tool_choice` controls selection (`ConversationRequestAlpha2.tool_choice`, with `AUTO` / `NONE` / `REQUIRED` / named-tool as typed options instead of a free-form string), a model turn emits `tool_call` blocks (`ConversationToolCalls` on the assistant result), and the application returns each result as a `tool_result` block in a `TOOL` turn (`ConversationMessageOfTool`). Differences from Conversation: arguments are a parsed `Struct` rather than a JSON string, so malformed arguments fail before the application sees them; results may carry `is_error` and typed text/JSON/media content; and a generation that ends awaiting results reports the distinct `TOOL_RESULTS_REQUIRED` outcome rather than a `finish_reason` string. As with Conversation, the sidecar never invokes the tool; the application executes it and issues the next request (see [Non-goals](#non-goals)).

Inline media bytes remain an input convenience subject to existing request limits, not an output transport. Generated media blocks use `artifact_reference` and the byte-read API in [artifact transfer](#artifact-transfer); `block_completed` carries the reference, never the file. Adapters returning media must support its retrieval through Completion's artifact path; a per-model support manifest is not required. Large input files use references rather than enlarging transport messages.

Opaque state preserves signed reasoning blocks, thought signatures, provider file references, and other required continuation data at its original scope and position. It is not user-visible reasoning text or application metadata. Two carriers exist, and adapters choose by scope:

- **`provider_state` attachments** (on a turn or a content block) carry state *about* visible content the application can already see, such as a signature over a text block or a turn-level cache marker. The attached content remains renderable; the state rides along when the turn is echoed back.
- **An `opaque_content` block** is used when the provider returns an item that has *no* visible counterpart, such as an encrypted reasoning item that must occupy an ordered position in the next request. It is a block in its own right so order is preserved, but it has nothing to render.

Both use `InferenceProviderStateAlpha1`. Adapters validate its known provider binding/format before invocation and reject mismatches with `ERR_INFERENCE_PROVIDER_STATE_INCOMPATIBLE`; compatibility requiring provider/model knowledge may be checked upstream. Editing associated content may invalidate state; provider rejection remains explicit. Dapr does not scrub or log opaque payloads by default.

`scrub_pii` uses presence semantics: an explicit value wins; when absent, the component's `scrubPii` metadata (default `false`) applies. The resolved decision is fixed for the invocation and recorded in the trace.

`metadata` is a bounded set of application tags. It is always attached to Dapr tracing attributes for the invocation. It is forwarded to the provider only when the component sets `forwardRequestMetadata = true`, in which case the adapter maps it to the provider's native tagging field and documents any key/value constraints; keys the provider cannot accept fail validation rather than being dropped. It is never interpreted as generation parameters or component configuration.

Options use presence semantics: absent temperature/token limits select provider defaults; explicit unsupported values fail. Absent/unspecified tool selection means `AUTO` when tools are supplied, otherwise `NONE`; required/named selection without a matching declared tool fails validation. An absent/unspecified output constraint means text; `JSON` and `JSON_SCHEMA` require native constrained generation, not prompt-only emulation. Schema mode requires a schema and rejects unsupported schema features; supplying a schema with another mode is invalid. On a `COMPLETED` structured response, concatenate text blocks in order to obtain the single JSON document and validate it; refusals, filtering, tool requests, and output limits are distinct outcomes and need not contain that document. An unknown native stop reason fails normalization rather than becoming a successful completion. Partial or malformed final tool arguments are explicit errors, including when an output limit truncated them.

Token counters are optional, authoritative provider measurements, never inferred from event count or diffusion steps. Missing counters remain absent. Cache/reasoning counters are provider-defined breakdowns, not extra values to add to total tokens; adapters document their mapping. Native prompt-cache policy is separate from response caching and is not represented by a universal retention-duration promise in this core contract.

### Streaming and delivery negotiation

The RPC is server-streaming with one request; HTTP always uses POST SSE. Absent/unspecified `delivery` means `EVENTS`; `ARTIFACT` explicitly requests transcript persistence. Unknown values fail validation, and the runtime never silently switches modes because output is large or media appears. Neither transport requires the client to know the model architecture or eventual content kinds.

```protobuf
message InferenceCompleteStreamOptionsAlpha1 {
  enum ContentMode {
    CONTENT_MODE_UNSPECIFIED = 0;
    APPEND = 1;
    ALLOW_REPLACE = 2;
    PREFER_REPLACE = 3;
  }
  ContentMode content_mode = 1;
  optional bool include_usage = 2;
  optional bool partial_tool_calls = 3;
}

enum InferenceUpdateModeAlpha1 {
  INFERENCE_UPDATE_MODE_ALPHA1_UNSPECIFIED = 0;
  INFERENCE_UPDATE_MODE_ALPHA1_APPEND = 1;
  INFERENCE_UPDATE_MODE_ALPHA1_REPLACE = 2;
}

enum InferenceStreamDeliveryAlpha1 {
  INFERENCE_STREAM_DELIVERY_ALPHA1_UNSPECIFIED = 0;
  INFERENCE_STREAM_DELIVERY_ALPHA1_NATIVE = 1;
  INFERENCE_STREAM_DELIVERY_ALPHA1_BUFFERED = 2;
}

message InferenceCompleteStreamEventAlpha1 {
  uint64 sequence = 1;
  oneof event {
    InferenceCompleteStartedAlpha1 started = 2;
    InferenceCompleteTurnStartedAlpha1 turn_started = 3;
    InferenceCompleteBlockStartedAlpha1 block_started = 4;
    InferenceCompleteBlockDeltaAlpha1 block_delta = 5;
    InferenceCompleteBlockCompletedAlpha1 block_completed = 6;
    InferenceCompleteTurnCompletedAlpha1 turn_completed = 7;
    InferenceUsageAlpha1 usage = 8;
    InferenceCompleteCompletedAlpha1 completed = 9;
  }
}

message InferenceCompleteStartedAlpha1 {
  string id = 1;
  InferenceCompleteDeliveryModeAlpha1 delivery = 2;
  // Effective model after default/override resolution; completed.model carries
  // the provider-reported identity, which may differ (alias resolution).
  string model = 3;
}

message InferenceCompleteTurnStartedAlpha1 {
  uint32 turn_index = 1;
  InferenceTurnAlpha1.TurnRole role = 2;
}

message InferenceCompleteBlockStartedAlpha1 {
  enum Kind {
    KIND_UNSPECIFIED = 0;
    TEXT = 1;
    MEDIA = 2;
    TOOL_CALL = 3;
    REASONING_SUMMARY = 4;
    OPAQUE_CONTENT = 5;
  }
  uint32 turn_index = 1;
  uint32 block_index = 2;
  string id = 3;
  Kind kind = 4;
  InferenceStreamDeliveryAlpha1 delivery = 5;
  bool revisable = 6;
}

message InferenceCompleteBlockDeltaAlpha1 {
  uint32 turn_index = 1;
  uint32 block_index = 2;
  oneof delta {
    InferenceTextUpdateAlpha1 text = 3;
    string tool_arguments_json = 4;
  }
}

message InferenceTextUpdateAlpha1 {
  InferenceUpdateModeAlpha1 mode = 1;
  string text = 2;
}

message InferenceCompleteBlockCompletedAlpha1 {
  uint32 turn_index = 1;
  uint32 block_index = 2;
  repeated InferenceProviderStateAlpha1 provider_state = 3;
  oneof final_value {
    InferenceMediaAlpha1 media = 4;
    InferenceToolCallAlpha1 tool_call = 5;
    InferenceProviderStateAlpha1 opaque_content = 6;
  }
}

message InferenceCompleteTurnCompletedAlpha1 {
  uint32 turn_index = 1;
  repeated InferenceProviderStateAlpha1 provider_state = 2;
}

message InferenceCompleteCompletedAlpha1 {
  InferenceCompletionOutcomeAlpha1 outcome = 1;
  optional string model = 2;
  // Present only for ARTIFACT delivery, after successful publication.
  InferenceCompleteArtifactAlpha1 artifact = 3;
}
```

The table abbreviates enum names; JSON uses their full protobuf names.

| Requested content mode | Native append source | Snapshot-only source | Dual-mode source |
| --- | --- | --- | --- |
| `APPEND` (also absent/unspecified) | Forward appends | Buffer each revisable block until final; emit final text in append chunks | Select append |
| `ALLOW_REPLACE` | Forward appends | Forward replacements | Retain default append; permission is not a request to generate snapshots |
| `PREFER_REPLACE` | Forward appends | Forward replacements | Prefer snapshots where supported; still permits append or required buffering |

`include_usage` defaults to true and `partial_tool_calls` to false. Unknown requested modes fail validation. `stream_options` applies to both delivered content events and persisted transcript events with the same defaults. Artifact callers should choose `ALLOW_REPLACE` to persist native snapshots without snapshot-to-append buffering. There is no architecture flag or guarantee of low latency merely because a model uses diffusion. A component may expose committed diffusion blocks as appends, revisable snapshots as replacements, or only a final unary result.

The component declares each source block revisable before emitting any of its content. The runtime selects delivery before publishing `block_started`. `revisable` describes updates the client may receive, independently of `delivery`: native snapshots have `revisable = true`; append-only and final-only delivery have `revisable = false`. An undeclared source replacement is a component contract error, even when the client accepts replacements. Snapshot-to-append buffering emits only finalized append chunks and never exposes replacements to an append-only client. For mixed responses, buffering and delivery are block-scoped; whole-response processing can force all blocks to `BUFFERED`. Live streaming never replays a response-cache entry.

`APPEND` concatenates text; `REPLACE` overwrites the entire addressed block, never the turn, its other blocks, tool identities, or continuation data. Empty replacement text clears the block, including when protobuf JSON omits the default-valued `text` field. Replacement is permitted only for text and visible reasoning-summary blocks, and only after negotiation. Media and opaque content are emitted complete, not as byte or signature deltas. Unknown/unspecified update modes fail the stream.

No stable prefix is inferred by comparing snapshots: `"cat"` followed by `"car"` proves why already-sent text cannot safely become an append stream. Partial-tail edits, block deletion/reordering, and revisable tool identities are not part of this version. A source that revises response structure must buffer until the structure is stable before opening its public blocks. Intermediate text is a preview, not valid JSON, an executable call, or a finalized answer.

#### Event grammar and folding (normative)

1. Emit exactly one `started` with the invocation ID, the effective model, and the resolved, non-unspecified delivery mode. In `EVENTS` delivery, turn indices are contiguous from zero in start order; block indices are contiguous within each turn in start order. A block's index, ID, and kind never change. Deltas for open blocks may interleave and are keyed by `(turn_index, block_index)`. Arrays fold in index order, not completion order.
2. Every opened block has exactly one `block_completed`, after all its deltas. Text/summary blocks always receive a text update, even for empty final text. The component must leave the final text in effect before completion; there is no final text field that can hide an unnegotiated revision. Completed blocks accept no further updates.
3. For text/summary blocks, `block_completed` is a marker plus optional block-scoped provider state; `final_value` MUST be absent. It does not repeat text, the block ID, or the kind supplied at start. Media/opaque blocks have no deltas and carry the matching final value; tool blocks carry a final structured call, optionally preceded by argument fragments. A missing or mismatched final-value kind is a protocol error. State and non-text values must fit existing transport message limits.
4. Tool arguments are append-only UTF-8 fragments when partial delivery is enabled; otherwise only the completed call is exposed. Final fragments must parse as a JSON object matching the completed call's arguments. Revisable arguments are buffered by the adapter and delivered only once stable; replacement text is never interpreted as a tool call. Valid JSON does not confer execution authorization or guarantee adherence to the tool schema. Applications validate and execute calls only after successful request completion with `TOOL_RESULTS_REQUIRED`.
5. Emit one `turn_completed` after every block in that turn completes; it supplies turn-scoped state. Emit at most one authoritative `usage`, after all turns complete and only when requested and available. Emit exactly one `completed` after all normalization, output validation, and response processing succeed. No events follow it. For `EVENTS`, `completed.artifact` MUST be absent; generated-media references belong to content blocks.
6. For `ARTIFACT`, the outward stream contains only `started`, optional `usage`, and `completed`. No turn/block events are sent outward. The terminal event MUST include the successfully published transcript descriptor and format in `artifact`, plus the same outcome/model/usage semantics as the stored content stream. A missing reference or unexpected content event fails validation. The stored transcript follows the `EVENTS` grammar, as specified below.
7. An opt-in fold helper consumes an `EVENTS` stream, collecting the response ID and effective model from `started`, role/ID/kind from turn/block starts, text from append/replace updates, non-text values and state from block/turn completion, optional usage, and outcome/model metadata from `completed`. It assembles the ordered `InferenceContentBlockAlpha1` / turn arrays only because its caller requested materialization. For a fixed processed provider response, the folded content must equal the normalized provider result, except explicitly omitted usage. An artifact-reference stream is not an empty completion: folding it fails explicitly and directs the caller to read the transcript first. No helper implicitly downloads artifacts or reruns inference.
8. Success requires `completed` **and** successful transport termination (including OK gRPC status). Provider failure, protocol violation, cancellation, or EOF without completion yields no successful result, even if some blocks finished or a reference was received. SDKs discard provisional snapshot previews on failure. An append-only prefix received before the failure may be exposed, but only through the failure itself: the SDK's error/exception type carries a `partial_output` member (C# `PartialOutput`, Python `partial_output`, TypeScript `partialOutput`, Go `PartialOutput()`, Java `getPartialOutput()`, Rust `partial_output()`) containing the folded incomplete turns, and the ordinary success return path never yields it. Rendering code that only reads the success path therefore cannot mistake a truncated answer for a complete one.

Adapters split append text and JSON argument fragments on UTF-8 code-point boundaries as needed to fit existing transport message limits. A replacement is one atomic snapshot and cannot be split into successive replacements without changing its meaning; a snapshot exceeding the applicable transport limit fails explicitly. Sequence numbers are contiguous from zero and assigned at emission, after coalescing; they do not support resumption. Clients fail on unknown semantic events or block kinds rather than silently constructing a lossy result. Future ignorable diagnostics must be distinguishable from fold-affecting events; new semantic kinds require versioning or explicit negotiation.

#### HTTP and runtime behavior

`POST /v1.0-alpha1/completioninference/{name}/complete` always returns SSE on success, for both delivery modes. There is no `stream` query selector or unary JSON response; supplying a `stream` query parameter fails with `ERR_INFERENCE_INVALID_REQUEST`. An absent `Accept` header or one permitting `text/event-stream` (including a matching wildcard with nonzero quality) is accepted; an incompatible `Accept` returns `406`. The path supplies `name`; a conflicting body name fails validation.

Validate request shape, component resolution, model-override policy, model-access authorization, and any locally known restrictions before flushing `200`, `Content-Type: text/event-stream`, `Cache-Control: no-cache`, and `X-Accel-Buffering: no`. Model-specific restrictions may subsequently fail upstream; no capability inventory is required. SSE uses `event:` equal to the event oneof name, `data:` equal to that selected payload's protobuf JSON, and `id:` equal to decimal `sequence`. The `id:` field exists so clients can detect gaps; it does not enable resumption. A request carrying a `Last-Event-ID` header is rejected with `400` so that a reconnecting generic SSE client fails fast instead of silently receiving a fresh generation. Keep-alive comments are emitted every `keepAliveInterval` (default 15 seconds) from the moment headers flush, including during provider waits and buffering. Browser callers use POST `fetch` or an SDK, not GET-only `EventSource`.

Pre-open errors use normal Dapr HTTP/gRPC errors from the [error code reference](#error-code-reference). Post-open HTTP errors use `event: error` with Dapr's `errorCode` and `message`, then close without `completed`. gRPC uses a non-OK trailing status, not an application-level success event.

The outbound timeout and request deadline bound the entire operation; neither deltas nor keep-alives reset them. Delay `started` until the first usable component event or processed buffered response. Retry policies may retry only before any semantic event is emitted, discarding unsent state; SSE headers and comments do not end that window. Such retries may still incur duplicate provider work/billing. After any semantic event, fail without transparent replay. Client cancellation MUST propagate through the runtime to the active provider invocation, including blocked sends, upstream reads, and native cancellation of any job owned by that invocation. Merely returning a canceled local task while provider work continues unaddressed does not satisfy the contract. [End-to-end cancellation](#end-to-end-cancellation) defines the required actions, confirmation rules, and providers' cancellation limitations.

Use existing runtime/SDK flow control and backpressure for transport and artifact sinks; slow consumers suspend production rather than accumulating the entire response. Coalesce only queued, unsent text updates for the same block that a later `REPLACE` supersedes, without crossing block lifecycle or non-text event barriers. Never drop tool fragments, state, lifecycle events, or unsuperseded appends. Assign sequence numbers after coalescing.

On the native incremental path, runtime state contains in-flight events, per-open-block validation state, counters, and protocol metadata, not accumulated text or completed block payloads. Providers must decode incrementally rather than allocate an entire native response before splitting it; duplicated native final-text fields must also be skipped/checked incrementally, not deserialized into a full string. A large append-only text block can span many events without being accumulated.

This proposal adds no inference-specific chunk, event, JSONL-record, queue, turn/block-count, SDK-fold, or total-stream size limits or configuration knobs. Existing gRPC send/receive message limits and runtime/SDK transport behavior continue to apply. A per-message limit is not a limit on the combined size of a stream; HTTP/SSE and stored transcripts do not acquire an additional inference-specific cap.

Full-response/block buffering is permitted only for an explicit materializing operation or a documented requirement below. Configured external staging can reduce memory only where the relevant processor actually supports incremental or external-memory access; writing a file and then loading it all back is not a streaming implementation. Existing transport failures remain explicit errors, never silent truncation.

| Requirement | Permitted buffering and visibility |
| --- | --- |
| Explicit SDK `Fold` | Assemble locally only because the caller requested materialization; no unary completion response is sent by the runtime |
| Unary-only provider | Replay its materialized result as `BUFFERED`; this is not native incremental generation |
| Snapshot source serving `APPEND` | Retain current text per affected block, replacing prior snapshots and applying intervening appends, then split final text into appends; mark `BUFFERED` |
| PII scrubbing | Buffer and scrub before any provider content or tool fragments reach a client or published artifact; all blocks are `BUFFERED` |
| JSON/schema validation | Use an incremental validator where semantics permit; otherwise final-document processing is required and affected blocks are `BUFFERED` |
| Tool JSON / opaque state | Materialize structured values where required; final event messages remain subject to existing transport limits |

The runtime, not the provider component, owns full-response processing and fallback decisions. A streaming-only component still produces events when runtime processing requires buffering; it is not instructed to allocate a full response internally. Select/report buffering before exposing affected blocks, and reject known unsupported requirements before invocation. JSON validation consumes the logical concatenation specified by the output constraint; this need not be an allocated string when an incremental validator can implement it. Incremental validation errors still terminate without successful `completed`. The unary fallback emitter also uses marker-only text completion.

Signed/opaque state is never rewritten; if required scrubbing would modify content bound to such state, fail explicitly rather than emit invalid continuation state or leak original content. Text/summary blocks and textual tool-result content are eligible text surfaces; opaque payloads and binary media carry no PII-scrubbing guarantee.

Completion has no cacheable operation and no cache fields; the [shared cache contract](#optional-shared-inference-cache-contract) applies to adopting siblings only. Explicit artifact persistence is a caller-selected delivery mode, not a cache.

### Artifact-backed completion

Set `delivery = ARTIFACT` on the same `InferenceCompleteRequestAlpha1` and invoke `CompleteAlpha1` / `POST /v1.0-alpha1/completioninference/{name}/complete`. The response remains a stream, ending in `completed.artifact` only after successful persistence. There is no separate artifact-generation route/RPC, unary descriptor response, or asynchronous operation handle. The request remains active through generation, publication, and stream termination.

The runtime connects the common normalized event producer to the component's configured `artifactStore`, serializing one event at a time with backpressure. No new provider-specific "generate to artifact" interface is needed. Validate the store binding, ordinary access controls, and artifact-read path before generation; a missing `artifactStore` fails with `ERR_INFERENCE_FEATURE_UNSUPPORTED`. Operators own storage and sensitive-data policies. A `Completer`-only provider is available through the documented materializing fallback, not as a native streaming provider. Canceling this call cancels the active provider invocation as well as the storage write; it must not leave generation running merely because the client would have received only an artifact descriptor.

`completed.artifact.format = COMPLETION_EVENTS_JSONL_V1ALPHA1` identifies a UTF-8 JSON Lines artifact with MIME type `application/x-ndjson`. Each newline-terminated record is the protobuf JSON encoding of one complete `InferenceCompleteStreamEventAlpha1` envelope, including `sequence` and the selected event field, not the SSE selected-payload encoding. The stored stream uses `started.delivery = EVENTS` and ends with `completed` without a transcript reference; it never recursively references itself. Its invocation ID matches the outward stream, but each sequence independently starts at zero. Embedded newlines are JSON-escaped; 64-bit values use protobuf JSON strings and opaque bytes use its base64 mapping. This preserves the exact event grammar and round-trippable state while processing one record at a time. Future incompatible formats require a new format value.

The artifact is a successfully completed **event transcript**, not a JSON serialization of the folded response or just concatenated text. It may include interim diffusion snapshots, provisional tool fragments, and provider continuation state. Selecting artifact delivery explicitly retains these sensitive previews. Operators configure retention, encryption, and provider/store data handling; this proposal does not add a Dapr governance policy or certify those external guarantees. Normal Dapr authorization and existing scrubbing/logging behavior still apply. The transcript is not an audit record of every provider token: normalization and safe coalescing may omit superseded unsent snapshots. PII-enabled processing persists only processed output, never the unprocessed preview transcript.

Append/replace events are written directly; the runtime does not fold snapshots merely to persist them. A client choosing `ALLOW_REPLACE` or `PREFER_REPLACE` permits this path; `APPEND` retains its buffering guarantee. Buffered processors write their processed event sequence instead. All records, including terminal `completed`, are written to an unpublished staging object. Commit publication and expose its immutable descriptor in the outward `completed` only after component success, validation, storage finalization, and integrity checks. A generation, cancellation, or storage failure fails the outward stream without `completed` and aborts/cleans up staging, even if a terminal record was already written there. Do not transparently retry generation after the first event is persisted or emitted to the client. A response lost after successful publication has the usual ambiguous-delivery outcome; no exactly-once generation promise is made.

Read the artifact through `ReadCompletionArtifactAlpha1` and the generic descriptor/read contract in [artifact transfer](#artifact-transfer). A lazy SDK `ReadCompletionEvents` helper verifies the advertised format, incrementally splits JSONL records across arbitrary byte-chunk boundaries, parses typed events, and validates ordering and terminal success. It does not fold them. Success requires both a valid terminal completion record and successful underlying artifact length/digest/transport validation; a truncated file with a completion-looking prefix is not a successful response.

Applications may process/discard events, render previews, persist them elsewhere, or explicitly invoke a fold helper. Persisting a completion transcript does not copy or guarantee continued availability of referenced image/video artifacts; each reference is resolved through its own storage and access checks. Download cancellation stops only that read, not a generation or the stored artifact. Artifact reads preserve the original completion ID; they do not invoke the model again.

## Provider-native completion batches

Batch completion targets independent, non-urgent requests: evaluations, document classification, bulk extraction, and content generation. It uses the provider's native deferred-processing API, where batch pricing and separate quotas may be available. Dapr does not promise a particular discount or turnaround time, and MUST NOT implement this API by fanning out ordinary interactive calls. Unsupported integrations return `ERR_INFERENCE_BATCH_UNSUPPORTED` (HTTP `501` / gRPC `Unimplemented`) before provider work.

`CompleteAlpha1` remains request-bound and always returns a stream. A batch instead has a short-lived submission call, an independently running provider job, and later result retrieval. Status/submission responses contain only control metadata; generated content is read as a stream.

### Batch submission and input contract

Each batch addresses one Completion component and one effective model in this Alpha. Model defaults, override locks, and component-owned model routing resolve once from the initial frame. Capture the resolved routing options and configuration identity in the durable batch binding; later configuration changes cannot reinterpret accepted work. Native batch support for those options is checked independently of interactive support, and unsupported options fail rather than being dropped. Neither batch options nor items accept routing overrides. Requests needing different models are submitted separately; the runtime does not silently split one batch into multiple provider jobs.

Submission is client-streaming over gRPC. HTTP uses `POST .../batches` with `Content-Type: application/x-ndjson`: each newline-terminated record is one `InferenceSubmitCompletionBatchRequestAlpha1` protobuf-JSON envelope. The path supplies the initial component name; a conflicting body name fails validation. Both transports require exactly one `initial`, one or more `item` frames with a present request, exactly one `finish`, and clean request-body termination. The declared final count must match the number of items. No records follow `finish`. Missing finish, duplicate IDs, malformed records, mismatched counts, or aborted uploads fail without creating a provider batch.

The runtime validates the entire submission before creating the billable job. It stages normalized input incrementally in `batchStore` (or `batchStagingStore` when configured); adapters translate the validated input into the provider's inline request array or uploaded input file. Validate model authorization and all input artifact references before provider uploads, not merely before job creation. A component without `batchStore` fails configuration validation when the adapter implements batches, rather than buffering an unbounded batch in memory. Existing transport and provider limits apply; there is no new inference-specific total-upload or item-count cap.

Items reuse the common completion request rather than a provider-specific JSON body:

- `item_id` is a nonempty, caller-chosen correlation ID, unique within the batch. Results may arrive in any order; array position is not identity. Adapters preserve IDs or maintain a reversible mapping when provider ID syntax differs.
- An item's `name` and `model` may be omitted or exactly match the resolved batch component/model. Conflicting values fail validation; no per-item default re-resolution occurs.
- Instructions, ordered history, text/media inputs, tools/results, output constraints, generation controls, metadata, and `scrub_pii` retain their normal semantics, subject to native batch support.
- Item `stream_options` must be absent and `delivery` must be absent/`EVENTS`. Final batch results do not contain live diffusion previews, and per-item transcript delivery is not selected at submission. Unsupported fields or an item-level `idempotency_key` fail explicitly rather than being ignored.
- Optional idempotency belongs to the **batch submission** and is forwarded only when that provider's batch-create operation supports it. Per-item IDs are correlation keys, not deduplication keys.
- Tools may be declared and prior tool results supplied where the provider's batch endpoint supports them. An item ending in `TOOL_RESULTS_REQUIRED` is a completed model response; the application executes the calls and submits the follow-up as a new request, exactly as in the interactive path.

<details>
<summary>Batch wire contract</summary>

These types belong in `completion.proto`. RPC bindings are listed in the [API table](#completion-api).

```protobuf
message InferenceSubmitCompletionBatchRequestAlpha1 {
  oneof frame {
    InferenceCompletionBatchOptionsAlpha1 initial = 1;
    InferenceCompletionBatchItemAlpha1 item = 2;
    InferenceCompletionBatchFinishAlpha1 finish = 3;
  }
}

message InferenceCompletionBatchOptionsAlpha1 {
  string name = 1;
  optional string model = 2;
  optional string idempotency_key = 3;
  map<string, string> metadata = 4;
}

message InferenceCompletionBatchItemAlpha1 {
  string item_id = 1;
  InferenceCompleteRequestAlpha1 request = 2;
}

message InferenceCompletionBatchFinishAlpha1 {
  uint64 item_count = 1;
}

message InferenceCompletionBatchRequestAlpha1 {
  string name = 1;
  string batch_id = 2;
}

enum InferenceCompletionBatchStatusAlpha1 {
  INFERENCE_COMPLETION_BATCH_STATUS_ALPHA1_UNSPECIFIED = 0;
  INFERENCE_COMPLETION_BATCH_STATUS_ALPHA1_PENDING = 1;
  INFERENCE_COMPLETION_BATCH_STATUS_ALPHA1_RUNNING = 2;
  INFERENCE_COMPLETION_BATCH_STATUS_ALPHA1_CANCELING = 3;
  INFERENCE_COMPLETION_BATCH_STATUS_ALPHA1_COMPLETED = 4;
  INFERENCE_COMPLETION_BATCH_STATUS_ALPHA1_FAILED = 5;
  INFERENCE_COMPLETION_BATCH_STATUS_ALPHA1_CANCELED = 6;
  INFERENCE_COMPLETION_BATCH_STATUS_ALPHA1_EXPIRED = 7;
}

message InferenceCompletionBatchAlpha1 {
  string batch_id = 1;
  InferenceCompletionBatchStatusAlpha1 status = 2;
  string model = 3;
  uint64 total_items = 4;
  optional uint64 processed_items = 5;
  bool results_available = 6;
  InferenceCompletionBatchErrorAlpha1 error = 7;
  // Runtime acceptance time; always present.
  google.protobuf.Timestamp created_at = 8;
  // Provider-reported end of the execution window, when the provider exposes one.
  optional google.protobuf.Timestamp expires_at = 9;
  // Provider-reported terminal time, when known.
  optional google.protobuf.Timestamp completed_at = 10;
}

message InferenceCompletionBatchErrorAlpha1 {
  string code = 1;
  string message = 2;
}

message InferenceCompletionBatchItemErrorAlpha1 {
  enum Kind {
    KIND_UNSPECIFIED = 0;
    FAILED = 1;
    CANCELED = 2;
    EXPIRED = 3;
  }
  Kind kind = 1;
  InferenceCompletionBatchErrorAlpha1 error = 2;
}

message InferenceCompletionBatchResultEventAlpha1 {
  uint64 sequence = 1;
  // Empty for batch-level started/completed; required for item events.
  string item_id = 2;
  oneof event {
    InferenceCompletionBatchAlpha1 started = 3;
    InferenceCompleteStreamEventAlpha1 completion_event = 4;
    InferenceCompletionBatchItemErrorAlpha1 item_error = 5;
    InferenceCompletionBatchResultsCompletedAlpha1 completed = 6;
  }
}

message InferenceCompletionBatchResultsCompletedAlpha1 {
  uint64 succeeded = 1;
  uint64 failed = 2;
  uint64 canceled = 3;
  uint64 expired = 4;
}
```

</details>

### Batch lifecycle and cancellation

Successful submission returns HTTP `202` with `InferenceCompletionBatchAlpha1` and `Location` pointing to `.../batches/{batchId}`; gRPC returns the same handle/status after the upload closes. This is acceptance, not completion of all items. Providers may validate individual requests asynchronously after acceptance.

`PENDING` includes native validation/queueing; `RUNNING` includes processing/finalization; `CANCELING` indicates an accepted but unconfirmed stop. `COMPLETED`, `FAILED`, `CANCELED`, and `EXPIRED` are terminal. `COMPLETED` means provider processing ended normally, not that every item succeeded. Item failures, refusals, and requests for tool results must remain distinguishable. `EXPIRED` represents a provider's batch execution window ending; `expires_at` exposes that window when the provider reports it so Workflow pollers can size timers, and `created_at`/`completed_at` bound the job's observed lifetime. Unknown statuses fail mapping explicitly. Provider-native completion windows and retention limits are documented by the adapter, not fixed in the common API.

Batch IDs are opaque and bound to the application/namespace, component, provider job, and original effective model. Preserve an immutable submission manifest in `batchStore` with item IDs and the minimum per-item normalization context (output constraints, tool definitions, and processing/scrubbing requirements), so that bindings survive sidecar restarts and replicas. Sensitive manifests follow operator storage/access policy. Report the originally resolved model in the batch snapshot; individual results may additionally carry provider-reported identities.

`total_items` is the validated submission count. `processed_items` is optional provider-reported progress, never inferred from polls or claimed when unavailable, and must not exceed that total. `results_available` is true only for a terminal batch with retrievable results. Batch-level failures carry a nonempty mapped error code/message; status or count inconsistencies are explicit adapter errors. Persisting the accepted job binding must succeed before returning a usable handle. A persistence failure after native creation triggers cancellation/cleanup where supported and an explicit submission error, not an untracked successful batch.

Cancellation before submission handoff stops upload/provider I/O, cleans unpublished staging, and requests native cancellation for a job already created, using the existing finite cleanup context. Job-ID races and ambiguous delivery are handled explicitly; a lost acknowledgment does not authorize blind resubmission. After handoff, a poll/read cancellation only stops that lookup/download. `CancelCompletionBatchAlpha1` requests cancellation of the independent job and returns HTTP `204` / gRPC empty success when accepted, not a guarantee that no items completed or were billed. An already-terminal batch returns `ERR_INFERENCE_BATCH_NOT_CANCELABLE`. Confirm terminal cancellation through status; completion races preserve the actual outcome. Unsupported native cancellation returns `ERR_INFERENCE_CANCELLATION_UNSUPPORTED`, never a false local-only success.

No background polling is required inside daprd. The provider owns execution; applications or Workflows may poll the status API. Submission retries are off by default and require the native submission-idempotency contract when enabled. Status/result reads never submit new generation or retry failed items automatically. Applications can submit selected failed items in a new batch; reusing their correlation IDs in that new batch does not imply exactly-once execution.

### Batch result streaming

`ReadCompletionBatchResultsAlpha1` streams the immutable final result set once the provider job is terminal and results are available. HTTP `GET .../results` returns SSE, with the same Accept, keep-alive, error, and successful-termination rules as interactive completion; it is not a unary array or a redirect to a provider URL. `event:` selects the outer event variant, `id:` is its decimal sequence, and `data:` contains the **full** `InferenceCompletionBatchResultEventAlpha1` protobuf-JSON envelope so `item_id` is retained. Providers returning separate success/error files are merged by the adapter into one logical result stream without assuming order.

1. Emit one `started` snapshot with an empty `item_id`. Then process items sequentially in provider result order; do not interleave two items' event sequences.
2. Each result identifies its original `item_id` and contains either a normalized completion event sequence or one terminal `item_error`. Nested completion sequences use the existing `EVENTS` grammar, have independent sequence numbers starting at zero, and mark final-response replay as `BUFFERED`. Result IDs remain stable for the batch/item across reads. Text is chunked to existing message limits rather than returned as a whole-batch array; generated media uses the existing authorized artifact-reference/read path.
3. A normalization/validation failure may terminate an item's provisional nested events with `item_error` instead of nested `completed`. That item is unsuccessful, its provisional output is discarded, and other items may continue. Provider error records use explicit nonempty mapped codes/messages; they are data about an item, not a transport failure. Unknown IDs, duplicates, corrupt result framing, or failure to read the source fail the overall result stream rather than silently dropping records.
4. Emit exactly one outer `completed` after every submitted ID has one terminal disposition. Its counts sum to `total_items`; `succeeded` counts normalized responses, including valid `REFUSED` or `TOOL_RESULTS_REQUIRED` outcomes, not only generated text. Classify missing unprocessed items as canceled/expired/failed only when authoritative provider semantics establish that outcome; otherwise fail with `ERR_INFERENCE_BATCH_RESULTS_INCOMPLETE` (HTTP `502` before headers / gRPC `Internal`; after SSE headers, use the stream error path without `completed`). Never invent successful results to satisfy the count.
5. Outer `completed` plus successful transport termination means retrieval succeeded, even when some items failed or the batch was canceled/expired. It does not change the batch's terminal status. The outer sequence is contiguous from zero; no resumable cursor or transparent replay after emitted events is promised.

Nonterminal retrieval returns `ERR_INFERENCE_BATCH_RESULTS_NOT_READY` (HTTP `409` / gRPC `FailedPrecondition`). Terminal batches without retrievable results return `ERR_INFERENCE_BATCH_RESULTS_UNAVAILABLE` (HTTP `404` / gRPC `NotFound`). `results_available` is a snapshot, not a retention guarantee. Missing results never trigger regeneration. Successfully completed items may remain retrievable after batch cancellation/expiration where the provider supports them; this is distinct from exposing partial files from a failed individual generation.

The runtime normalizes one item at a time, reapplying the bound output-processing requirements before exposure. It does not load the complete native results file or all item responses to serve a stream. Existing per-item buffering exceptions still apply, and correlation/validation metadata may scale with item count. Cross-record validation and processing must not be skipped to claim constant memory. A missing required processing manifest is an explicit read failure, not permission to return unvalidated or unscrubbed provider output.

### Batch component boundary

```go
type CompletionBatcher interface {
    SubmitBatch(ctx context.Context, req *BatchSubmitRequest) (*Batch, error)
    GetBatch(ctx context.Context, batchID string) (*Batch, error)
    CancelBatch(ctx context.Context, batchID string) error
    ReadBatchResults(ctx context.Context, batchID string, out chan<- BatchResultChunk) error
}
```

`BatchSubmitRequest` carries the resolved component model and operator-owned routing snapshot, supported submission key, metadata, and a closable iterator over validated staged items, not an unbounded request slice. Component types mirror the batch wire semantics without importing generated protos. `BatchResultChunk` identifies the item and carries normalized completion chunks or an explicit item error; the runtime owns outer framing, sequence numbers, integrity checks, and public completion. The component never closes the runtime-owned channel, respects cancellation/backpressure on every send/read, and reports source errors explicitly.

Native file upload, endpoint selection, correlation-ID mapping, output/error-file merging, and provider status mapping belong in the adapter. Provider input/output file handles and credentials remain internal. Reuse the existing Completion normalizer; do not turn the batch interface into an OpenAI-shaped universal payload or a cross-purpose dispatcher.

| Integration | Native mapping | Contract boundary |
| --- | --- | --- |
| [OpenAI Batch](https://developers.openai.com/api/docs/guides/batch) | Translate items into its input-file format, create one native batch, and merge result/error files by correlation ID | Select a supported completion endpoint/model; no interactive fan-out |
| [Anthropic Message Batches](https://platform.claude.com/docs/en/build-with-claude/batch-processing) | Map items to native message requests and custom IDs, then normalize final result records | Preserve native succeeded/errored/canceled/expired item outcomes |
| [Gemini Batch](https://ai.google.dev/gemini-api/docs/batch-api) | Map one effective model and keyed requests to native inline/file submission and result retrieval | Decode inline result arrays incrementally or stream file results; no whole-batch materialization |

These integrations illustrate native discount opportunities, not a Dapr price guarantee, mandatory provider list, or static model catalog. Adapters document supported batch options, provider limits, completion windows, cancellation, and result retention. A locally known unsupported batch feature fails before submission; unknown restrictions surface as provider validation errors or correlated item failures.

### Batch governance and shared runtime behavior

`CompletionAccessPolicy` adds `submitBatch`, `getBatch`, `cancelBatch`, and `readBatchResults`. A `complete` grant does not grant batch submission, or vice versa. Authorize submission against the resolved model and all input references before provider uploads. Authorize subsequent calls against the trusted original model and current policy before status/cancel/result access. Later component-default changes cannot retarget a batch; invalid/unverifiable bindings fail closed. Reading generated artifacts separately still requires `readArtifact`.

Batch operations have no cache fields and perform no cache reads or writes. Native provider prompt caching remains separate. Metrics use bounded building-block/component/operation/status labels; item IDs, batch IDs, prompts, and models are not metric labels. Preserve authoritative per-item usage where available without inferring monetary savings.

## Shared runtime extras

This section defines reusable conventions, not an Inference building block or a mandatory feature checklist. Completion uses its applicable portions; each sibling proposal explicitly adopts the extras it needs and specifies its purpose-specific behavior. Sharing code or a backend does not share component configuration, credentials, retry policies, model defaults, cache entries, or cancellation ownership.

| Runtime concern | Reference |
| --- | --- |
| Schema ownership | [Shared wire types](#shared-wire-types) |
| Configuration and governance | [Model resolution](#model-resolution-and-override-policy), [access policies](#model-access-policies) |
| Invocation behavior | [Resiliency and processing](#resiliency-processing-and-observability), [idempotency](#provider-native-idempotency), [cancellation](#end-to-end-cancellation) |
| Generated output | [Artifact transfer](#artifact-transfer) |
| Optional sibling response caching | [Cache contract](#optional-shared-inference-cache-contract), [Centralized Cache integration](#centralized-cache-integration); adopted by siblings only |

### Shared wire types

Define reusable values once in `dapr/proto/runtime/v1/inference_common.proto`; purpose-specific protos import them in the existing runtime proto package. Completion's messages/events belong in `completion.proto`. Each sibling owns its request/result types and RPC bindings in its own proto.

Every top-level inference message and enum starts with `Inference`, including purpose-specific requests, results, and supporting types. Examples include `InferenceCompleteRequestAlpha1`, `InferenceDecideRequestAlpha1`, `InferenceEmbedRequestAlpha1`, `InferenceRerankRequestAlpha1`, `InferenceGenerateImageRequestAlpha1`, and `InferenceGenerateVideoRequestAlpha1`. Shared operation/read requests follow the same rule, such as `InferenceGetOperationRequestAlpha1` and `InferenceReadArtifactRequestAlpha1`. File separation organizes definitions but does not create a protobuf namespace boundary. Top-level enum values use the corresponding `INFERENCE_` prefix; nested types and values remain scoped by their enclosing message.

This convention applies to wire types and their generated bindings, not to RPC names, HTTP paths, component types, or policy operations. The purpose-specific SDK examples use shorter convenience-model names without the `Inference` prefix or `Alpha1` suffix; those wrappers map explicitly to the prefixed wire types. Provider-facing Go types likewise remain scoped by their purpose-specific packages.

| Shared types | Definition in this proposal |
| --- | --- |
| `InferenceUsageAlpha1`, `InferenceMediaAlpha1`, `InferenceProviderStateAlpha1` | Shown alongside Completion content types above; emitted in `inference_common.proto` |
| `InferenceCacheOptionsAlpha1`, `InferenceCacheResultAlpha1` | Optional shared cache contract below |
| Artifact descriptors/references and read request/response/completion messages | Artifact transfer below |
| Operation request/envelope/status messages | Asynchronous operation conventions below; only adopting job-based blocks expose these APIs |
| Model listing/metadata requests, responses, deployment and price values | Model discovery below; each purpose binds its own RPCs and component interfaces |

Shared value types do not create shared routing. The selected purpose-specific RPC/HTTP route fixes the registry used to resolve `name`, operation IDs, and artifact references. `InferenceArtifactReferenceAlpha1.name` resolves in that same building block, including media inputs. Cross-block transfer requires an explicit source read and a supported destination input/upload.

### Model resolution and override policy

Completion requests and adopting building blocks' generation/evaluation requests have an optional `model`. SDKs expose it directly rather than placing it in metadata. The runtime resolves the effective model once, before cache lookup, idempotency forwarding, or provider invocation:

1. Read `defaultModel` from the addressed purpose-specific component. There is no family-level or other-task fallback.
2. With `disableModelOverride` absent/`false`, use the request's model when supplied, otherwise that component default.
3. With `disableModelOverride = true`, use the default. A request specifying that exact same model is allowed; a different model fails with `ERR_INFERENCE_MODEL_OVERRIDE_DISABLED` (HTTP `400` / gRPC `InvalidArgument`), not a silent fallback. A cache hit must not bypass this check.
4. If no model can be resolved, or a supplied model is empty/whitespace-only, reject the request before invocation. A locked component with no configured default cannot use a request model as a substitute. Preserve model identifiers' case and provider-specific syntax; no runtime-maintained model inventory is required.
5. Authorize the effective model and operation against any applicable [model access policies](#model-access-policies), including when the model came from the component default or matched a locked default. An optional operator-authored allowlist restricts use; it is not a Dapr-maintained capability catalog.
6. For new inference, enforce the component's [model routing boundary](#component-owned-model-routing) and bind its configured options before cache access or provider work. Discovery uses the same model boundary with its own authorization rules below; existing job/artifact access retains original bindings.

The adapter receives the effective model on the request, never by mutating a shared component/client's default. Concurrent requests for different models must remain isolated. Unknown models or models incompatible with the requested task fail at the provider unless the adapter already knows they cannot work; no automatic model substitution is permitted. Component credentials, endpoint, and access scope remain operator-owned and are not changed by a model override.

For adopting cacheable operations, cache keys include the effective model, so implicit-default and explicit-same-model requests can share an entry while different models cannot. Changing defaults takes effect on subsequent requests, not in-flight invocations. Model aliases and provider routing can change without a Dapr release; operators control cache lifetime/namespace and may disable caching for such cases. When returned, `model` is the provider-reported model identity, falling back to the effective requested identity if the provider supplies none; this Alpha does not require discovery of an immutable model revision.

Operation handles and artifact IDs remain bound to the invocation that created them. Get/cancel/read requests do not accept a replacement model, and a later default change must not retarget an existing operation. Opaque continuation state remains provider/model-bound where required; changing the request model does not authorize discarding incompatible state.

### Component-owned model routing

All six building blocks adopt an optional component metadata entry named `modelRouting`: a JSON object keyed by exact model IDs, with a JSON object of adapter-specific routing settings per model. Parse values as structured JSON, equivalent to `map<string, google.protobuf.Struct>` at the runtime boundary, not a string-to-string map. Component-facing types use equivalent JSON-compatible values without importing runtime-generated protos. This is operator configuration, not a new public request field.

- Omitted `modelRouting` preserves dynamic model selection under `defaultModel`, `disableModelOverride`, and access policies. When present, only listed models may be selected for new inference or discovery; an empty object permits none. Unlisted selections fail with `ERR_INFERENCE_MODEL_NOT_CONFIGURED` (HTTP `400` / gRPC `InvalidArgument`) before provider/cache work. A configured default must be listed. Empty/duplicate model IDs, non-object values, and malformed JSON are configuration errors; reject duplicate keys rather than silently taking the last value.
- Each adapter defines accepted keys, types, values, and applicable execution modes. An empty per-model object imposes only the model boundary. Known unsupported settings fail initialization; restrictions discoverable only upstream fail explicitly at invocation. Do not ignore settings or infer that compatible API syntax implies identical routing support.
- Resolve a per-invocation immutable snapshot and pass it internally to the adapter; never mutate a shared client's model/options. A valid configuration update affects new work. Invalid updates retain the last valid configuration or fail activation, never remove restrictions. Existing batches/media jobs preserve their original model/options/configuration binding, and get/cancel/read still use current access policy without requiring the model to remain configured for new work.
- Application requests retain typed generation parameters but expose no routing/filter override. `metadata` remains application tagging and cannot inject provider request fields. Component type fixes the integration, credentials and service URL; per-model settings cannot replace them, choose another Dapr component, or introduce cross-model fallback. An aggregator can select underlying serving providers only within its configured restrictions.
- Distinguish enforceable filters from best-effort preferences. A supported quantization allowlist restricts eligible deployments; throughput sorting expresses preference, not a performance guarantee. Provider-native fallback must preserve hard filters or fail when no eligible deployment exists. Operators must use supported hard controls for actual bounds.

For an illustrative `completion.openrouter` component, this metadata could accompany its credentials and a matching `defaultModel`:

```yaml
- name: modelRouting
  value: |
    {
      "organization/model-a": {
        "quantizations": ["int8", "fp16"],
        "sort": "throughput"
      },
      "organization/model-b": {
        "sort": "price"
      }
    }
```

The adapter maps each entry to its native routing object. These keys are examples for a supporting router, not mandatory settings for OpenAI, Anthropic, Ollama, or any sibling provider. Ollama selecting an installed quantized model variant is not equivalent to routing among deployments by quantization. Similarly, embedding model weight quantization does not change the Embeddings API's float32 output contract. The shared structure does not promise universal price limits, quantization filters, or throughput controls.

For cacheable sibling invocations, include the canonical effective routing settings and relevant configuration identity in the cache key. Validate them before cache lookup, and retain the same snapshot across supported retries. Discovery, like Completion itself, bypasses response caching. No automatic provider/model substitution is added to satisfy a configured option.

### Model discovery and pricing

Applications can query provider catalogs and prices through Dapr without installing a provider SDK or receiving provider credentials. Each building block exposes its own `List<Purpose>ModelsAlpha1` and `Get<Purpose>ModelMetadataAlpha1` RPCs through its own registry; shared wire values and internal normalization helpers do not introduce a global catalog service or cross-component fan-out. These APIs are separate from Dapr's runtime `GetMetadata` endpoint.

All purposes bind `GET .../{name}/models` and `GET .../{name}/models/metadata?model={model}`. Model IDs are query values, URL-encoded without treating `/` or `:` as route segments. HTTP returns `200` with protobuf JSON; `pageSize`/`pageToken` map to the fields below. Metadata requests may omit `model` to use the component default; explicit model selection follows the existing override lock and component model boundary. Listing requires no default but returns only IDs within that boundary (only the default when overrides are locked). Listing describes discoverable models, not a promise that this application can invoke them.

#### Discovery component boundary

Each purpose's components-contrib package declares these optional interfaces, with purpose-scoped request/result types mirroring the shared wire values:

```go
type ModelLister interface {
    ListModels(ctx context.Context, req *ModelListRequest) (*ModelList, error)
}

type ModelMetadataGetter interface {
    GetModelMetadata(ctx context.Context, req *ModelMetadataRequest) (*ModelMetadata, error)
}
```

Adapters own authenticated provider catalog/metadata requests, native pagination, and field normalization. They expose the catalog relevant to their purpose, excluding known-incompatible models without treating unknown capabilities as unsupported. The runtime owns component selection, configured model restrictions, authorization, safe pagination tokens, and attaching the effective non-secret routing options. No catalog snapshot is a required inference preflight or a release-bound model manifest. Adapters may support listing, metadata retrieval, or both; unsupported operations return `ERR_INFERENCE_DISCOVERY_UNSUPPORTED` (HTTP `501` / gRPC `Unimplemented`), not an empty successful catalog. Provider-native lookup of a nonexistent model returns `ERR_INFERENCE_MODEL_NOT_FOUND` (HTTP `404` / gRPC `NotFound`); authentication, throttling, and upstream failures remain distinct mapped errors.

#### Discovery wire reference

Define these shared values once in `inference_common.proto`.

<details>
<summary>Model catalog, metadata, deployment, and price schemas</summary>

```protobuf
message InferenceListModelsRequestAlpha1 {
  string name = 1;
  uint32 page_size = 2;
  string page_token = 3;
}

message InferenceListModelsResponseAlpha1 {
  repeated InferenceModelSummaryAlpha1 models = 1;
  string next_page_token = 2;
  string source = 3;
  google.protobuf.Timestamp observed_at = 4;
}

message InferenceModelSummaryAlpha1 {
  string model = 1;
  optional string display_name = 2;
  optional string description = 3;
}

message InferenceGetModelMetadataRequestAlpha1 {
  string name = 1;
  optional string model = 2;
  uint32 page_size = 3;
  string page_token = 4;
}

message InferenceModelMetadataAlpha1 {
  InferenceModelSummaryAlpha1 model = 1;
  string source = 2;
  google.protobuf.Timestamp observed_at = 3;
  optional string revision = 4;
  repeated string input_modalities = 5;
  repeated string output_modalities = 6;
  optional uint64 context_window_tokens = 7;
  google.protobuf.Struct capabilities = 8;
  repeated InferenceModelPriceAlpha1 prices = 9;
  repeated InferenceModelDeploymentAlpha1 deployments = 10;
  string next_page_token = 11;
  google.protobuf.Struct effective_routing_options = 12;
  google.protobuf.Struct routing_options_schema = 13;
  google.protobuf.Struct provider_metadata = 14;
}

message InferenceModelDeploymentAlpha1 {
  string id = 1;
  string provider = 2;
  optional string quantization = 3;
  repeated InferenceModelPriceAlpha1 prices = 4;
  google.protobuf.Struct capabilities = 5;
  google.protobuf.Struct provider_metadata = 6;
}

message InferenceModelPriceAlpha1 {
  string amount = 1;
  string currency = 2;
  string unit = 3;
  uint64 unit_quantity = 4;
  google.protobuf.Struct conditions = 5;
}
```

</details>

#### Metadata meaning and availability

`source` identifies the provider/integration supplying the catalog, and `observed_at` is the runtime's retrieval timestamp, not a claim that the provider just refreshed its data. Metadata `model.model` retains the queried selection identity; a provider-reported revision is separate. Unknown scalar facts are absent, not zero/false; empty prices/modalities/deployments mean none reported, not free service or proof of no support. In extensible `capabilities`, absent means unknown and an explicit `false` means reported unsupported. Adapters document their keys and coverage; metadata is advisory, not a replacement for runtime validation.

Require nonempty model IDs and source, a valid observation timestamp, and nonempty deployment IDs/provider identities when deployment records are supplied. Reject duplicate model/deployment IDs within a page. Present context-window sizes must be positive; reported price records require valid decimal amounts, nonempty currency/unit, and positive quantities. Malformed records fail with `ERR_INFERENCE_METADATA_INVALID` (HTTP `502` / gRPC `Internal`), not a partially successful page with discarded facts.

Prices are reported rates, not inferred bill amounts. `amount` is a nonnegative decimal string (not binary floating point), `currency` is an explicit currency identifier such as `USD`, and positive `unit_quantity` specifies the denominator. For example, amount `"0.15"`, unit `"input_token"`, quantity `1000000` means USD 0.15 per million input tokens. Units distinguish input/output/cached tokens, images, video seconds, requests, or documented provider-specific billing units. Conditions preserve applicable resolution, context tiers, region, batch discounts, or other qualifications. Endpoint-specific prices stay attached to that deployment; do not flatten different currencies, units, or tiers into a single cheapest price. Unrepresentable pricing stays in documented provider metadata rather than inventing a normalized rate. Missing price information is unknown, never zero.

Deployment metadata describes advertised serving choices, not the deployment used for a completed generation. Quantization is reported only when known; do not derive it from routing preferences or treat it as a universal ordered quality level. `effective_routing_options` contains the operator-selected non-secret settings, and `routing_options_schema`, when supplied by the adapter, describes supported configuration keys using JSON Schema. These are read-only information, not request-side controls. Provider properties and advertised deployments are distinct from Dapr-enforced configuration; only claim a deployment satisfies filters when the adapter can establish that fact.

Provider metadata is a documented, sanitized JSON extension, not a dump of provider responses, credentials, private service URLs, or internal configuration. Pricing/capability discovery is a snapshot and makes no availability, execution-precision, monetary ceiling, or final-invoice guarantee. Do not copy metadata into completion results as confirmed execution information. New provider models remain discoverable without a Dapr release, subject to operator model boundaries and application authorization.

#### Discovery pagination, authorization, and runtime behavior

`page_size = 0` selects the documented adapter default; a positive value bounds returned models or deployments, not an invitation to exceed existing message/provider limits. Metadata pagination applies to `deployments`; repeat the model-level fields on each page. Preserve explicit errors rather than truncating a single oversized record. Adapters normalize one page at a time; non-paginated upstream arrays must be decoded incrementally rather than materializing a whole catalog just to produce a page. A provider unable to honor that contract fails explicitly. There is no new inference-specific catalog-size cap or guaranteed stable snapshot across pages.

The runtime returns opaque, integrity-protected continuation tokens bound to the purpose, component/configuration, application/namespace, and query, including model and page size. Protect private cursor contents with authenticated encryption or a recoverable server-side binding, not merely a signed readable token. Never expose raw provider cursors containing private information. Reauthorize every page; invalid, cross-scope, or obsolete tokens fail with `ERR_INFERENCE_DISCOVERY_PAGE_TOKEN_INVALID` (HTTP `400` / gRPC `InvalidArgument`). A page may contain no authorized models and still have a continuation token; clients finish only when the token is empty. Do not expose unfiltered totals or denied model identities in pagination metadata.

Each purpose's access policy adds exact-model `listModels` and `getModelMetadata` operations. These grants do not imply generation access, and a generation grant does not imply discovery. Before any catalog request, enforce component scoping and require at least one `listModels` grant intersecting the configured model boundary when an applicable policy exists; otherwise return the usual access-denied error without provider traffic. Filter each returned model under that call's policy snapshot. Before a model metadata request, resolve and authorize the exact model with `getModelMetadata`. No applicable policy retains the existing open semantics under other controls. A policy may deliberately grant discovery for a model without granting inference.

Discovery calls use component resiliency, deadlines, and end-to-end cancellation of the active provider lookup. They never start generation, poll independently, install provider SDKs in the application, or emulate missing catalogs from configured model names. All six purposes bypass runtime/adapter response caching for discovery in this Alpha, even where their inference responses are cacheable; return source freshness rather than silently serving stale prices. Any future metadata-cache policy must be explicit and independent of generation-response eligibility.

#### Shared routing and discovery conformance

Each adopting proposal covers omitted/empty/populated model boundaries, default/override interactions, duplicate/invalid configuration, unsupported keys and execution modes, attempted request-metadata overrides, configuration updates, and concurrent per-model isolation. Cacheable invocations must not reuse entries under changed routing settings; native jobs retain their original bindings. Provider fixtures distinguish enforceable filters from preferences and quantized variants from routing support.

Discovery fixtures cover native listing/lookup without generation, all optional-interface combinations, independent discovery/inference permissions, filtering before exposure, empty filtered pages with continuation, invalid/cross-scope tokens, policy/configuration changes between pages, and catalogs larger than one transport message. Preserve decimal pricing, currencies, denominators, tier conditions, endpoint attribution, missing values, and unknown capabilities. Exercise provider errors, cancellation, no response-cache reuse, safe extension fields, and SDK page iteration without provider packages or implicit full-catalog materialization.

### Model access policies

Ship optional model-use governance with each building block's initial API, as requested in the discussion on dapr/proposals#108. Reuse the **pattern**, not the Workflow policy resource itself: namespaced Kubernetes CRDs, equivalent self-hosted YAML resources, top-level application `scopes`, and pure allow-list rules. These new resource kinds are proposed here; the existing `WorkflowAccessPolicy` does not already authorize inference.

| Building block | Proposed resource kind | Allowed operation names |
| --- | --- | --- |
| Completion | `CompletionAccessPolicy` | `complete`, `readArtifact`, `submitBatch`, `getBatch`, `cancelBatch`, `readBatchResults`, `listModels`, `getModelMetadata` |
| Decisions | `DecisionsAccessPolicy` | `decide`, `listModels`, `getModelMetadata` |
| Embeddings | `EmbeddingsAccessPolicy` | `embed`, `listModels`, `getModelMetadata` |
| Reranking | `RerankingAccessPolicy` | `rerank`, `listModels`, `getModelMetadata` |
| Image Generation | `ImageGenerationAccessPolicy` | `generateImage`, `getOperation`, `cancelOperation`, `readArtifact`, `listModels`, `getModelMetadata` |
| Video Generation | `VideoGenerationAccessPolicy` | `generateVideo`, `getOperation`, `cancelOperation`, `readArtifact`, `listModels`, `getModelMetadata` |

Each resource applies only to its own building block's component registry. A shared evaluator is an internal implementation detail, not a family-wide component, request dispatcher, or model capability interface. Each sibling adopts this schema/semantics and owns its resource kind; approval of Completion does not require the other kinds to ship.

```yaml
apiVersion: dapr.io/v1alpha1
kind: CompletionAccessPolicy
metadata:
  name: assistant-model-access
  namespace: production
scopes:
  - assistant-app
spec:
  rules:
    - components:
        - name: chat
          operations: [complete, readArtifact]
          models: ["gpt-4o", "gpt-4o-mini"]
```

Deploy the component and policy in the same namespace and configure the sidecar's App ID to match the intended scope. This permits `assistant-app` in `production` to use only these two models through `chat` for the listed operations. The example's `defaultModel: gpt-4o` is allowed, and a request may select `gpt-4o-mini` while overrides are enabled. Any other model is denied even if the provider credentials can access it. Use separate rules to grant different model sets for different operations.

Completion's `complete` grant covers the single streaming API for both request delivery modes; explicit transcript persistence additionally requires the operator's artifact-store configuration. Reading any resulting artifact is separately governed by `readArtifact`. There are no separate unary, stream, or transcript-generation policy operations.

The example grants interactive completion only. Grant `submitBatch`, `getBatch`, `cancelBatch`, and `readBatchResults` explicitly to permit the corresponding [native batch operations](#batch-governance-and-shared-runtime-behavior). Grant `listModels` and `getModelMetadata` explicitly for discovery. These operations are independently model-scoped and do not inherit a `complete` grant.

#### Schema and matching

- `metadata.namespace` selects the deployment namespace. Top-level `scopes` lists exact App IDs whose sidecars load the policy; omitted/empty scopes apply to all applications in that namespace. Self-hosted mode follows the configured runtime namespace/App ID.
- `spec.rules` may be omitted/empty to express deny-all once this policy applies. Each supplied rule requires a nonempty `components` list. Each component entry requires a nonempty `name`, `operations` list, and `models` list.
- Component names and model identifiers use exact, case-sensitive matching in this Alpha, including provider-qualified names containing `/` or `:`. Unlike Workflow's name globbing, no model/component globs or implicit wildcard grants are introduced here. Operation names must belong to this policy kind; unknown operations, malformed entries, or empty model identifiers are configuration errors.
- Rules and applicable policies combine as a union of grants: one entry must match the component, operation, and effective model together. Do not combine a model match from one entry with an operation match from another. There are no deny rules or precedence ordering.

#### Authorization semantics

1. With no applicable policy of this kind loaded for the application, model access remains open, subject to normal component scoping, provider permissions, and `disableModelOverride`. There is no mandatory allowlist for simple deployments.
2. With any applicable policy loaded, default-deny all operations of that building block unless an entry grants the exact component/operation/model combination. A policy for Completion does not change Embeddings access, and a policy scoped to another application does not activate deny-by-default here.
3. Enforce on the sidecar invoking the provider, using its configured application identity and normal API authentication. **Local/self calls are not exempt.** Workflow's self-call bypass would defeat model governance because ordinary inference calls are local app-to-sidecar calls. No caller-supplied App ID, request metadata, or policy name may select or bypass the policy.
4. These APIs do not introduce cross-app inference routing. Workflow's cross-app `callers`, mTLS identity propagation, and history `requires` fields are not imported into this schema. If cross-app invocation is proposed later, its authenticated identity semantics require a separate design; do not trust an App ID header as a substitute.
5. Resolve and authorize before response-cache lookup, provider submission/upload, or output-stream headers. A denial returns `ERR_INFERENCE_ACCESS_DENIED` (HTTP `403` / gRPC `PermissionDenied`) without provider traffic or a cached result. Authorization is evaluated on every call and is never itself response-cached. A cache hit, native idempotency key, default model, internal omni-model dispatcher, or compatibility adapter cannot bypass it.
6. Invalid policies must not silently become "no policies." Surface configuration errors and refuse affected initial activation; on an invalid update retain the last valid policy set rather than opening access. Publish valid policy sets atomically. Explicit removal of the last applicable policy restores the documented open default; a load failure does not. Use a consistent policy snapshot for dispatch, and reauthorize before any new provider retry. This does not promise immediate termination of already-running inference when a policy changes.

#### Existing operation/artifact handles

Get/cancel/read APIs authorize their purpose-specific operation against the **original effective model**, not today's component default or an untrusted request field. Preserve that binding with the component/application scope in trusted runtime metadata or an authenticated opaque handle across restarts/replicas; provider-reported aliases do not replace the authorized selection.

Check application/component/operation access before resolving a handle, then resolve only the trusted binding metadata needed for model authorization. Enforce the full model grant before provider status/cancellation calls, response-cache lookup, or artifact-byte access. Missing or unverifiable bindings fail explicitly rather than falling back to an allowed default; backend lookup failures remain errors. Thus a revoked read grant cannot be bypassed by an old artifact ID or cached operation snapshot.

Artifact references used as inference inputs require the same `readArtifact` authorization against the source binding, in addition to authorization for the new inference operation/model. Resolve and authorize those references before response-cache access or provider work; a generation grant must not implicitly grant access to existing artifacts.

Automatic cancellation/cleanup of an already-authorized invocation retains its original authorization context and must still reach the provider if policy changes; it is not a new inference call. Explicit later cancellation requests follow current `cancelOperation` grants for media jobs or `cancelBatch` grants for completion batches. Operators may preserve cancellation permission for an old model while removing permission to start new generation with it.

An allowlist controls the selected model identifier, not the immutable internals of a provider alias or routing service, and does not guarantee a monetary ceiling. Unknown-but-authorized model identifiers are still validated by the provider; no release-bound model inventory is added. Normal provider credentials, component scoping, and access controls remain additional restrictions. Policy configuration is operator-owned and is separate from cache TTL/eligibility and provider data-retention policy.

### Resiliency, processing, and observability

- **Resiliency:** Use the addressed component's outbound policy, with operation-specific overrides where applicable. Generation retries are off by default; neither a seed nor an endpoint name guarantees deterministic output, duplicate-billing protection, or retry safety. Completion retries stop before the first semantic event; artifact reads stop retrying before the descriptor/HTTP headers. Cancellation never triggers retry and failed reads never regenerate output.
- **Processing:** Each block defines its own `scrub_pii` surfaces. Completion scrubs instructions and eligible history text before invocation; output scrubbing follows the [Completion contract](#completion-contract), including buffering and incompatible signed-state errors. Shared extras do not add fields or output-scrubbing promises to other blocks.
- **Optional caching:** Apply the shared controls below only when enabled for that component. A shared physical backend is allowed, but each building block owns eligibility and configuration.
- **Model access:** Apply the purpose-specific access-policy resource after effective-model resolution and before response-cache access or provider work.
- **Observability:** Common helpers may emit `dapr_inference_count` and `dapr_inference_latency_ms` with bounded `(building_block, component, operation, status)` labels. Completion additionally records first-content latency, completion latency, event/coalescing counts, and buffered delivery. Artifact transfer records bytes, transfer latency, backpressure, integrity failures, and cancellation separately. Prompts, tool arguments, opaque state, model IDs, provider URLs, and artifact IDs are not metric labels.
- **Validation:** Validate request shape, known integration restrictions, and configuration locally; unknown per-model restrictions fail through mapped provider errors. Do not silently drop unsupported options. Normal authorization and operator-owned data policy apply independently to every component.

### Provider-native idempotency

Model-invoking requests expose optional `idempotency_key`. Adapters implement forwarding and validation case by case where the provider's corresponding operation supports it. A supplied key must be nonempty; an adapter that cannot honor it fails explicitly with `ERR_INFERENCE_IDEMPOTENCY_UNSUPPORTED` (HTTP `400` / gRPC `InvalidArgument`) instead of ignoring it. Support is an integration detail, not a new per-model manifest requirement.

The same key is retained across authorized retries of the same request. The adapter documents the provider's native scope, expiry, and payload-conflict behavior; Dapr does not extend those guarantees, synthesize a generic deduplication ledger, or promise exactly-once computation/billing. In particular, a seed is not an idempotency key. A request containing an idempotency key bypasses runtime response-cache reads and writes, so a cache cannot hide the provider's key/payload conflict checks or return a result from another native operation. Unsupported-key validation happens before inference; no fallback to an unkeyed call is allowed.

### End-to-end cancellation

When an application cancels the token/task/signal attached to an active inference call, the SDK, runtime, and provider adapter MUST propagate it end to end:

1. The SDK cancels the underlying gRPC call or HTTP request, not just a local wait, iterator, or wrapper task. In .NET, `CancellationToken` must reach the transport; using only a cancellation-aware `WaitAsync` around an uncanceled call is insufficient. Python task cancellation and TypeScript `AbortSignal` must likewise abort the active transport. Disposing a live stream early follows the same path.
2. The runtime cancels the invocation context, stops scheduling retries and provider work, and unblocks channel sends/reads. It retains responsibility for canceling the associated upstream operation even after the application connection has gone away.
3. The adapter passes invocation cancellation into the actual provider client and aborts/closes its upstream request or response stream. If the provider uses a background job or requires an explicit stop/cancel API, the adapter MUST invoke that API for invocation-owned work as well. Closing only the application's connection or stopping event consumption is not sufficient.
4. Explicit provider cancellation and staging cleanup run under a runtime-owned, supervised cleanup context bounded by the component's `cleanupTimeout` (default 30 s), preserving the necessary authorization. They MUST NOT use the already-canceled application context, which would prevent the cancellation request itself from reaching the provider. Cleanup may outlive the client connection, but cannot run indefinitely; shutdown, timeout, and cleanup failures remain observable.
5. Suppress successful stream completion and unpublished artifact publication when cancellation wins the completion race. Stop/abort storage writes and clean up unpublished output. Do not treat caller cancellation as a retryable generation error or submit a replacement job.

The cancellation scope is the work initiated by that call:

| Active call | Work cancellation must address |
| --- | --- |
| Interactive Completion calls, or an adopting block's active generation/evaluation call | The active provider inference invocation; artifact-backed completion also aborts its staging write |
| Completion batch submission before handoff | Input upload/staging and native submission; cancel any newly created batch where supported |
| Completion batch status/result read after handoff | Only the active lookup/download; cancel the independent job through `CancelCompletionBatchAlpha1` |
| Completion batch cancellation | Request native job cancellation; the acknowledgment is not confirmed terminal cancellation or reversal of completed items |
| Any purpose's model listing/metadata lookup | The active provider catalog/metadata request, never an inference job |
| An adopting block's submission before asynchronous handoff | Active generation/submission and any provider job already created for that invocation |
| Purpose-specific operation lookup | The upstream status lookup, not the already-independent generation job |
| Artifact read / SDK event-artifact reader | The upstream download/storage read, not the generation that produced an existing artifact |
| Purpose-specific operation cancellation RPC | A remote job-cancellation request; canceling this RPC does not undo a cancellation already accepted by the provider |

For an adopting block's asynchronous submission, adapters track job IDs as soon as they become available. If cancellation races with job creation, they must cancel an accepted job once identified, or reconcile it through a provider-supported lookup/correlation mechanism within the cleanup deadline. If the provider accepted work but offers neither a returned handle nor a reconciliation mechanism, record the cancellation outcome as unknown; do not assert that work stopped. Ordinary request-context disposal after successfully returning a `PENDING`/`RUNNING` operation is an intentional handoff, not a reason to cancel that job. After handoff, the application uses the purpose-specific operation cancellation RPC with the operation ID; a token on a later poll does not implicitly own the job.

The purpose-specific operation cancellation RPC invokes the provider's actual cancellation mechanism. An empty successful RPC response means the cancellation was accepted or the operation was already canceled, not that computation has necessarily stopped. Continue exposing the provider's actual operation status until it confirms `CANCELED`; accepted-but-pending cancellation must not be presented as terminal cancellation. Repeated cancellation requests must not submit additional generation. If completion wins the race, preserve the actual `SUCCEEDED`/`FAILED` outcome; cancellation does not roll back completed work or delete published artifacts. An already-terminal non-canceled operation returns a failed-precondition error if it can no longer be canceled. Providers without a remote cancellation mechanism return `ERR_INFERENCE_CANCELLATION_UNSUPPORTED` (HTTP `501` / gRPC `Unimplemented`) for their cancellation endpoint, never successful local-only cancellation.

**Provider confirmation is not implied by caller cancellation.** Some providers contractually stop inference when the upstream connection is aborted; others expose a job-cancel API; some offer only best-effort abort with no confirmation. Every adapter must perform all supported upstream cancellation actions and document which guarantee its configured endpoint provides. Unconfirmed cancellation is recorded as such, not silently promoted to provider-stopped status. A local `OperationCanceledException`, task cancellation, or aborted fetch reports the caller's canceled operation; it is not proof that remote computation or billing has stopped.

Record cancellation requested, upstream abort attempted, native cancellation accepted, provider-confirmed cancellation, unsupported cancellation, and cleanup timeout/failure distinctly in structured logs/traces and bounded-label metrics. After the caller disconnects these signals may be the only observable outcome; explicit cancellation callers receive provider errors through the normal Dapr error path. Cancellation support does not promise zero additional tokens/cost, instantaneous interruption, or reversal of work completed before the provider acted.

### Artifact transfer

Completion owns `ReadCompletionArtifactAlpha1` and its HTTP route. Other proposals may independently bind the same value types and transfer semantics to their own artifact-read RPC/route without depending on each other's implementation.

```protobuf
message InferenceArtifactAlpha1 {
  string mime_type = 1;
  string artifact_id = 2;
  optional uint64 size_bytes = 3;
  optional string revised_prompt = 4;
  optional bytes sha256 = 5;
}

message InferenceArtifactReferenceAlpha1 {
  string name = 1;
  string artifact_id = 2;
}

message InferenceReadArtifactRequestAlpha1 {
  string name = 1;
  string artifact_id = 2;
}

message InferenceReadArtifactResponseAlpha1 {
  oneof part {
    InferenceArtifactAlpha1 artifact = 1;
    common.v1.StreamPayload payload = 2;
    InferenceArtifactReadCompletedAlpha1 completed = 3;
  }
}

message InferenceArtifactReadCompletedAlpha1 {
  uint64 bytes_read = 1;
  bytes sha256 = 2;
}
```

`inference_common.proto` imports `dapr/proto/common/v1/common.proto` for unchanged `StreamPayload`, plus `google/protobuf/timestamp.proto` for cache generation timestamps. File bytes occur only in `StreamPayload.data`, never as a whole-file field in an operation or descriptor. SHA-256 values are fixed 32-byte digests.

The component-facing read shape can be reused internally without sharing a public registry:

```go
type ArtifactReader interface {
    OpenArtifact(ctx context.Context, artifactID string) (*ArtifactRead, error)
}

type ArtifactRead struct {
    Artifact ArtifactDescriptor
    Content  io.ReadCloser
}
```

`ArtifactDescriptor` mirrors the common descriptor without importing runtime-generated protos. Each purpose-specific handler obtains its reader from the addressed component or that component's configured backing-store adapter. Completion's runtime-written transcripts use its backing-store reader; provider-generated Completion media must also provide an authorized read path. `OpenArtifact` resolves and authorizes the ID before returning, binds the source to the read context, and surfaces later failures through `Read`. The runtime always closes `Content`, including on send failure or cancellation.

#### Artifact identity and retrieval

Every returned artifact has a nonempty opaque ID, scoped to the owning building block, component, and authorized application. The ID identifies immutable content, not a model invocation to re-run or a caller-controlled URL to fetch. Artifact IDs are independent of operation IDs, so synchronously generated files and completion transcripts remain readable without a provider job handle. Repeated successful polls return the same artifacts, not newly generated files. Only successful generation/finalization exposes artifacts; a failed operation or completion invocation does not publish partial files.

The provider/component must make content retrievable after generation: either through a provider-managed immutable file/version locator, or an explicitly configured artifact backing store such as Binary Store. Provider URLs and credentials stay behind the component; reads use normal Dapr authentication and component access controls, and identifiers alone do not grant cross-application access. Locators must survive ordinary sidecar restarts and replica changes through provider-addressable handles or shared persistent storage rather than process-local dictionaries.

For providers returning only inline/base64 content, the adapter must incrementally decode and persist it to that configured store before publishing a successful result or artifact reference. Native clients that materialize an entire file must be replaced with a streaming response/parser on this path; chunking a fully allocated byte array afterwards does not meet this contract. If neither provider retrieval nor configured storage is available, fail storage/retrieval validation before starting a billable generation. No implicit unlimited in-memory spool or new mandatory storage dependency for providers with native retrieval is introduced. On persistence failure, fail the invocation and clean up unpublished partial artifacts.

`size_bytes` and `sha256` are supplied only when known; zero length is distinct from absent length. Availability and deletion remain with the provider/backing store and operator. Components may refresh internal provider URLs without changing IDs or bytes. Unknown or no-longer-available artifacts return `ERR_INFERENCE_ARTIFACT_NOT_FOUND`. A mid-read loss of the source fails the read. Cancellation of a download closes that reader but does not delete the artifact or cancel a separate generation operation.

#### gRPC byte-stream contract

1. Send exactly one `artifact` descriptor first, matching the requested ID and resolved immutable content. Then send zero or more `payload` messages and exactly one `completed`. No messages follow completion; unknown parts, duplicate descriptors, and invalid ordering fail the read.
2. Reuse `common.v1.StreamPayload` unchanged: `data` contains raw bytes and `seq` starts at zero and increases by one per payload. Chunk sizes may vary and boundaries have no file/codec semantics. Empty payloads are allowed by the shared type and do not indicate EOF. An empty artifact can emit no payloads and a completion with zero bytes.
3. Serialized payload messages must fit existing configured gRPC send/receive message limits. Chunk sizing follows the existing runtime/SDK streaming implementation, with no additional inference-specific maximum. The receiver validates ordering, counts bytes with 64-bit counters, and never concatenates the entire response by default.
4. Compute SHA-256 incrementally. `completed.bytes_read` and its 32-byte digest describe the delivered file; both must agree with the payload stream and with any advertised descriptor length/digest. A mismatch, provider read failure, premature EOF, or non-OK gRPC status is an error, not a shorter successful file.
5. Success requires completion and OK transport termination. Data already consumed cannot be retracted; SDKs expose later errors, and callers needing atomic persistence write to a temporary destination and publish it only after successful completion.

Only finalized immutable bytes are delivered: this is not Completion's text `APPEND`/`REPLACE` protocol. Byte chunks are neither UTF-8 strings nor SSE events; they are never coalesced away. There is no range, seek, resume, or automatic reconnection guarantee in this initial API.

#### HTTP binary delivery

`GET /v1.0-alpha1/completioninference/{name}/artifacts/{artifactId}` returns `200` with the artifact MIME type (or `application/octet-stream`), `X-Content-Type-Options: nosniff`, and `Cache-Control: private, no-store` by default. Send `Content-Length` only when authoritative; otherwise use HTTP streaming framing. The body is the raw file, not protobuf JSON, an SSE/base64 envelope, or a redirect exposing provider credentials. No inline flag is provided on generation or polling endpoints.

Validate authorization and open the source before committing HTTP headers. Pre-body errors use standard Dapr error responses; after headers/body are sent, abort the response/HTTP stream on failure without appending JSON to the binary data. Do not send a normal terminating frame for a failed unknown-length transfer. For known lengths, withhold the final chunk until source EOF and integrity checks succeed, so a late source error cannot appear as a completed `Content-Length` body. For a declared zero-length file, verify EOF and integrity before committing headers. A successful HTTP EOF must represent the same checked content as gRPC completion. Length/integrity failures are also logged with structured Dapr error codes, since raw HTTP cannot carry a late JSON error body.

HTTP does not expose gRPC chunk boundaries or the final protobuf digest message. SDKs validate available descriptor length/digest while reading; otherwise rely on successful HTTP framing and runtime source validation. This endpoint requires a dedicated binary streaming handler, not a unary JSON gateway. It is response-only streaming, so it does not share the bidirectional HTTP limitations discussed in the original Crypto proposal.

#### Incremental transfer and SDK mapping

Provider reads, optional backing-store writes, runtime delivery, and SDK consumption all use existing streaming flow control and backpressure. Transfer buffers do not grow with total file size. Existing gRPC message limits apply per message, not to the total artifact; this proposal adds no chunk-size cap, queue-size setting, or artifact-size quota. Underlying provider/store errors remain explicit rather than becoming truncated successful results. Avoid `ReadAll`, whole-response JSON decoding, `MemoryStream.ToArray`, and eager SDK enumeration on the default path.

The read has its own request deadline and artifact-read resiliency policy, separate from generation/polling. Retry only before the first gRPC descriptor or HTTP headers have been emitted. Never replay bytes after delivery starts, and never re-run generation to repair a failed read. Slow consumers suspend upstream reads rather than accumulating the whole file. Disconnect, deadline, early async-enumerator disposal, or stream disposal must cancel the source and release its `io.ReadCloser`, HTTP response, and gRPC call. The same rules apply when an SDK incrementally decodes a completion transcript from these bytes.

### Optional shared inference cache contract

Shared caching is an optional runtime integration, disabled by default, for suitably small, successfully finalized unary API responses only. The **proposed Centralized Cache building block in dapr/proposals#71** is an explicitly supported backend option once available, through the [integration contract below](#centralized-cache-integration); a general Dapr state store remains an alternative. Neither inference nor its caching contract depends on that proposal shipping. The selected integration needs get/set/delete behavior for versioned envelopes containing processed final results. It does not store conversation sessions, request history, intermediate events, streamed bodies, or references used as substitutes for large generated results. No streaming, transaction, locking, or expiry capability is required of every cache backend.

```protobuf
message InferenceCacheOptionsAlpha1 {
  enum Mode {
    MODE_UNSPECIFIED = 0;
    DEFAULT = 1;
    BYPASS = 2;
    REFRESH = 3;
  }
  Mode mode = 1;
}

message InferenceCacheResultAlpha1 {
  enum Status {
    STATUS_UNSPECIFIED = 0;
    HIT = 1;
    MISS = 2;
    BYPASS = 3;
  }
  Status status = 1;
  optional string reason = 2;
  google.protobuf.Timestamp generated_at = 3;
}
```

`DEFAULT`/absent mode follows operator configuration and endpoint eligibility; caching is disabled unless `cacheStore` and `cacheEnabled = true` enable it for that component. `BYPASS` neither reads nor writes. `REFRESH` skips lookup and replaces the entry only after a new eligible successful result; it is invalid for a cache-disabled component, an inherently noncacheable endpoint such as live/artifact delivery, or a request carrying an idempotency key. It is an inference-level regeneration request, not a requirement for a backend refresh/sliding-expiration API. Unknown modes are validation errors.

Cache-entry TTL and eviction are optional operator/backend policies, separate from artifact storage. Use the selected backend's supported configuration; do not require sliding/absolute expiration or invent artifact expiry to make caching work. Include configured cache policy in the namespace/fingerprint so a policy change does not reuse entries under obsolete settings. `cacheNamespace` is an operator-controlled invalidation/version boundary for deployment, model-alias, and configuration changes.

The cache integration's operator-configured small-entry byte budget applies to the complete serialized response envelope, including metadata, and must respect existing backend value-size limits. This is a cache admission policy, not a runtime/SDK response or streaming limit. A result above that budget is returned normally with cache status `BYPASS` and a diagnostic reason; do not truncate it, fail otherwise successful inference, split it across cache keys, or replace it with a cached artifact reference. Reject unusable/oversized stored entries as diagnosed misses. Measure admission without building a second unbounded serialization solely for caching.

**Keys and result semantics.** Hash a versioned canonical representation of the application/namespace, building-block identity, component identity and relevant non-secret configuration, operation, effective model and component-owned routing settings, full ordered inputs/state/history, tools, output constraints, generation parameters, request metadata, opaque continuation data, and scrubbing/processing policy. Operation lookups key on their bound operation ID rather than re-resolving today's component model default/routing settings. Preserve array order and presence distinctions that affect semantics; map key order is canonicalized. Never use component name or model name alone as a cache key. Operator cache namespaces and normal access controls prevent accidental cross-application/account reuse; raw prompts, keys, credentials, and opaque state are not logged.

Cache envelopes identify the final unary representation and original generation time. Validate endpoint eligibility before lookup: live and artifact delivery never reuse a unary entry, regardless of output size. Different building blocks, component instances, or model selections never share entries, even if they reuse an internal provider client or physical cache backend. A model alias is not a frozen revision: short TTLs, explicit namespace changes, or bypass are operator choices rather than a Dapr model catalog.

Report `HIT`, `MISS`, or `BYPASS` in the response's `cache` field. Supply diagnostic reasons for unsupported eligibility, disabled policy, oversize entries, or cache failures. Cached usage describes the **original generation**, not new provider tokens/billing; observability separates cache reads from fresh inference.

#### Completion-specific eligibility

No Completion operation is cacheable, and no Completion request or response carries `InferenceCacheOptionsAlpha1` / `InferenceCacheResultAlpha1`. The shared contract exists in this proposal because siblings adopt it by reference. For adopters, the following distinctions apply.

Other proposals define their own small-final-response eligibility while adopting these controls, isolation, and failure handling. No tool execution is cached or performed by the runtime. Media generation bypasses response caching rather than reuse large output through a cached descriptor. A small successful terminal media-operation lookup may be cached as metadata keyed by its existing operation ID. It does not cache or promise availability of the file, satisfy a new generation request, or turn the cache into a job/session store. Active, failed, and canceled snapshots bypass caching. Cancellation always bypasses caching and invalidates related cached media-operation metadata where applicable.

Idempotency-keyed requests bypass caching regardless of component policy. No background polling is introduced to populate a cache, and a cached terminal lookup never becomes an entry for the original generation request.

**Streaming and failures.** Only unary results already materialized for the caller are candidates, after final processing and cache admission. Streaming endpoints have no cache path; no cache-only buffer, artifact writer, or transcript is created for them.

If a fresh unary result in an adopting sibling proves ineligible (for example, oversized output), including after an otherwise valid `REFRESH`, return it normally with `BYPASS` and a reason. Do not repair cache failure by rerunning successful billable inference. Cache eviction has no effect on independently stored artifacts, and an artifact read never regenerates missing content.

The shared contract assumes no transaction/locking support from either a cache or state backend, so concurrent misses may perform duplicate provider work; this cache does not provide single-flight or exactly-once semantics. Failed, canceled, incomplete, or intermediate responses never become successful cache entries.

Explicitly configured cache dependency/configuration failures are surfaced at startup; an uncached deployment requires no cache/state component. For transient cache read/write failures, the default is to continue uncached with a structured warning/metric and a response diagnostic, not silently report a hit; do not retry an already successful billable generation to repair its cache entry. Unknown/corrupt envelope versions are diagnosed and treated as unusable entries. Operators own cache/provider/store retention, encryption, access, and sensitive-data policy. Dapr enforces request isolation, normal authorization, and existing scrubbing semantics without claiming to certify those external policies.

When enabled for a cacheable sibling, this shared mechanism replaces inference-adapter-specific response caches. Completion migration removes legacy `responseCacheTTL` and disables adapter response-cache layers, since the always-streamed API has no cache path. Provider-native prompt/prefix caching remains separate and follows provider/component configuration. A state-store integration can provide shared inference caching without dapr/proposals#71; do not silently substitute a process-local cache for an explicitly selected backend.

#### Centralized Cache integration

This proposal defines the optional [dapr/proposals#71](https://github.com/dapr/proposals/pull/71) backend binding adopted by the sibling proposals. For cacheable adopting blocks, `cacheStore` may reference a proposed `state.centralizedcache.<provider>` component; `cacheEnabled` and cache policy remain local to the consuming purpose-specific component. Applications keep the same inference request/result types regardless of whether the operator selects Centralized Cache or general state.

| Inference cache action | Centralized Cache mapping |
| --- | --- |
| Look up the isolated, canonical key | `GetCentralizedCacheAlpha1`; absent/expired `data` is a cache miss, not an inference error |
| Publish an eligible successful result | `SetCentralizedCacheAlpha1` with overwrite enabled, the serialized versioned envelope, and operator-configured cache-entry expiration options |
| Invalidate an unusable entry or cached terminal-operation metadata | `RemoveCentralizedCacheAlpha1`; never delete independently stored artifacts as a side effect |
| Touch an existing entry without reading its value, if needed | `RefreshCentralizedCacheAlpha1`; this renews sliding expiration only, does not regenerate inference, and does not extend an absolute expiration bound |

Use the proposed cache's get/access sliding-expiration behavior rather than issuing a redundant refresh after every hit. Inference `cache.mode = REFRESH` is deliberately different: bypass lookup, run fresh inference, and set a new eligible entry. `BYPASS` and native-idempotency-keyed requests make no cache get/set/touch calls.

The selected integration's operator policy supplies the sliding expiration and optional absolute expiration required by the proposed API, encoded as ISO 8601 durations. These govern **cache entries only**, never artifact availability or Binary Store cleanup. They are not mandatory policies for a general state backend. Unsupported or missing required backend policy fails configuration validation rather than silently dropping expiration semantics.

Store serialization/version metadata inside the cache value, not separate companion keys. The adapter maps that envelope to the finalized cache value representation; the draft's API sketches are not copied as new inference wire definitions. Store only admitted small final unary responses, including eligible terminal lookup metadata, never streamed bodies, cached generation references, or conversation/session state. Do not depend on key listing, ETags, transactions, locks, or cache-backed idempotency guarantees.

All adopting blocks apply their own eligibility rules to this binding. Conformance covers get/set/remove, access/touch sliding behavior, the absolute bound, expired misses, inference-refresh versus cache-touch, bypass, unavailable backends, and isolation between building blocks sharing one cache. Enabling this integration must not add artifact expiry, streaming caps, a required cache deployment, or a dependency on another inference provider.

## Migration and delivery

These sections define the runtime replacement and its delivery boundary. SDK responsibilities are intentionally broad; the later examples illustrate consumption without prescribing identical client APIs in every language.

### Conversation migration and independent lifecycle

1. Ship the Completion preview with one server-streaming `Complete` operation, `ReadCompletionArtifact`, and the explicit native batch submission/status/cancellation/result APIs under its purpose-specific namespace. Interactive `EVENTS` delivery requires neither a cache nor a transcript store; `ARTIFACT` delivery requires explicit storage. Native batches require an integration capable of preserving submission/processing bindings across restarts. Generated media independently requires an authorized retrieval path. SDK callers needing a materialized item result explicitly fold its content stream.
2. Deprecate `ConverseAlpha1` / `ConverseAlpha2` and freeze Conversation's preview contract. Supply a documented, feature-limited migration adapter for legacy `conversation.<provider>` implementations. Migration to `completion.<provider>` changes the component type and API; it is not a wire-compatible alias or an automatic cross-purpose registration. Map a legacy configured model to `defaultModel`; reject unsupported per-request models and lossy content conversions. Disable legacy response-cache layers rather than replaying cached responses through the new streaming API.
3. Migrate SDK and `dapr-agents` call sites to the Completion contract. Graduate Completion independently when its own conformance and deprecation criteria are met; remove preview Conversation APIs under Dapr's alpha deprecation policy.

There is no Phase 2 dependency on media generation or any sibling proposal. New AI/Inference purposes can join the documentation family without reopening Completion's purpose boundary.

### Acceptance criteria

The criteria below separate runtime delivery, retirement of the Conversation preview, and SDK integration. The normative contracts above remain applicable; SDK work is stated at the capability level rather than as a per-language implementation checklist.

#### Runtime: new functionality

- **API and component integration:** Add `completion.proto`, the adopted `inference_common.proto` types, `CompleteAlpha1`, `ReadCompletionArtifactAlpha1`, and their HTTP/SSE handlers. Register `completion.*` components and integrate selected provider adapters through `components-contrib/completion`. Both completion delivery modes use the same request and server-streaming API. Component credentials, model defaults, resiliency, and cancellation remain isolated; registering Completion does not register any sibling purpose.
- **Provider-neutral content and processing:** Normalize ordered text/media blocks, tool calls/results, provider continuation state, optional usage, and completion outcomes. Apply structured-output validation, PII processing, and the documented buffering rules, reporting `BUFFERED` where required. Known unsupported inputs/options fail locally; unknown model limitations surface as mapped provider errors without silent option loss, model substitution, or a release-bound capability catalog. Newly installed Ollama models and routing-provider models remain usable without a Dapr release. Synthetic diffusion fixtures do not constitute a shipped native diffusion adapter.
- **Streaming correctness:** Implement append/replacement negotiation, block-scoped ordering, safe coalescing, contiguous sequence numbers, marker-only text completion, and explicit terminal success/failure. Cover interleaved blocks, empty replacements, UTF-8 boundaries, malformed tool arguments, and undeclared revisions. Append-only callers never receive replacements; dual-mode providers retain append delivery for `ALLOW_REPLACE` and prefer snapshots for `PREFER_REPLACE`. A fixed provider fixture produces equivalent content through live events and stored transcripts; independent generations are not compared. Neither provisional arguments nor block completion authorize tool execution.
- **Transport behavior:** Maintain equivalent gRPC and SSE event semantics, including protobuf JSON encoding, 64-bit values, delivery metadata, pre-open errors, post-open failures, Accept validation, rejected legacy stream selectors, and keep-alives during buffering/persistence. Reference-only responses still require lifecycle events and successful transport termination. Unknown modes, missing/unexpected references, or content events in an outward transcript-reference stream fail explicitly. Initial metadata need not predict generated media.
- **Artifact publication and retrieval:** Persist requested JSONL transcripts incrementally and provide authorized byte reads for transcripts and generated media. Stored and outward streams share an invocation ID but have independent sequences; stored content uses `EVENTS` and never recursively references itself. Validate format, sequencing, length/digest, and source/transport completion, including empty content, arbitrary chunk boundaries, truncated records, and late source failures. Publication failure or cancellation cleans staging and prevents a successful terminal descriptor, even if a terminal record was staged. Artifact bindings survive ordinary restarts/replicas while the underlying content exists.
- **Bounded-memory transfer:** Demonstrate that a native append-only block can grow from 64 MiB to 5 GiB without accumulating response text in the runtime, and that a synthetic 5 GiB artifact transfers without full-file materialization. Both fixtures use synthetic generators and `io.Discard`-style sinks so they need no disk or network; the 5 GiB cases run in a nightly/scheduled conformance job with runtime RSS assertions, while per-PR CI runs the same tests at 256 MiB. Exercise backpressure, 64-bit counters, slow consumers, blocked sends, and existing gRPC message limits without adding inference-specific stream, chunk, queue, or total-size caps. Unary-provider fallback splits text into valid events; atomic non-text values still obey per-message limits. Cover incremental provider base64 decoding into `artifactStore` and reject known unsupported retrieval/storage before generation. Document the existing processing and unary-provider buffering exceptions rather than claiming all providers stream natively.
- **Model selection and access governance:** Resolve request models, component defaults, and override locks before dispatch, including absent defaults, empty models, same/different explicit models, and concurrent selections. Ship `CompletionAccessPolicy` schema/CRD, operator distribution/RBAC, self-hosted loading, and runtime enforcement. Cover scoped/unscoped policies, no-policy openness, empty-rule denial, union-of-grants matching, exact model identifiers, isolation, and malformed-policy handling. Local calls are not exempt. Authorize artifact downloads and artifact inputs against trusted original-model bindings after policy/default changes; missing bindings cannot fall back to current defaults. Denied generation starts no provider work and opens no successful stream.
- **Resiliency, cancellation, and idempotency:** Apply component resiliency and observability without resetting the overall timeout as events arrive. Retry only before semantic output is emitted or persisted, and never retry after cancellation. Propagate cancellation to provider I/O and perform supported native cancellation of invocation-owned jobs, including job-ID and publication races, under `cleanupTimeout`. Conformance observes the provider boundary: confirmed cancellation and best-effort transport abort are distinguished, and cleanup remains authorized after policy revocation. Preserve explicit idempotency keys through supported retries, surface native conflicts as `ERR_INFERENCE_IDEMPOTENCY_CONFLICT`, and reject unsupported keys instead of ignoring them. Provider support varies; neither local cancellation nor an idempotency key implies universally stopped computation, zero additional billing, or exactly-once execution.
- **Error mapping:** Every adapter maps native errors onto the [error code reference](#error-code-reference); conformance fixtures assert the HTTP status, gRPC code, and `errorCode` for each row, pre-open and post-open, and reject any undocumented public code.
- **Native batch execution:** Add client-streamed/NDJSON submission, status, explicit cancellation, and streamed result retrieval. Provider fixtures prove use of native batch endpoints, not interactive fan-out; unsupported integrations/options fail explicitly without ordinary generation. Cover empty/duplicate IDs, late invalid items, missing/miscounted finish frames, aborted uploads, model conflicts, input authorization, and no job creation before complete validation. Native file-backed fixtures process uploads/results larger than a unary transport message incrementally; document permitted per-item buffering and metadata growth without whole-batch materialization. Batch-only components need not expose interactive completion.
- **Model routing and discovery:** Deliver component-owned per-model settings, `ListCompletionModelsAlpha1`, `GetCompletionModelMetadataAlpha1`, their HTTP handlers, and optional component discovery interfaces. Apply [shared routing/discovery conformance](#shared-routing-and-discovery-conformance), including native batch option compatibility and durable configuration snapshots. No request-level routing filters or provider SDK dependency in applications is introduced.
- **Batch result and recovery conformance:** Cover out-of-order results, separate success/error files, item failures after provisional content, missing/duplicate/unknown IDs, terminal count consistency, refusals/tool-result outcomes, and result-source corruption before/after headers. Every item has a final disposition or retrieval fails explicitly. Exercise all native status mappings, provider expiration, cancellation before/after handoff, accepted-but-pending cancellation, partial successful items after cancellation, lost acknowledgments, native submission-key behavior, and unavailable results without regeneration. Restart/replica fixtures preserve job/model/item/processing bindings; current access policies govern all control/read paths. No background Dapr worker or price guarantee is implied.
- **Configuration and scope boundaries:** Every runtime-owned key in the [component metadata reference](#component-metadata-reference) is validated at load, including rejection of cache keys and `responseCacheTTL` on `completion.*` components and detection of provider-defined keys that collide with reserved names. No Completion path creates cache-only buffers or cache entries. Shared cache/state helpers are defined for eligible sibling operations only. The [non-goals](#non-goals) remain excluded; only provider-owned native batch jobs are added.

Shared-extra conformance for adopting siblings remains owned by their proposals: test optional cache backends, isolation, native-idempotency bypass, historical usage, diagnosed failures, and serialized envelopes below/at/above the cache admission budget. Job-based adopters also validate get/cancel authorization against original-model bindings, including cached terminal metadata. Those endpoints and their provider integrations are not prerequisites for Completion approval or delivery.

#### Runtime: deprecations and migration

- Deprecate and freeze `ConverseAlpha1` / `ConverseAlpha2` as described in [Conversation migration](#conversation-migration-and-independent-lifecycle). Remove the preview APIs under Dapr's alpha deprecation policy; this proposal is an intentional replacement, not a wire-compatible rename.
- Provide the documented, feature-limited migration path from `conversation.<provider>` to `completion.<provider>`. Map the legacy configured model to `defaultModel`; reject lossy content conversions and unsupported request-model overrides rather than silently adapting them. Do not automatically register legacy components for multiple inference purposes.
- Remove legacy Completion response-cache configuration and disable adapter response-cache layers on the replacement path. They must not re-enable caching behind the always-streamed API; provider-native prompt/prefix caching remains separate. Completion graduation and Conversation retirement do not depend on approval of the sibling building blocks.

#### SDKs: broad integration scope

- Add purpose-specific Completion clients and generated protocol bindings, with ergonomic request/result models and access to model selection, delivery preferences, stream options, and explicit idempotency keys.
- Accommodate the supported text and media input forms, ordered history, tool calls/results, and opaque continuation state without lossy conversions. Surface unsupported provider/model capabilities as errors rather than promising uniform feature support.
- Expose the common streamed response for content and artifact-reference delivery. Support typed event consumption, append/replacement handling, optional explicit folding, and artifact/transcript reads without implicit downloads or whole-response buffering.
- Support incremental batch submission with a validated finish frame, explicit status/cancellation calls, and correlated per-item result iteration. Preserve item IDs, native item errors, and provider-reported usage; do not automatically fan out, resubmit failed items, or interpret completed result retrieval as success of every item. Batch cancellation after handoff is distinct from canceling a polling/read operation.
- Expose model listing and metadata/pricing retrieval with explicit page iteration, cancellation, optional/unknown values, and discovery errors. Surface read-only effective model settings without request-side routing overrides or provider SDK dependencies.
- Support application cancellation, deadlines, and early stream disposal through each SDK's normal mechanisms, propagating them into the runtime. Actual provider work cancellation depends on the integration; SDKs must surface limitations and outcomes rather than equate a canceled local operation with confirmed provider cancellation.
- Support incremental artifact consumption and surface protocol, integrity, and late transport failures. A received block or artifact reference is not successful completion until the stream and transport finish successfully. On failure, expose any append-only prefix solely through the error's `partial_output` member, never through the success path. Follow the detailed streaming/artifact contracts above without prescribing identical helper APIs in every language.
- Map every `ERR_INFERENCE_*` code to a typed SDK error (or error-code enum) so callers can branch on the code without string matching.
- Provide migration guidance and representative examples for existing Conversation callers, including `dapr-agents`, and align SDK deprecations with the runtime transition.

### Implementation checklist

- [ ] Runtime (new): prefixed wire types, streaming/read RPCs and HTTP handlers, component registration, and selected provider adapters
- [ ] Runtime (new): event normalization, incremental artifacts, processing, flow control, resiliency, provider-reaching cancellation, and native idempotency
- [ ] Runtime (new): model resolution, access-policy resources/loading/enforcement, component metadata validation, error-code mapping, and runtime/provider conformance
- [ ] Runtime (new): native Completion batch routes/RPCs, adapter interface, staged-input/manifest bindings, per-item result normalization, batch access-policy operations, and lifecycle conformance
- [ ] Runtime (new): per-model component settings, discovery routes/RPCs/interfaces, pricing normalization, pagination, discovery authorization, and shared conformance
- [ ] Runtime (deprecated): Conversation API annotations, documented component/model migration, and removal under the alpha deprecation policy
- [ ] Runtime (deprecated): legacy response-cache configuration/layers disabled on the replacement path
- [ ] SDKs: broad Completion client, input/output, streaming/artifact, cancellation, error-handling, and migration support described above

## Illustrative SDK appendix

These examples illustrate the runtime contract, not released SDK APIs or mandatory method signatures. They do not extend Completion's scope or require every SDK/provider combination to ship together. Python, C#, TypeScript, Java, Rust, and Go examples demonstrate the same two completion delivery modes; artifact and job examples are shared consumption patterns.

### Completion SDK examples

Each language uses an initialized, purpose-specific Completion client; imports, dependency injection, authentication, and enclosing functions are omitted unless useful to the sketch. C# uses `DaprCompletionClient` and caller token `ct`; TypeScript uses caller `signal`; Go uses caller `ctx`, SDK package alias `completion`, and standard packages `errors`, `fmt`, and `io` inside a function returning `error`. Python and Rust snippets run in an async function; Rust's enclosing function returns `Result`. Java illustrates a closeable blocking iterator, Go a closeable receive stream, and Rust a fallible async stream. These are illustrative SDK wrappers, not claims about existing generated-client interfaces. They validate terminal completion and transport success and propagate errors; disposal/drop cancels unfinished work. Convenience types omit the wire-level `Inference` prefix and `Alpha1` suffix.

The content-event and transcript-reference examples are alternative invocations of the same streaming method, not two deliveries of one generation. The artifact examples reuse the request with a different delivery preference. The `chat` component is a `completion.*` instance. Artifact byte-copy/stream APIs are shown once under [artifact SDK patterns](#artifact-sdk-patterns).

#### Completion: streamed content (`CompleteAlpha1`)

For these Completion examples, `chat` is an OpenAI-backed component with `defaultModel: gpt-4o` and overrides enabled. The request selects `gpt-4o-mini`; omit `model` to use the component default. Absent `delivery` selects `EVENTS`. All Completion delivery modes expose the same `model` and optional provider-supported `idempotency_key` fields (C#: `IdempotencyKey`; TypeScript: `idempotencyKey`); set the key only on integrations that support it.

`ALLOW_REPLACE` accepts native diffusion snapshots without forcing dual-mode providers away from append delivery; use `PREFER_REPLACE` to prefer snapshots. These examples inspect and discard typed events without retaining the whole answer. A UI must key previews by turn/block and honor replacement, including empty replacements; previews are discarded on failure. Text block completion contains no repeated final text. Generated-media references appear in completed media blocks without a different request or method. Successful enumeration requires the terminal event and successful transport termination. An explicit SDK fold helper can consume the same content sequence when a materialized response is desired.

<details>
<summary>Python</summary>

```python
request = CompleteRequest(
    name="chat",
    model="gpt-4o-mini",
    stream_options=StreamOptions(content_mode=ContentMode.ALLOW_REPLACE),
    history=[Turn(role=TurnRole.USER, content=[
        ContentBlock(id="prompt", text="Explain Dapr in one sentence.")
    ])],
)
async for event in client.complete(request):
    print(event)
```

</details>

<details>
<summary>C#</summary>

```csharp
var request = new CompleteRequest
{
    Name = "chat",
    Model = "gpt-4o-mini",
    StreamOptions = new() { ContentMode = ContentMode.AllowReplace },
    History = [new() { Role = TurnRole.User, Content = [
        new() { Id = "prompt", Text = "Explain Dapr in one sentence." }
    ] }]
};
await foreach (var ev in client.CompleteAsync(request, ct))
{
    Console.WriteLine(ev);
}
```

</details>

<details>
<summary>TypeScript</summary>

```typescript
const request: CompleteRequest = {
  name: "chat",
  model: "gpt-4o-mini",
  streamOptions: { contentMode: ContentMode.AllowReplace },
  history: [{ role: TurnRole.User, content: [
    { id: "prompt", text: "Explain Dapr in one sentence." },
  ] }],
};
for await (const event of client.complete(request, { signal })) {
  console.log(event);
}
```

</details>

<details>
<summary>Java</summary>

```java
var request = CompleteRequest.newBuilder()
    .setName("chat")
    .setModel("gpt-4o-mini")
    .setStreamOptions(StreamOptions.newBuilder()
        .setContentMode(ContentMode.ALLOW_REPLACE))
    .addHistory(Turn.newBuilder()
        .setRole(TurnRole.USER)
        .addContent(ContentBlock.newBuilder()
            .setId("prompt")
            .setText("Explain Dapr in one sentence.")))
    .build();

try (var events = client.complete(request)) {
    while (events.hasNext()) {
        System.out.println(events.next());
    }
}
```

</details>

<details>
<summary>Rust</summary>

```rust
use futures_util::TryStreamExt;

let request = CompleteRequest {
    name: "chat".into(),
    model: Some("gpt-4o-mini".into()),
    stream_options: StreamOptions {
        content_mode: ContentMode::AllowReplace,
        ..Default::default()
    },
    history: vec![Turn {
        role: TurnRole::User,
        content: vec![ContentBlock::text(
            "prompt", "Explain Dapr in one sentence."
        )],
        ..Default::default()
    }],
    ..Default::default()
};
let mut events = client.complete(request.clone()).await?;
while let Some(event) = events.try_next().await? {
    println!("{event:?}");
}
```

</details>

<details>
<summary>Go</summary>

```go
request := &completion.CompleteRequest{
    Name:  "chat",
    Model: "gpt-4o-mini",
    StreamOptions: completion.StreamOptions{
        ContentMode: completion.ContentModeAllowReplace,
    },
    History: []completion.Turn{{
        Role: completion.TurnRoleUser,
        Content: []completion.ContentBlock{{
            ID: "prompt", Text: "Explain Dapr in one sentence.",
        }},
    }},
}
events, err := client.Complete(ctx, request)
if err != nil {
    return err
}
defer events.Close()
for {
    event, err := events.Recv()
    if errors.Is(err, io.EOF) {
        break
    }
    if err != nil {
        return err
    }
    fmt.Println(event)
}
```

</details>

#### Completion: streamed transcript reference (`CompleteAlpha1`)

The `chat` component has an explicitly configured artifact backing store. Set `delivery = ARTIFACT` on the same request and consume the same stream, retaining the terminal reference only after successful enumeration. These examples then read the completed transcript lazily through SDK helpers over `ReadCompletionArtifactAlpha1`; reading does not invoke the model again. The helpers consume the returned `CompleteArtifact` descriptor/format, validate JSONL records and underlying transfer completion, and return the content events. No download or full-response fold is implicit in `Complete`.

<details>
<summary>Python</summary>

```python
from contextlib import aclosing

request.delivery = CompleteDeliveryMode.ARTIFACT
stored = None
async for event in client.complete(request):
    if event.completed is not None and event.completed.artifact is not None:
        stored = event.completed.artifact
if stored is None:
    raise RuntimeError("Completion did not return a transcript artifact")
async with aclosing(client.read_completion_events("chat", stored)) as events:
    async for event in events:
        print(event)
```

</details>

<details>
<summary>C#</summary>

```csharp
request.Delivery = CompleteDeliveryMode.Artifact;
CompleteArtifact? stored = null;
await foreach (var ev in client.CompleteAsync(request, ct))
{
    if (ev.Completed?.Artifact is { } artifact)
        stored = artifact;
}
if (stored is null)
    throw new InvalidOperationException("Completion did not return a transcript artifact");
await foreach (var ev in client.ReadCompletionEventsAsync("chat", stored, ct))
{
    Console.WriteLine(ev);
}
```

</details>

<details>
<summary>TypeScript</summary>

```typescript
let stored: CompleteArtifact | undefined;
for await (const event of client.complete({
  ...request,
  delivery: CompleteDeliveryMode.Artifact,
}, { signal })) {
  if (event.completed?.artifact) stored = event.completed.artifact;
}
if (!stored) throw new Error("Completion did not return a transcript artifact");
for await (const event of client.readCompletionEvents("chat", stored, { signal })) {
  console.log(event);
}
```

</details>

<details>
<summary>Java</summary>

```java
var storedRequest = request.toBuilder()
    .setDelivery(CompleteDeliveryMode.ARTIFACT)
    .build();
CompleteArtifact stored = null;
try (var events = client.complete(storedRequest)) {
    while (events.hasNext()) {
        var event = events.next();
        if (event.hasCompleted() && event.getCompleted().hasArtifact()) {
            stored = event.getCompleted().getArtifact();
        }
    }
}
if (stored == null) {
    throw new IllegalStateException("Completion did not return a transcript artifact");
}
try (var events = client.readCompletionEvents("chat", stored)) {
    while (events.hasNext()) {
        System.out.println(events.next());
    }
}
```

</details>

<details>
<summary>Rust</summary>

```rust
let mut stored_request = request.clone();
stored_request.delivery = CompleteDeliveryMode::Artifact;
let mut events = client.complete(stored_request).await?;
let mut stored = None;
while let Some(event) = events.try_next().await? {
    if let CompletionEvent::Completed(completed) = event {
        stored = completed.artifact;
    }
}
let stored = stored.ok_or_else(|| std::io::Error::new(
    std::io::ErrorKind::InvalidData,
    "Completion did not return a transcript artifact",
))?;
let mut transcript = client.read_completion_events("chat", stored).await?;
while let Some(event) = transcript.try_next().await? {
    println!("{event:?}");
}
```

</details>

<details>
<summary>Go</summary>

```go
storedRequest := *request
storedRequest.Delivery = completion.CompleteDeliveryModeArtifact
artifactEvents, err := client.Complete(ctx, &storedRequest)
if err != nil {
    return err
}
defer artifactEvents.Close()
var stored *completion.CompleteArtifact
for {
    event, err := artifactEvents.Recv()
    if errors.Is(err, io.EOF) {
        break
    }
    if err != nil {
        return err
    }
    if event.Completed != nil && event.Completed.Artifact != nil {
        stored = event.Completed.Artifact
    }
}
if stored == nil {
    return errors.New("completion did not return a transcript artifact")
}
transcript, err := client.ReadCompletionEvents(ctx, "chat", stored)
if err != nil {
    return err
}
defer transcript.Close()
for {
    event, err := transcript.Recv()
    if errors.Is(err, io.EOF) {
        break
    }
    if err != nil {
        return err
    }
    fmt.Println(event)
}
```

</details>

### Discovery SDK pattern

Each purpose-specific client exposes model listing and metadata lookup through its own RPCs. Applications need only the Dapr SDK; provider SDKs and credentials remain in the component. Helpers may offer lazy page iteration, but must not fetch or materialize the full catalog implicitly. Listing permission does not grant metadata or inference permission.

<details>
<summary>Language-neutral discovery and illustrative pricing</summary>

With the required discovery grants, an application can walk model pages and explicitly query a selected model:

```text
token = ""
repeat:
    page = client.ListModels(name=component, page_size=25, page_token=token)
    consume(page.models)
    token = page.next_page_token
until token == ""

metadata = client.GetModelMetadata(name=component, model=selected_model)
consume(metadata.prices, metadata.effective_routing_options)
consume(metadata.deployments)
while metadata.next_page_token != "":
    metadata = client.GetModelMetadata(
        name=component, model=selected_model, page_token=metadata.next_page_token)
    consume(metadata.deployments)
```

All calls carry the caller's cancellation/deadline through the SDK. A filtered empty catalog page is not necessarily the final page. Preserve source timestamps and optional/unknown fields; surface lookup errors rather than inventing metadata. Deployments may be paginated independently of the model catalog.

An illustrative normalized price record, not a quote for a real provider:

```json
{
  "amount": "0.15",
  "currency": "USD",
  "unit": "input_token",
  "unitQuantity": "1000000"
}
```

Compare rates only with compatible currencies, billing units, and conditions. Provider-advertised deployment quantization and configured routing settings are not proof of the deployment selected for a later invocation.

</details>

### Batch SDK pattern

Batch helpers are illustrative additions to `DaprCompletionClient`, separate from its request-bound `Complete` iterator. A helper may accept an iterable of `(item_id, CompleteRequest)` values and emit the initial/items/finish upload frames without requiring the application to build a whole request array. Polling is explicit or application/Workflow-managed; result iteration yields correlated item events and errors, not an implicitly materialized batch.

<details>
<summary>Language-neutral batch lifecycle and HTTP upload sketch</summary>

The component's policy must explicitly grant the batch operations; the interactive-only policy example does not do so.

```text
SubmitCompletionBatch(items) -> batch handle
GetCompletionBatch(handle)  -> poll with a delay until terminal
ReadCompletionBatchResults(handle)
  -> per-item completion events or item errors
  -> final counts and successful transport termination
```

An HTTP submission body uses one envelope per line, not a JSON array:

```jsonl
{"initial":{"name":"chat","model":"gpt-4o-mini"}}
{"item":{"itemId":"document-1","request":{"history":[{"role":"USER","content":[{"id":"prompt","text":"Summarize document one."}]}]}}}
{"item":{"itemId":"document-2","request":{"history":[{"role":"USER","content":[{"id":"prompt","text":"Summarize document two."}]}]}}}
{"finish":{"itemCount":"2"}}
```

Keep the returned batch ID for later retrieval or `CancelCompletionBatch`. A terminal batch may contain item errors or only partial successful work. Inspect per-item dispositions and final counts; do not assume output order matches input order or automatically retry failed items. Canceling an active result iterator stops retrieval, not the provider batch.

</details>

### Artifact SDK patterns

These patterns apply to a Completion, Image Generation, or Video Generation client bound to its own registry. Set `component_name` / `componentName` to that component and `artifact_id` / `artifactId` to a descriptor returned by it (for Completion, `stored.artifact`). They do not require a universal Inference SDK client.

Here `artifactId` / `artifact_id` is selected from a successfully published descriptor. These examples stream into a caller-selected temporary destination and do not accumulate the file in memory. Publish/rename the destination only after success; remove incomplete temporary files according to application policy. Python uses synchronous per-chunk file writes for brevity; an application can substitute an async sink. The TypeScript example targets Node.js; browser callers can pipe an SDK `ReadableStream` into a writable file stream instead.

<details>
<summary>.NET artifact-reader API sketch</summary>

The .NET surface follows the existing [Binary Store client](https://github.com/dapr/dotnet-sdk/blob/master/src/Dapr.BinaryStore.Abstractions/DaprBinaryStoreClient.cs) pattern, with an additional read-stream convenience:

```csharp
IAsyncEnumerable<ReadOnlyMemory<byte>> ReadArtifactChunksAsync(
    string componentName, string artifactId,
    CancellationToken cancellationToken = default);

Task ReadArtifactAsync(
    string componentName, string artifactId, Stream destination,
    CancellationToken cancellationToken = default);

Task<Stream> OpenArtifactReadAsync(
    string componentName, string artifactId,
    CancellationToken cancellationToken = default);
```

These are proposed members on each adopting purpose-specific client, including `DaprCompletionClient`. The iterator checks framing/sequence/integrity and throws on failed termination; returned chunk memory is not reused or mutated after yielding it. Retaining every chunk is the caller's choice, not implicit SDK buffering. The destination overload copies incrementally and leaves the caller's stream open. The returned stream is read-only and non-seekable; for a nonempty read buffer, `ReadAsync` returns zero only after validated completion, and failures propagate from enumeration/copy/read rather than becoming clean EOF. `Length` and seeking are not required; callers obtain optional size from the artifact descriptor. Disposing early cancels without claiming a complete download.

```csharp
// For a job-based adopting client, operation is already SUCCEEDED.
foreach (var artifact in operation.Artifacts)
{
    await using var content = await client.OpenArtifactReadAsync(
        componentName, artifact.ArtifactId, cancellationToken);
    await using var destination = File.Create(GetDestinationPath(artifact));
    await content.CopyToAsync(destination, cancellationToken);
}
```

The sketch does not promise atomic destination-file replacement on failure. Unlike Completion's structured-response fold helper, artifact helpers must not accumulate a whole result in memory. Go exposes a closable reader or writer-copy API; Python, JavaScript, and Java expose equivalent lazy byte iterators/readable streams, with the same cancellation and late-error behavior.

</details>

<details>
<summary>Python</summary>

```python
from contextlib import aclosing

with open("output.partial", "wb") as destination:
    async with aclosing(client.read_artifact_chunks(component_name, artifact_id)) as chunks:
        async for chunk in chunks:
            destination.write(chunk)
```

</details>

<details>
<summary>C#</summary>

```csharp
await using var content = await client.OpenArtifactReadAsync(componentName, artifactId, ct);
await using var destination = File.Create("output.partial");
await content.CopyToAsync(destination, ct);
```

</details>

<details>
<summary>TypeScript</summary>

```typescript
import { createWriteStream } from "node:fs";
import { pipeline } from "node:stream/promises";

await pipeline(
  client.readArtifactChunks(componentName, artifactId, { signal }),
  createWriteStream("output.partial"),
  { signal },
);
```

</details>

### Operation SDK patterns

These media-operation patterns belong to an adopting block's client, **not `DaprCompletionClient`**; Completion batches use the separate [batch pattern](#batch-sdk-pattern). Set `component_name` / `componentName` to that purpose's component and `submitted_operation` / `submittedOperation` to its generation result. Python uses `asyncio`; C# uses caller token `ct`; TypeScript uses caller signal `signal`. Polling cancellation stops the lookup, not the independent job.

#### Polling

Short-lived callers can poll with a delay and an overall application timeout; long-lived jobs should persist the reference and use durable Workflow timers instead. Immediate success skips polling. A successful polling RPC can still report a failed/canceled operation, so explicitly check the terminal status before accessing artifacts. SDKs validate that pending/running operations contain a nonempty operation ID.

<details>
<summary>Python</summary>

```python
async with asyncio.timeout(300):
    operation = submitted_operation
    while operation.status in (OperationStatus.PENDING, OperationStatus.RUNNING):
        await asyncio.sleep(2)
        operation = await client.get_operation(component_name, operation.operation_id)
    if operation.status != OperationStatus.SUCCEEDED:
        raise RuntimeError(f"{operation.status}: {operation.error_message}")
    print(operation.artifacts)
```

</details>

<details>
<summary>C#</summary>

```csharp
using var polling = CancellationTokenSource.CreateLinkedTokenSource(ct);
polling.CancelAfter(TimeSpan.FromMinutes(5));
var operation = submittedOperation;
while (operation.Status is OperationStatus.Pending or OperationStatus.Running)
{
    await Task.Delay(TimeSpan.FromSeconds(2), polling.Token);
    operation = await client.GetOperationAsync(componentName, operation.OperationId, polling.Token);
}
if (operation.Status != OperationStatus.Succeeded)
    throw new InvalidOperationException($"{operation.Status}: {operation.ErrorMessage}");
Console.WriteLine(operation.Artifacts.Count);
```

</details>

<details>
<summary>TypeScript</summary>

```typescript
import { setTimeout as delay } from "node:timers/promises";

const pollingSignal = AbortSignal.any([signal, AbortSignal.timeout(300_000)]);
let operation = submittedOperation;
while ([OperationStatus.Pending, OperationStatus.Running].includes(operation.status)) {
  await delay(2_000, undefined, { signal: pollingSignal });
  operation = await client.getOperation(componentName, operation.operationId, { signal: pollingSignal });
}
if (operation.status !== OperationStatus.Succeeded) {
  throw new Error(`${operation.status}: ${operation.errorMessage}`);
}
console.log(operation.artifacts);
```

</details>

#### Explicit job cancellation

This is an **alternative** to waiting for the pending/running submitted operation, not a step after the polling example. Once submission has handed back the operation, canceling a polling request/task cancels that upstream lookup, not the independent generation job. Use this endpoint to cancel the job itself. Cancellation of the original generation request before handoff instead triggers the [end-to-end provider-cancellation path](#end-to-end-cancellation) automatically. A successful cancellation request is not proof that generation stopped before a completion race; poll again if the final status matters.

<details>
<summary>Python</summary>

```python
if submitted_operation.status in (OperationStatus.PENDING, OperationStatus.RUNNING):
    await client.cancel_operation(component_name, submitted_operation.operation_id)
```

</details>

<details>
<summary>C#</summary>

```csharp
if (submittedOperation.Status is OperationStatus.Pending or OperationStatus.Running)
    await client.CancelOperationAsync(componentName, submittedOperation.OperationId, ct);
```

</details>

<details>
<summary>TypeScript</summary>

```typescript
if ([OperationStatus.Pending, OperationStatus.Running].includes(submittedOperation.status)) {
  await client.cancelOperation(componentName, submittedOperation.operationId, { signal });
}
```

</details>

## Sibling operation reference

This media-operation reference is retained here for reuse by the independently approved image/video proposals. Completion's [native batch lifecycle](#batch-lifecycle-and-cancellation) uses its own batch/status/item-result contract rather than this single-generation envelope. Illustrative media polling and cancellation examples are in the [SDK appendix](#operation-sdk-patterns).

### Asynchronous operation conventions

These reusable types and semantics are available to independently approved media-generation blocks. Interactive Completion and its artifact generation remain request-bound; native Completion batches use the separate batch APIs above. Adopting proposals define their own purpose-specific RPC names and routes; there is no generic `GetInferenceOperation` or `CancelInferenceOperation` RPC.

```protobuf
message InferenceGetOperationRequestAlpha1 {
  string name = 1;
  string operation_id = 2;
  InferenceCacheOptionsAlpha1 cache = 3;
}

message InferenceCancelOperationRequestAlpha1 {
  string name = 1;
  string operation_id = 2;
}

enum InferenceOperationStatusAlpha1 {
  INFERENCE_OPERATION_STATUS_ALPHA1_UNSPECIFIED = 0;
  INFERENCE_OPERATION_STATUS_ALPHA1_PENDING = 1;
  INFERENCE_OPERATION_STATUS_ALPHA1_RUNNING = 2;
  INFERENCE_OPERATION_STATUS_ALPHA1_SUCCEEDED = 3;
  INFERENCE_OPERATION_STATUS_ALPHA1_FAILED = 4;
  INFERENCE_OPERATION_STATUS_ALPHA1_CANCELED = 5;
}

message InferenceOperationAlpha1 {
  // Provider or component-encoded operation token; empty if completed synchronously without a job ID.
  string operation_id = 1;
  InferenceOperationStatusAlpha1 status = 2;
  optional string model = 3;
  repeated InferenceArtifactAlpha1 artifacts = 4;
  optional string error_message = 5;
  optional InferenceUsageAlpha1 usage = 6;
  InferenceCacheResultAlpha1 cache = 7;
}
```

Synchronous generation returns `SUCCEEDED` and descriptors (`HTTP 200`); its operation ID may be empty when there is no job handle. Asynchronous submission returns `PENDING` or `RUNNING`, a nonempty bound operation ID, and `HTTP 202` with a purpose-specific operation `Location`. Successful polls return the same immutable artifacts. Failed/canceled operations do not expose partial successful artifacts.

The operation is bound to the creating building block, component instance, provider, and effective model. Polling, cancellation, and reads never re-resolve a new default model or generate a new sample. Native provider handles or shared persistent storage must preserve that binding across ordinary restarts/replicas; do not rely on a process-local job dictionary.

Adopters apply [end-to-end cancellation](#end-to-end-cancellation), including job-creation races, intentional asynchronous handoff, provider-confirmed terminal cancellation, and explicit unsupported cancellation. Only small successful final lookup snapshots are cacheable metadata; active, failed, and canceled snapshots bypass caching. A cached snapshot records completion, not continued artifact availability, and cannot satisfy a new generation request. Operation lookups and cancellation are separate calls with their own deadlines.

## Related items

- [20240618-RCBS-Conversation-building-block.md](./20240618-RCBS-Conversation-building-block.md) — original Conversation building block (`Alpha1`)
- [dapr/proposals#113](https://github.com/dapr/proposals/pull/113) — Conversation streaming + text-diffusion design. This proposal carries forward its delivery guarantees, buffering, and resiliency rules, but replaces its Alpha2-shaped event payloads with the typed Completion contract below. The proposals have different compatibility goals; this does not amend the open Conversation proposal.
- [Cryptography building block](./20230327-RCBS-Crypto-building-block.md) and [dapr/proposals#88](https://github.com/dapr/proposals/pull/88) (Binary Store) — prior art for chunked binary transfer. Artifact reads reuse [`common.v1.StreamPayload`](https://github.com/dapr/dapr/blob/master/dapr/proto/common/v1/common.proto), also used by Crypto, and follow the .NET Binary Store's chunk-enumeration and destination-stream API pattern.
- [dapr/proposals#71](https://github.com/dapr/proposals/pull/71) — proposed Centralized Cache building block, one supported backend for optional caching of small final responses. A general Dapr state store can serve the same purpose; inference is not dependent on this proposal shipping. Large/streamed results bypass caching; explicitly requested artifact storage is separate.
- [dapr/proposals#108](https://github.com/dapr/proposals/pull/108) — request-time model selection and model-aware cache isolation. This proposal adopts request model selection but deliberately retains optional component defaults and an operator-controlled override lock, unlike that proposal's required request-only model.
- The [model-allowlist feedback on dapr/proposals#108](https://github.com/dapr/proposals/pull/108#issuecomment-4865906836) and [policy implementation reference on dapr/proposals#108](https://github.com/dapr/proposals/pull/108#issuecomment-4902668245) motivate initial-release model governance. The [Workflow access-policy documentation](https://docs.dapr.io/operations/security/workflow-access-policy/) provides the resource-scoping and allow-list pattern; the inference-specific differences are defined under [model access policies](#model-access-policies).
