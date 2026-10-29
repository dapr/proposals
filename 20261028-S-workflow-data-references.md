# Standardizing the Data Reference Envelope Across Dapr SDKs

- Author: Whit Waldo @whitwaldo
- State: Proposed, Partially Implemented
- Introduced: 2026-10-28
- Target SDKs: Rust, Python, Java, Go (reference imlementations: .NET `Dapr.Workflow` 1.19.0, JavaScript `@dapr/dapr` 3.19.0)
- Runtime dependency: Dapr v1.19 or later (requires Binary Store API, alpha); no runtime changes proposed

## Overview 
This proposal standardizes, across the Rust, Python, Java, and Go SDKs, the *workflow data reference* scheme that shipped in the .NET SDK ([dotnet-sdk #1934](https://github.com/dapr/dotnet-sdk/pull/1934)) and the JavaScript SDK ([js-sdk #905](https://github.com/dapr/js-sdk/pull/905)) for Dapr 1.19. A data reference lets an SDK keep a large workflow payload out of inline workflow history: the SDK serializes the value, writes the bytes to a Dapr Binary Store component, and records a small, versioned JSON envelope in its place. The runtime is not involved in this decision at all; it stores and replays whatever bytes the SDK hands it.

Because the runtime is deliberately unaware of the envelope, interoperability is entirely an SDK-to-SDK contract. Today that contract exists only implicitly, as two independent implementations that agree on the contract. This proposal makes it explicit so that a payload written by one SDK can be read by any other, and so that each remaining SDK implements the same policy surface rather than inventing its own.

It proposes three things:

1. A normative definition of the version-1 `$daprRef` envelope: field names, types, key format, hashing, and the parse/emit rules every SDK must follow.
2. A common resolution model (write preferences, offload sites, inbound handling, cross-app inlining, limits, and error categories) with language-idiomatic API guidance for Rust, Python, Java, and Go.
3. A prerequisite Binary Store client surface each SDK must expose before it can implement data references. Python and Java carry alpha binary-store client methods on their main branches today; Go and Rust have none, and no SDK other than .NET and JavaScript has the workflow integration.

Affected areas: SDK workflow packages (Rust, Python, Java, Go), SDK binary-store clients (Go, Rust; Python and Java to confirm streaming coverage), SDK documentation, and a shared conformance test suite. No changes to `dapr/dapr`, `dapr/durabletask-go`, or the workflow gRPC protocol.

## Background

Workflow history carries every input, output, and event inline. A document forwarded through several activities is copied into history at each boundary, replayed on every orchestrator turn, and fetched again by every client that reads the instance state. The .NET benchmark on the [data-references docs page](https://docs.dapr.io/developing-applications/sdks/dotnet/dotnet-workflow/dotnet-workflow-data-references/#benchmark-comparison) (a 2,000,000-byte document across nine activities) measured 12,010,505 bytes of workflow protobuf traffic inline versus about 12.3 KB with references, and explicit references ran 2.47x faster with 80.1% fewer managed allocations. Dapr 1.19 added the Binary Store building block ([overview](https://docs.dapr.io/developing-applications/building-blocks/binarystore/binarystore-overview/)), which gives SDKs a portable place to put those bytes.

### What .NET and JavaScript shipped

Both SDKs implement the same scheme entirely client-side:

- **Offloading happens after serialization.** The SDK serializes the value with its configured serializer, measures the UTF-8 byte length, and, if policy says so, uploads the bytes to the binary store and substitutes a reference envelope in the outgoing wire string.
- **The envelope is a reserved JSON object.** A single-property object `{"$daprRef": {...}}` carrying the format version, store name, key, size, SHA-256, and optional stage metadata. Anything else is ordinary application data.
- **Inbound handling preserves the recorded representation.** A reference arriving where an ordinary `T` is expected is resolved and deserialized; a reference arriving where a `PayloadReference<T>` is expected is passed through untouched; inline data arriving where a reference is expected is a contract error, never silently uploaded.
- **Policy is host-wide and future-write only.** `WritePreference` (`Preserve`, `PreferReference`, `PreferInline`), `OffloadThreshold`, `OffloadSites`, `MaxInMemoryRehydrationSize`, `VerifyIntegrityOnRead`, and `SkipWriteIfExists` govern new writes and buffered reads; nothing rewrites retained history.
- **Cross-app inputs inline by default.** A top-level reference sent to another `TargetAppId` is resolved by the sender and sent inline unless the call explicitly sets `AllowPayloadReferences: true`.
- **Cleanup is explicit.** Neither completion nor purge deletes stored objects; a durable manifest plus an explicit purge call is the only deletion path.

The JavaScript implementation was written against the .NET envelope and API by design, and the two docs pages cross-reference each other as the only interoperable pair. The .NET docs state plainly: do not assume other SDKs support this scheme. That is the gap this proposal closes.

### Where the other SDKs stand

| SDK | Binary Store client | Workflow data references | Tracking |
| --- | --- | --- | --- |
| .NET | `Dapr.BinaryStore` 1.19.0 (stream + buffered) | Shipped, `Dapr.Workflow` 1.19.0 | [#1934](https://github.com/dapr/dotnet-sdk/pull/1934) |
| JavaScript | `DaprClient.binaryStore` (HTTP and gRPC, chunked) | Shipped, `@dapr/dapr` 3.19.0 | [#905](https://github.com/dapr/js-sdk/pull/905) |
| Python | `set/get/delete_binary_file_alpha1` on gRPC client (sync and aio) | None | [#1296](https://github.com/dapr/python-sdk/issues/1296) |
| Java | `setBinaryFile`, `getBinaryFile` (Flux), `getBinaryFileBytes`, `deleteBinaryFile` | None | [#1851](https://github.com/dapr/java-sdk/issues/1851) |
| Go | None | None | [#848](https://github.com/dapr/go-sdk/issues/848) (open) |
| Rust | None | None | No issue filed |

Status taken from each repository's default branch on 2026-10-27. The Python and Java binary-store surfaces exist but have not been validated against the streaming and error-mapping requirements in the "Required SDK support" section below.

## Related Items

- [Multi-App Workflows](https://github.com/dapr/proposals/blob/main/20250320-RS-multi-app-workflows.md): defines `TargetAppId` routing and history propagation, which set the cross-app boundary this proposal inlines across.
- [Workflow Patching and Versioning](https://github.com/dapr/proposals/blob/main/20251028-BRS-workflow-versioning.md): the versioning mechanism required when an existing `T` contract becomes a `PayloadReference<T>` contract.
- [SDK Resiliency](https://github.com/dapr/proposals/blob/main/20230714-S-sdk-resiliency.md): binary-store reads and writes made by the resolver should participate in SDK-level retry policy where one exists.

### Related issues

- [dotnet-sdk #1533](https://github.com/dapr/dotnet-sdk/issues/1533): first-class input/output data references (closed by #1934)
- [js-sdk #884](https://github.com/dapr/js-sdk/issues/884): data references in binary store (closed by #905)
- [python-sdk #1296](https://github.com/dapr/python-sdk/issues/1296), [java-sdk #1851](https://github.com/dapr/java-sdk/issues/1851), [go-sdk #848](https://github.com/dapr/go-sdk/issues/848): Binary Store API (alpha) support

## Expectations and Alternatives

### In scope

- A normative specification of the version-1 envelope that Rust, Python, Java, and Go must emit and accept byte-for-byte compatibly with .NET and JavaScript.
- The shared policy model (write preference, offload sites, threshold, rehydration limit, integrity verification, cross-app inlining, inbound contract rules, error taxonomy) with per-language API sketches.
- The Binary Store client capabilities each SDK needs before the workflow integration can land.
- A cross-SDK conformance test fixture (envelope test vectors plus a multi-app interop scenario).

### Out of scope

- Runtime awareness of references. The runtime continues to treat payloads as opaque bytes; a future runtime-native scheme is discussed under Alternatives.
- Stage transforms (compression, encryption). The `stages` field is reserved and defined, but no SDK is required to implement a stage in this proposal; all must reject envelopes with non-empty stages they cannot reverse.
- Automatic garbage collection of stored objects. Cleanup stays explicit and manifest-driven, as in .NET and JavaScript.
- Nested references inside application objects. Only top-level payloads at workflow boundaries receive envelope handling.
- Changes to the .NET or JavaScript public APIs. Where this proposal tightens a rule, both existing SDKs already comply.

### Trade-offs

- **SDK-side logic over runtime support.** Keeping the runtime unaware means the feature ships without a runtime release and works against any 1.19+ sidecar, but it also means every SDK must implement the same policy code and a mixed fleet of payload-aware and payload-unaware workers for one app ID is unsafe.
- **JSON sentinel over a protobuf field.** The `$daprRef` sentinel rides inside the existing string payload, so no protocol change is needed, but it reserves a property name in application JSON space and requires each SDK to sniff every inbound payload with a cheap prefix check before parsing.
- **Content-addressed keys over random keys.** Content addressing makes retries idempotent and lets `SkipWriteIfExists` treat an `AlreadyExists` response as success, at the cost of requiring a full SHA-256 pass over the serialized bytes on every write.
- **Strict inline-or-error over best-effort fallback.** A reference that cannot be resolved for a cross-app call fails the call rather than forwarding the reference, which is predictable but means misconfiguration surfaces as hard failures.

## Implementation details

### Design: the version-1 envelope

The envelope is the only part of this scheme that crosses a language boundary, so it is specified normatively. It is taken verbatim from the shipped .NET `PayloadReference` and JavaScript `PayloadReference.toJSON()`; nothing below changes what those SDKs emit.

An envelope is a JSON object with exactly one property, `$daprRef`, whose value is an object with these members:

| Property | Type | Required | Meaning |
| --- | --- | --- | --- |
| `v` | integer | yes | Envelope format version. Must be `1`. |
| `store` | string | yes | Name of the binary-store component the bytes were written to. Non-empty after trimming. |
| `key` | string | yes | File name within that store. Non-empty after trimming. |
| `size` | integer | yes | Byte length of the stored object (after any stages). Non-negative; must fit in 53 bits so JavaScript can represent it exactly. |
| `sha256` | string | yes | Lowercase hex SHA-256 (64 chars, `^[0-9a-f]{64}$`) of the serialized UTF-8 bytes *before* any stages. |
| `stages` | array | no | Ordered transforms applied before storage. Omitted (not `[]`) when empty. Each item: `{"n": string, "m"?: {string: string}}`. |

A minimal unstaged envelope, which is the only form any current SDK writes:

```json
{"$daprRef":{"v":1,"store":"workflow-payloads","key":"workflow-payloads/order-42/9f86d081884c7d659a2feaa0c55ad015a3bf4f1b2b0b822cd15d6c15b0f00a08","size":131102,"sha256":"9f86d081884c7d659a2feaa0c55ad015a3bf4f1b2b0b822cd15d6c15b0f00a08"}}
```

#### Key format

`<keyPrefix>/<instanceId>/<sha256>`, where `keyPrefix` defaults to `workflow-payloads`, `instanceId` is the workflow instance the payload was written for, and `sha256` is the same hash recorded in the envelope. Keys are therefore content-addressed within an instance: writing the same bytes twice for the same instance yields the same key, which is what makes `SkipWriteIfExists` safe. Readers must use the `store` and `key` recorded in the envelope, never a locally configured store name.

#### Hash and size

The hash covers the serialized payload exactly as the writer produced it: the UTF-8 encoding of the serializer's output string. `size` is the byte count of what was actually stored. With no stages these are the same bytes, so `size` equals the hashed length; with stages they may differ. Readers that buffer the object must verify `sha256` when integrity verification is enabled (the default) and must enforce the rehydration limit against both the declared `size` and the bytes actually received.

#### Stages (reserved)

`stages` records transforms such as compression or encryption applied after hashing and before upload, in application order; a reader reverses them in reverse order. Neither the .NET default converter nor the JavaScript resolver implements any stage today, and both throw on a non-empty `stages` array during buffered resolution. New SDKs must do the same: parse and preserve `stages` so the envelope can be forwarded intact, but refuse to materialize a staged payload unless they implement every named stage. Stage names are not registered here; a later proposal can define the first stage and its metadata keys.

#### Detection and parse rules

Every SDK must apply the same recognizer to every inbound wire payload it would otherwise deserialize:

1. If the payload, after leading whitespace, does not start with `{`, it is ordinary data. Stop.
2. Attempt a JSON parse. If parsing fails and the text does not contain the substring `$daprRef`, it is ordinary data (a non-JSON serializer may legitimately produce a leading brace). If it fails and the substring is present, raise an invalid-envelope error.
3. If the parsed value is not an object with exactly one property named `$daprRef`, it is ordinary data.
4. Otherwise validate the body against the table above. Any missing or mistyped required member, a `v` other than `1`, an unparseable `sha256`, or a malformed stage item raises an invalid-envelope error. It must not fall back to treating the text as application data.

An application object that happens to contain a `$daprRef` property alongside other properties is not an envelope and must deserialize normally; only the single-property form is reserved.

#### Emit rules

- Emit members in the order `v`, `store`, `key`, `size`, `sha256`, `stages`, with no whitespace. Byte-identical output across SDKs is not a correctness requirement, but it makes test vectors and history diffs trivially comparable.
- Omit `stages` entirely when empty.
- Never emit an envelope nested inside another value. Only a top-level workflow payload (input, output, event, custom status, continue-as-new input) can be an envelope.
- The envelope string is what goes on the wire in place of the serialized value; it is not re-wrapped by the application serializer. SDKs with pluggable serializers must bypass the serializer for the envelope itself, as .NET and JavaScript do.

### Design: resolution model without runtime involvement

Each SDK introduces a *payload resolver* that sits between the application serializer and the workflow transport. It is consulted at every payload *site* (a boundary where a value enters or leaves workflow history) and nowhere else. The runtime never sees a resolver; it receives and replays strings.

#### Payload sites

The site kinds are a bit set shared by all SDKs, used both to scope automatic offloading (`OffloadSites`) and to identify the boundary in logs and errors:

`WorkflowInput`, `WorkflowOutput`, `ActivityInput`, `ActivityOutput`, `ChildWorkflowInput`, `ChildWorkflowOutput`, `ExternalEvent`, `ContinueAsNewInput`, `CustomStatus`. Detached workflow inputs and rerun replacement inputs use `WorkflowInput`; termination output uses `WorkflowOutput`. `All` and `None` are the two named combinations.

#### Outbound: the write decision

For an outgoing value at a site, after serialization to a wire string:

1. Recognize whether the wire string is already an envelope.
2. If the site requires inline (cross-app input without `AllowPayloadReferences`) or the write preference is `PreferInline`: an envelope is resolved to its original bytes and those bytes are sent inline (inline-or-error); ordinary data passes through.
3. Otherwise, an existing envelope passes through unchanged, and ordinary data is offloaded only if *all* of: preference is `PreferReference`, the site is in `OffloadSites`, the UTF-8 length is at or above `OffloadThreshold` (or the threshold is unset), and the string is non-empty.
4. Offloading computes SHA-256, builds the key, uploads with overwrite disabled, treats an `AlreadyExists` failure as success when `SkipWriteIfExists` is true, records the reference in the manifest if one is configured, and substitutes the envelope string.

The decision is made per write and never persisted. Changing the preference or threshold affects only subsequent writes.

#### Inbound: the contract table

Inbound handling is independent of the write preference and identical for local, cross-app, and event payloads:

| Received | Declared contract | Behavior |
| --- | --- | --- |
| Inline data | Ordinary `T` | Deserialize normally. |
| Envelope | Ordinary `T` | Resolve (download, verify, deserialize) through the resolver. |
| Envelope | Reference | Pass through unresolved. |
| Inline data | Reference | Contract mismatch error. Never upload to manufacture a reference. |

How a contract is "declared" is language-specific: a generic type in .NET, Java, and Rust; a registration flag in JavaScript and Python; a wrapper type or option in Go. The behavior must not depend on which mechanism is used.

#### Buffered resolution

`resolve(reference)` returns the deserialized value and must, in order: reject if `size` exceeds `MaxInMemoryRehydrationSize`; reject if `stages` is non-empty and unsupported; stream the object while enforcing the limit against bytes received; verify SHA-256 if enabled; decode UTF-8; deserialize. Resolved values may be cached, but only within one orchestrator execution and only for unstaged references, bounded by `ResolutionCacheSize` entries and `ResolutionCacheBytes` summed envelope `size`.

`open(reference)` returns a byte stream of the stored object with no limit, no verification, and no stage reversal. It is the only path for payloads larger than the rehydration limit.

#### Where I/O is allowed

Storage I/O never runs inside the deterministic workflow body. Explicit `store` is a client or activity operation. In-workflow `resolve` and `open` are exposed as workflow tasks (so they are recorded and replayed like any other awaited task) and are discouraged in favor of resolving in activities. Cross-app inlining performed on behalf of a workflow runs after replay has filtered already-recorded calls.

#### Error taxonomy

Every SDK exposes these distinguishable failures, named idiomatically:

| Error | Raised when |
| --- | --- |
| `PayloadNotFound` | The store returns not-found for the recorded key. |
| `PayloadIntegrity` | SHA-256 of received bytes differs from `sha256`. |
| `PayloadRehydrationLimit` | Declared or actual size exceeds `MaxInMemoryRehydrationSize`. |
| `PayloadNotResolvable` | A reference was received but no resolver is registered. |
| `InvalidPayloadEnvelope` | The detection rules above found a malformed or unsupported envelope (including `v != 1` and unsupported stages). |
| `PayloadContractMismatch` | Inline data arrived at a reference contract, or an inline result was returned to a reference-result request. |

None of these may degrade to returning the raw envelope string as if it were application data.

#### Default option values

| Option | Default | Notes |
| --- | --- | --- |
| `StoreName` | required | Component name; must match the component. |
| `KeyPrefix` | `workflow-payloads` | First key segment. |
| `WritePreference` | `Preserve` | Registering a resolver does not by itself offload anything. |
| `OffloadSites` | `All` |  |
| `OffloadThreshold` | 65,536 bytes | Inclusive; unset means every non-empty value. |
| `MaxInMemoryRehydrationSize` | 16,777,216 bytes | Applies to every buffered read of every retained reference; lowering it is a compatibility change. |
| `VerifyIntegrityOnRead` | true | Buffered reads only. |
| `SkipWriteIfExists` | true |  |
| `ResolutionCacheSize` | 128 entries | Per orchestrator execution; 0 disables. |
| `ResolutionCacheBytes` | 67,108,864 bytes | JavaScript exposes this; .NET bounds by entries only. New SDKs expose both. |
| `Manifest` | none | Durable `record`/`list`/`clear` implementation required for `deletePayloads`. |

These defaults are the current .NET and JavaScript defaults and are part of the standard so that a workflow behaves the same way whichever SDK hosts it.

### Required SDK support: Binary Store client

Data references are built on top of the Binary Store API, so an SDK cannot implement this proposal until it exposes a binary-store client with the capabilities below. .NET (`Dapr.BinaryStore`) and JavaScript (`DaprClient.binaryStore`) meet all of them. Python and Java have alpha client methods that need to be checked against this list. Go and Rust have no binary-store client at all and must add one first; this is the larger part of the work for those two SDKs.

The runtime surface is the alpha gRPC service in `dapr/proto/runtime/v1/binarystore.proto`: `SetBinaryFileAlpha1` (client-streaming; first message carries `component_name`, `file_name`, optional `overwrite`; subsequent messages carry `StreamPayload` chunks), `GetBinaryFileAlpha1` (server-streaming `StreamPayload` chunks), and `DeleteBinaryFileAlpha1`. The HTTP API is `v1.0-alpha1`.

#### Capability checklist

| Capability | Why the resolver needs it | .NET | JS | Python | Java | Go | Rust |
| --- | --- | --- | --- | --- | --- | --- | --- |
| Upload from bytes | `store` writes the serialized value | yes | yes | yes | yes | no | no |
| Upload from a stream | Not required for v1 (writers serialize in memory), but keeps large explicit stores bounded | yes | yes | verify | verify | no | no |
| `overwrite=false` (POST semantics) | Content-addressed keys rely on create-if-absent | yes | yes | verify | yes | no | no |
| Distinguish `AlreadyExists` on upload | `SkipWriteIfExists` treats it as success | yes | yes (gRPC `AlreadyExists` / HTTP 409) | verify | verify | no | no |
| Download as a chunk iterator / stream | `open` and the limit-enforcing path of `resolve` must not buffer first | yes | yes | yes (iterable response) | yes (`Flux<byte[]>`) | no | no |
| Download fully buffered | Convenience for small values | yes | yes | yes (`read()`) | yes (`getBinaryFileBytes`) | no | no |
| Distinguish `NotFound` on download and delete | `PayloadNotFound`; delete tolerates missing objects | yes | yes (gRPC `NotFound` / HTTP 404) | verify | verify | no | no |
| Delete | `deletePayloads` via manifest | yes | yes | yes | yes | no | no |
| Cancellation / deadline on every call | Resolver calls accept a cancellation token or context | yes | yes (`AbortSignal`) | yes (timeout) | verify | no | no |
| Same endpoint and API token as the workflow client | References are resolved through the reader's own sidecar | yes | yes | verify | verify | no | no |

"verify" means the method exists on the default branch but the specific behavior has not been confirmed against this list; the SDK's binary-store issue should be reopened or a follow-up filed for any gap.

#### Minimum client shape

In language-neutral terms the resolver depends on this interface and nothing more:

```text
set(store, key, bytes | stream, overwrite=false, cancel) -> void      // AlreadyExists must be identifiable
get_chunks(store, key, cancel) -> stream of bytes                    // NotFound must be identifiable
delete(store, key, cancel) -> void                                   // NotFound must be identifiable
```

Go and Rust should add this to the existing client (`client.Client` in Go; the `Client` struct in Rust) as alpha methods named consistently with the proto (`SetBinaryFileAlpha1` and so on, or the language's idiomatic equivalent), with a streaming download that yields chunks without buffering, before any workflow work begins. The Go issue ([#848](https://github.com/dapr/go-sdk/issues/848)) should become the tracking item for that half; a matching Rust issue should be opened.

#### Component requirements

All readers of a given workflow must resolve the recorded `store` name to the *same* underlying objects through their own sidecars. Two sidecars each configured with an in-memory component of the same name do not satisfy this, which is why the in-memory store is documented as experiment-only. Durable components must retain objects for as long as any history, forwarded reference, rerun, or descendant can need them.

### Per-language guidance

The envelope, policy, defaults, and errors are fixed; the API shape is not. Each SDK should expose the same five concepts in its own idiom: a `PayloadReference` value, a resolver with `store` / `resolve` / `open` / `delete_payloads`, a way to declare a reference contract on a workflow or activity, per-call `result_as_reference` and `allow_payload_references` options, and reference accessors on workflow state. The sketches below are starting points for the SDK maintainers, not final APIs.

#### Rust

Rust has no binary-store client, so that lands first. The workflow integration then follows the .NET shape most closely because the type system can carry the contract:

```rust
#[derive(Clone, Debug, Serialize, Deserialize)]
pub struct PayloadReference<T = ()> {
    pub store_name: String,
    pub key: String,
    pub size: u64,
    pub content_hash: String,          // lowercase hex sha256
    pub stages: Vec<PayloadStage>,
    pub version: u32,                  // always 1
    _marker: PhantomData<T>,
}

#[async_trait]
pub trait WorkflowPayloadResolver: Send + Sync {
    async fn store<T: Serialize + Send>(&self, value: &T, instance_id: &str) -> Result<PayloadReference<T>, PayloadError>;
    async fn resolve<T: DeserializeOwned>(&self, reference: &PayloadReference<T>) -> Result<T, PayloadError>;
    async fn open(&self, reference: &PayloadReference) -> Result<BoxStream<'static, Result<Bytes, PayloadError>>, PayloadError>;
    async fn delete_payloads(&self, instance_id: &str) -> Result<(), PayloadError>;
}

// Registration
let runtime = WorkflowRuntime::builder(client)
    .with_binary_store_payloads(BinaryStorePayloadOptions {
        store_name: "workflow-payloads".into(),
        write_preference: PayloadWritePreference::PreferReference,
        ..Default::default()
    })
    .build();
```

A workflow or activity whose input type is `PayloadReference<T>` has a reference contract; `Serialize`/`Deserialize` on the reference type implement the envelope exactly (custom `impl`, not derive, so member order and omission of empty `stages` are controlled). `PayloadError` is an enum with one variant per row of the error taxonomy. Call options gain `result_as_reference: bool` and `allow_payload_references: bool`.

#### Python

Python already has `set_binary_file_alpha1`, `get_binary_file_alpha1` (iterable response), and `delete_binary_file_alpha1` on both the sync and `aio` clients. The workflow integration mirrors JavaScript, since runtime type information is similarly weak:

```python
from dapr.ext.workflow import WorkflowRuntime, DaprWorkflowClient
from dapr.ext.workflow.payloads import (
    BinaryStoreWorkflowPayloadResolver, PayloadReference, PayloadWritePreference)

payloads = BinaryStoreWorkflowPayloadResolver(
    store_name="workflow-payloads",
    write_preference=PayloadWritePreference.PREFER_REFERENCE,
    offload_threshold=64 * 1024,
)
runtime = WorkflowRuntime(payloads=payloads)
client = DaprWorkflowClient(payloads=payloads)

@runtime.workflow(name="reference_workflow", input_as_reference=True)
def reference_workflow(ctx, document: PayloadReference):
    return yield ctx.call_activity(measure_reference, input=document,
                                   result_as_reference=False)

@runtime.activity(name="measure_reference", input_as_reference=True)
def measure_reference(ctx, document: PayloadReference) -> int:
    return len(ctx.payloads.resolve(document)["content"])
```

A type annotation of `PayloadReference` alone must not change behavior; the `input_as_reference=True` registration flag is the contract declaration, exactly as in JavaScript. `PayloadReference` is a frozen dataclass with `to_envelope()` / `parse()` class methods. Errors are a small exception hierarchy under `PayloadError`. The resolver must work with both the sync runtime and the `aio` client.

#### Java

Java has `setBinaryFile`, `getBinaryFile` (`Flux<byte[]>`), `getBinaryFileBytes`, and `deleteBinaryFile` on `DaprClient`. Like .NET it can carry the contract in the type:

```java
public final class PayloadReference<T> {
  public static final String SENTINEL = "$daprRef";
  public static final int CURRENT_VERSION = 1;
  // storeName, key, size, contentHash, stages, version
  public String toEnvelope();
  public static <T> Optional<PayloadReference<T>> parse(String wire);  // throws InvalidPayloadEnvelopeException
  public <U> PayloadReference<U> asTyped();
}

public interface WorkflowPayloadResolver {
  <T> Mono<PayloadReference<T>> store(T value, String instanceId);
  <T> Mono<T> resolve(PayloadReference<T> reference, TypeRef<T> type);
  Flux<byte[]> open(PayloadReference<?> reference);
  Mono<Void> deletePayloads(String instanceId);
}

// Registration
WorkflowRuntimeBuilder builder = new WorkflowRuntimeBuilder()
    .withBinaryStorePayloads(BinaryStorePayloadOptions.builder()
        .storeName("workflow-payloads")
        .writePreference(PayloadWritePreference.PREFER_REFERENCE)
        .build());
```

`WorkflowActivity` and `Workflow` implementations whose declared input is `PayloadReference<T>` have a reference contract; the SDK can inspect the generic signature at registration time. `TaskOptions` and `ChildWorkflowTaskOptions` gain `resultAsReference` and `allowPayloadReferences`. `WorkflowInstanceStatus` gains `readInputAsReference()` / `readOutputAsReference()` / `readCustomStatusAsReference()`. Envelope serialization uses the SDK's existing Jackson configuration but bypasses the user-configured workflow serializer.

#### Go

Go has no binary-store client; that lands first on `client.Client` with a chunked `GetBinaryFile` that returns an `io.ReadCloser` (not `[]byte`). The workflow integration lives in `workflow/`:

```go
type PayloadReference struct {
    StoreName   string
    Key         string
    Size        int64
    ContentHash string
    Stages      []PayloadStage
    Version     int
}

func (r PayloadReference) MarshalJSON() ([]byte, error)   // emits {"$daprRef":{...}}
func ParsePayloadReference(wire []byte) (*PayloadReference, bool, error)

type PayloadResolver interface {
    Store(ctx context.Context, value any, instanceID string) (PayloadReference, error)
    Resolve(ctx context.Context, ref PayloadReference, out any) error
    Open(ctx context.Context, ref PayloadReference) (io.ReadCloser, error)
    DeletePayloads(ctx context.Context, instanceID string) error
}

// Registration
r, err := workflow.NewRuntime(
    workflow.WithBinaryStorePayloads(workflow.BinaryStorePayloadOptions{
        StoreName:       "workflow-payloads",
        WritePreference: workflow.PreferReference,
    }))

// Contract declaration and call options
r.RegisterActivity(measureReference, workflow.InputAsReference())
var ref PayloadReference
err = ctx.CallActivity(measureReference, workflow.ActivityInput(ref),
    workflow.ResultAsReference()).Await(&ref)
```

Because Go activities receive input through `ctx.GetInput(&v)`, the contract is declared at registration with an option (as in JavaScript and Python), and `GetInput` into a `*PayloadReference` passes an envelope through while `GetInput` into any other type resolves it. Errors are sentinel values (`ErrPayloadNotFound` and so on) wrapped with `%w` so callers can use `errors.Is`. Sites are a `PayloadKind` bit flag type.

### Cross-SDK conformance

Because the runtime cannot validate envelopes, correctness is proven by tests shared across SDKs rather than by the sidecar. Two fixtures are proposed, hosted in `dapr/proposals/resources` or a small `dapr/sdk-conformance` repository and vendored into each SDK's test tree.

#### Envelope test vectors

A JSON file of cases, each with a wire string and an expected outcome (`ordinary`, `reference` with the expected parsed fields, or `error` with the taxonomy name). The initial set:

| Case | Wire input | Expected |
| --- | --- | --- |
| Minimal unstaged | the example envelope above | `reference`, fields as given |
| Staged | same with `"stages":[{"n":"gzip"}]` | `reference` on parse; `InvalidPayloadEnvelope` (unsupported stage) on buffered resolve |
| Version 2 | `"v":2` | `InvalidPayloadEnvelope` |
| Uppercase hash | `sha256` with `A-F` | `InvalidPayloadEnvelope` |
| Extra sibling property | `{"$daprRef":{...},"x":1}` | `ordinary` |
| Nested in application object | `{"doc":{"$daprRef":{...}}}` | `ordinary` |
| Non-JSON with brace | `{not json` | `ordinary` |
| Non-JSON containing sentinel | `{"$daprRef":` (truncated) | `InvalidPayloadEnvelope` |
| Leading whitespace | `   {"$daprRef":...} ` | `reference` |
| Empty string | \`\` | `ordinary` |
| Round trip | parse then emit | byte-identical to the canonical form |

Every SDK runs every vector in unit tests. .NET and JavaScript should adopt the file as well so a future divergence fails in both.

#### Interop scenario

An end-to-end test using multi-app workflows and a shared durable binary-store component (local file or MinIO):

1. App A (SDK X) schedules a workflow with a 128 KiB input under `PreferReference`.
2. The workflow calls an activity on App B (SDK Y) with `allow_payload_references` set; B receives a reference contract and resolves it.
3. B returns a 128 KiB result that is offloaded at `ActivityOutput`; A requests it as a reference and forwards it to a child workflow on App C (SDK Z) with a reference input contract.
4. C streams the payload with `open`, verifies the hash manually, and returns the byte count inline.
5. A client in SDK X reads the completed state and inspects `getOutputReference()` on the custom status, which was offloaded.

The matrix is run for every ordered triple drawn from the six SDKs once Rust, Python, Java, and Go have implementations; until then the existing .NET and JavaScript pair is the baseline. Passing the matrix is the acceptance gate for an SDK to document itself as interoperable.

This could also be implemented in the Quickstarts as there already exist multiple-app demonstration there and it can use mechanical markdown to prove the correctness from one application to another.

#### Acceptance criteria

- All envelope vectors pass in each SDK.
- The interop scenario passes with the SDK in each of the three roles against .NET and JavaScript.
- Default option values match the table in the resolution-model section.
- Each error in the taxonomy is distinguishable by type, not only by message.
- The SDK's docs page for data references carries the same cross-app compatibility note as the .NET and JavaScript pages, updated to list every SDK that has passed the matrix.

### Alternatives considered

**Runtime-native references.** The durabletask protocol could carry a reference field, letting the runtime (or the workflow actor) offload and resolve payloads so SDKs need no policy code. This is the cleaner long-term design and is the reason the .NET implementation keeps its resolver types internal: so a later switch is invisible to applications. It was not chosen now because it requires changes to `dapr/dapr`, `dapr/durabletask-go`, the protos, and every SDK's generated code, would not work with any sidecar older than the release that adds it, and would still need an SDK-side reference type for the explicit-reference programming model. This proposal is compatible with that future: an envelope-aware runtime could recognize the same `$daprRef` form.

**State store instead of binary store.** Offloading to a state store component would reuse clients every SDK already has. Rejected because state stores have small value limits on common backends, lack streaming reads, and are the same store workflow history already lives in, so it would not relieve the pressure the feature exists to relieve.

**Transparent offloading only (no explicit references).** Simpler API, but the .NET benchmark shows automatic offloading re-downloads on every replay (11 deserializations versus 1) and allocates more than inline. Explicit references are where the gains are, so both modes are kept.

**A protobuf or binary envelope.** More compact, but the envelope must survive inside whatever the application serializer produces and be recognizable without knowing that serializer. A JSON sentinel is the only form that satisfies both for every SDK's default serializer.

**Random or UUID keys.** Avoids hashing on write, but loses idempotent retries and `SkipWriteIfExists`, and the hash is needed anyway for integrity verification.

**Leaving each SDK to converge independently.** This is the status quo. It produced two compatible implementations only because one author wrote both; four more SDKs written by different maintainers from the docs alone would almost certainly diverge on edge cases (whitespace, sentinel sniffing, size semantics with stages, error fallbacks).

### Feature lifecycle

- **Maturity.** Alpha, tracking the Binary Store API. SDK packages mark the resolver APIs alpha and may change them with a minor release until the Binary Store API reaches stable. The *envelope* is treated as stable from this proposal onward: any change bumps `v`.
- **Versioning the envelope.** A reader must reject any `v` it does not implement. A future `v: 2` reader must still read `v: 1`; a writer must not emit `v: 2` while any reader of its histories is `v: 1`-only. The same host-wide rollout rule that applies to enabling references applies to bumping `v`.
- **Compatibility guarantees.** Enabling a resolver with `Preserve` is always safe. Enabling `PreferReference` is a storage-format change, safe for in-flight instances. Changing a `T` contract to a reference contract is an application-contract change requiring a new workflow version. Lowering `MaxInMemoryRehydrationSize` is a compatibility change against retained history.
- **Co-existence.** Inline histories, automatic references, and explicit references can coexist in one history. Once any SDK in a topology has written a reference, every reader in that topology must keep a resolver registered indefinitely.
- **Feature flag.** None at the runtime level. The SDK-level flag is the presence of a registered resolver.
- **Deprecation.** If a runtime-native scheme ships, SDKs keep reading `v: 1` envelopes for at least two minor releases after the runtime scheme is stable, and the resolver's write path becomes a no-op behind the runtime feature.

### Migration between SDKs

A workflow started by one SDK and continued or read by another follows the inbound contract table; nothing in the envelope records which SDK wrote it. The only cross-language requirements beyond the envelope are the ones that already apply to multi-app workflows: compatible serialization of the payload itself and agreed workflow and activity names. SDK docs pages should say which SDKs have passed the interop matrix rather than the current "do not assume other SDKs support this scheme". For a time though, there should also be text encouraging developers to opt-into this capability so apps are preemptively ready to handle data references if observed, even if the preference is to avoid writing them in new history events.

### Implementation plan

Work proceeds in three phases per SDK; phases for different SDKs run in parallel.

| Phase | Rust | Python | Java | Go |
| --- | --- | --- | --- | --- |
| 1. Binary Store client | Add alpha client (set, chunked get, delete, error mapping) | Verify streaming upload, `AlreadyExists` and `NotFound` mapping | Verify streaming upload, `AlreadyExists` and `NotFound` mapping | Add alpha client (set, chunked get, delete, error mapping) |
| 2. Envelope and resolver | `PayloadReference`, custom serde, resolver, options, errors; pass envelope vectors | `PayloadReference` dataclass, resolver (sync and aio), options, errors; pass vectors | `PayloadReference`, resolver, options, errors; pass vectors | `PayloadReference`, `MarshalJSON`, resolver, options, errors; pass vectors |
| 3. Workflow integration | Site hooks in runtime and client, contract declaration, call options, state accessors, docs; pass interop matrix | Same, with `input_as_reference` registration flags | Same, with generic-signature contract detection | Same, with registration options and `GetInput` dispatch |

Phase 2 has no runtime dependency and can be unit-tested entirely offline against the vectors; phase 3 needs a 1.19+ sidecar and a durable binary-store component in CI. Target: phases 1 and 2 for each SDK in the 1.20 cycle, phase 3 as the SDKs' workflow maintainers have capacity, with the interop matrix gating the "interoperable" claim in docs.

## Completion checklist

Shared

- [ ] Envelope test-vector file published and adopted by .NET and JavaScript
- [ ] Interop scenario defined as a runnable multi-app test harness
- [ ] Building-block docs page for workflow payload size links to each SDK's data-reference page

Rust

- [ ] Binary Store alpha client (tracking issue to be opened)
- [ ] `PayloadReference`, resolver, options, errors; vectors pass
- [ ] Workflow runtime and client integration; interop matrix passes
- [ ] Docs page `rust-workflow-data-references`

Python

- [ ] Binary Store client verified against the capability checklist ([#1296](https://github.com/dapr/python-sdk/issues/1296) follow-up)
- [ ] `PayloadReference`, resolver (sync and aio), options, errors; vectors pass
- [ ] `dapr-ext-workflow` integration; interop matrix passes
- [ ] Docs page `python-workflow-data-references`

Java

- [ ] Binary Store client verified against the capability checklist ([#1851](https://github.com/dapr/java-sdk/issues/1851) follow-up)
- [ ] `PayloadReference`, resolver, options, errors; vectors pass
- [ ] Workflow runtime and client integration; interop matrix passes
- [ ] Docs page `java-workflow-data-references`

Go

- [ ] Binary Store alpha client ([#848](https://github.com/dapr/go-sdk/issues/848))
- [ ] `PayloadReference`, resolver, options, errors; vectors pass
- [ ] `workflow` package integration; interop matrix passes
- [ ] Docs page `go-workflow-data-references`

## References

- [Dapr.Actors.Next overview](https://docs.dapr.io/developing-applications/sdks/dotnet/dotnet-actors-next/) (context for the SDK-side approach in the modern .NET SDK)
- [Workflow data references in the .NET SDK](https://docs.dapr.io/developing-applications/sdks/dotnet/dotnet-workflow/dotnet-workflow-data-references/)
- [Workflow data references in the JavaScript SDK](https://docs.dapr.io/developing-applications/sdks/js/js-workflow/js-workflow-data-references/)
- [Binary Store overview](https://docs.dapr.io/developing-applications/building-blocks/binarystore/binarystore-overview/)
- [dotnet-sdk #1934: Data references in Workflow via Binary Store](https://github.com/dapr/dotnet-sdk/pull/1934)
- [js-sdk #905: Add binary store data reference support in workflows](https://github.com/dapr/js-sdk/pull/905)
- [dapr/proposals template](https://github.com/dapr/proposals/blob/main/templates/proposal.md)
- Source consulted on 2026-10-27: `src/workflow/payloads/PayloadReference.ts` and `BinaryStoreWorkflowPayloadResolver.ts` (js-sdk main); `src/Dapr.Workflow/Payloads/PayloadReference.cs`, `PayloadReferenceParser.cs`, `BinaryStorePayloadConverter.cs` (dotnet-sdk master); `dapr/proto/runtime/v1/binarystore.proto`
