# Workflow History Propagation and History Signing

* Author(s): @joshvanl
* State: Ready for Review
* Updated: 2026-03-04

## Overview

This proposal introduces two complementary features for Dapr Workflows built on the Durable Task Framework:

1. **Workflow History Propagation**: The ability for a workflow to opt-in to sending its accumulated execution history to child workflows and activities, enabling downstream consumers to inspect the full execution context of their caller.

2. **Workflow History Signing**: A chain-of-custody signing mechanism where each identity that processes workflow history signs the accumulated event range (prior history + own additions) using its identity certificate. Signatures are persisted as metadata in the workflow's state store entry alongside the history events (without duplicating them), providing cryptographic provenance, auditability, and tamper-evidence both at rest and in transit.

These features together enable powerful authorization patterns, including **MCP (Model Context Protocol) authorization workflows** that gate access to MCP servers and tool calls based on the verified identity chain and contextual execution history of the caller.

### Areas Affected

- Durable Task Framework protobuf definitions (`durabletask-protobuf`)
- Durable Task Go SDK (`durabletask-go`)
- Dapr Runtime workflow engine (`dapr/dapr`)
- Dapr SDKs (Go, .NET, Java, Python, JavaScript)

## Background

### Motivation

Dapr Workflows support cross-application orchestration where a workflow in App A can invoke child workflows or activities in App B, which may in turn invoke App C. Today, child workflows and activities receive only their direct input - they have no visibility into the broader execution context that led to their invocation.

This creates several gaps:

1. **Authorization decisions lack context**: A child workflow or activity cannot make authorization decisions based on *who* called it and *what happened before*. It only knows its immediate input.

2. **No provenance for cross-app workflows**: When workflows span multiple applications (potentially owned by different teams or organizations), there is no way to cryptographically verify the chain of custody - which identities were involved and what each one did.

3. **Auditability gaps**: Post-hoc analysis of workflow execution across application boundaries requires correlating logs from multiple systems with no cryptographic proof of integrity.

4. **MCP authorization is impossible**: Emerging MCP (Model Context Protocol) patterns require that tool servers can verify not just *who* is calling them, but the full context of *why* - including the chain of AI agent delegations and human approvals that led to the tool invocation.

### Goals

- Enable workflows to opt-in to propagating their execution history to child workflows and activities.
- Provide configurable history propagation scopes (full history, parent-chain-only).
- Implement chain-of-custody signing where each identity signs the accumulated history using its Dapr identity certificate.
- Enable child workflows and activities to read and verify propagated history.
- Support MCP authorization workflows that make access decisions based on verified caller history.
- Maintain backward compatibility - history propagation and signing are fully opt-in.

### Non-Goals

- Replacing or modifying the existing workflow replay mechanism.
- Encrypting workflow history (this proposal covers signing for integrity/provenance, not confidentiality).
- Changing the existing Dapr identity/certificate infrastructure (we leverage it as-is).
- Defining specific MCP authorization policies (this proposal provides the mechanism; policy definition is out of scope).

### Current Shortfalls

**No history visibility in child workflows/activities**: The `OrchestrationContext` and `ActivityContext` interfaces provide no access to parent or ancestor workflow history. A child workflow only knows its input, its instance ID, and its parent instance info (ID, name, app ID).

**No cryptographic provenance**: While `ParentInstanceInfo` records the parent's instance ID and app ID, there is no cryptographic proof that the parent actually executed the claimed history. A malicious or compromised intermediate app could fabricate history.

**Identity is point-to-point only**: Dapr's mTLS ensures that each gRPC call between daprd instances is authenticated, but this proof is not accumulated or propagated. A workflow three hops deep cannot prove the identity chain of all prior hops.

## Related Items

## Expectations and Alternatives

### In Scope

- Protobuf message definitions for history propagation and signing in `durabletask-protobuf`.
- Runtime implementation in `durabletask-go` and `dapr/dapr`.
- SDK surface for opting into history propagation and reading propagated history.
- Chain-of-custody signing using Dapr identity certificates.
- Canonical digest computation for deterministic cross-language signing.
- Verification of history signatures against certificate validity.

### Not In Scope

- History encryption or confidentiality mechanisms.
- Specific MCP authorization policy engines or languages.
- UI/dashboard for visualizing signed history chains.
- History propagation for entity operations.

### Use Cases

#### 1. MCP Authorization Workflows (Single-Hop)

An AI agent application (App A) initiates a workflow that needs to call an MCP server tool hosted by App B. App B's workflow activity needs to verify:
- That App A is an authorized caller.
- That the workflow history shows a human approval step occurred.
- That the agent's reasoning chain (captured as workflow events) justifies the tool call.

```
User -> App A (Agent Workflow)
          |
          | [propagate signed history]
          v
        App B (MCP Tool Activity)
          |
          | [verify: App A's identity signed the history]
          | [verify: history contains human approval event]
          | [authorize tool call]
          v
        MCP Server (execute tool)
```

#### 2. Multi-Hop MCP Delegation

App A delegates to App B, which delegates to App C's MCP server. Each hop adds its identity signature to the accumulated history:

```
User -> App A (Orchestrator)
          |  signs: [history_A]
          |
          v
        App B (Child Workflow)
          |  signs: [history_A + history_B]
          |
          v
        App C (MCP Tool Activity)
          |  verifies full chain: A -> B -> C
          |  checks: all identities are authorized
          |  checks: history contains required approvals
          v
        MCP Server (execute tool)
```

App C can cryptographically verify that:
- App A initiated the workflow and what it did.
- App B received App A's history and added its own processing.
- The full chain of custody is intact and unmodified.

#### 3. Audit and Compliance

A financial services workflow spans multiple microservices. Each service signs its portion of the history. At the end, the complete signed history chain provides a tamper-evident audit trail showing exactly which services processed the request, in what order, and what each one did.

#### 4. Cross-Organization Workflow Trust

Organizations A and B collaborate via Dapr workflows. When Org A's workflow calls Org B's service, Org B can verify the signed history to confirm that the request originated from a legitimate workflow in Org A and passed through the expected processing steps.

#### 5. Conditional Authorization Based on History

A payment processing activity only executes if the propagated history proves:
- The request originated from an authorized frontend application.
- A fraud detection workflow step completed successfully.
- The amount was approved by an activity running in the authorization service.

#### 6. AI Agent Guardrails

An AI agent orchestration workflow propagates its execution history (including LLM interactions, tool calls, and their results) to downstream tool activities. The tool activities can inspect the history to enforce guardrails - for example, refusing to execute a destructive action if the history shows the agent has already performed too many mutations in the current session.

### Alternatives Considered

#### Alternative 1: Metadata-Only Propagation (No History)

Propagate only metadata (caller identity chain, tags) without full history events.

**Pros**: Smaller payload, simpler implementation.
**Cons**: Cannot support use cases requiring inspection of *what happened*, only *who was involved*. Insufficient for MCP authorization that needs contextual history.

**Rejected because**: The core value proposition requires downstream consumers to inspect the execution context, not just identity metadata.

#### Alternative 2: External Audit Log with References

Store signed history in an external audit log and pass references (log IDs) to child workflows.

**Pros**: Keeps workflow payloads small, centralized audit.
**Cons**: Introduces external dependency, adds latency for verification, availability coupling, requires the child to be able to reach the audit log.

**Rejected because**: Adds operational complexity and latency. The in-band propagation approach is self-contained and works in air-gapped environments.

#### Alternative 3: Sign Individual Events Instead of Chunks

Each identity signs each individual history event it produces rather than signing accumulated chunks.

**Pros**: Finer granularity, events can be independently verified.
**Cons**: Does not prove that an identity *saw* the prior history. Identity B signing its own events doesn't prove it received and accepted identity A's events. Breaks the chain-of-custody property.

**Rejected because**: Chain-of-custody requires proving that each participant saw and accepted the accumulated history, not just that they produced their own events.

#### Alternative 4: Sign Raw Protobuf Bytes

Sign the serialized protobuf bytes directly instead of computing a canonical digest.

**Pros**: Simpler, no canonicalization needed.
**Cons**: Protobuf serialization is not deterministic across implementations. Different language SDKs may serialize the same message to different bytes, causing signature verification failures. Field ordering, default values, and unknown fields can all differ.

**Rejected because**: Dapr is a multi-language ecosystem. Signatures must be verifiable across Go, .NET, Java, Python, and JavaScript SDKs. Canonical digest is required for cross-language correctness.

### Trade-offs

| Dimension | Trade-off |
|-----------|-----------|
| **Payload size** | History propagation increases message sizes. Mitigated by configurable scope filters and opt-in behavior. |
| **Latency** | Signing and verification add computational overhead per workflow step. Mitigated by using efficient algorithms (Ed25519/ECDSA) and only signing when opted in. |
| **Complexity** | Adds new concepts to the workflow programming model. Mitigated by making propagation fully opt-in with sensible defaults. |
| **Storage** | History signatures are persisted as metadata alongside history events, increasing state store usage. Each signature adds ~500 bytes (certificate + signature + digests) with zero event duplication. Mitigated by opt-in behavior. |
| **Determinism** | Canonical digest computation must be identical across all SDK languages. Requires careful specification and cross-language testing. |

## Implementation Details

### Design

#### History Propagation Model

History propagation is opt-in at the point of calling a child workflow or activity. The caller specifies a **propagation scope** that determines which history events are included:

| Scope | Description |
|-------|-------------|
| `FULL` | All history events from the current workflow and all ancestor propagated history. |
| `PARENT_CHAIN` | Only events from the direct parent chain (current workflow + parent + grandparent, etc.), excluding sibling activities and sub-orchestrations. |

The propagated history is attached to the child workflow's `ExecutionStartedEvent` or the activity's `TaskScheduledEvent` as a new field. The child can then read this propagated history from its context.

#### History Signing Model (Chain of Custody)

History signing is **persisted as part of the workflow history in the state store**, not computed on-the-fly at propagation time. This means:

- Signatures are created when events are produced, using the identity certificate that is active at that moment.
- The signing metadata (`HistorySignature` entries) is stored alongside the history events in the workflow's runtime state, referencing events by index range rather than duplicating them.
- When history is propagated to a child workflow or activity, events are copied into the `PropagatedHistory` message along with the already-computed signatures. No re-signing is needed.
- The state store contains a tamper-evident record of which identities processed the workflow and when, with zero event duplication.

Each identity that processes workflow history produces a **history signature** that is persisted as a separate actor state key (e.g., `signature-000000`), following the same pattern as `history-NNNNNN` and `inbox-NNNNNN` keys. Crucially, the signature is metadata-only - it does **not** duplicate the history events. Instead, it references a contiguous range of events by index and stores only the signing artifacts:

```
HistorySignature {
    // Index of the first event covered by this signature (inclusive).
    // References events in the existing OldEvents/NewEvents arrays.
    start_event_index: int32

    // Number of events covered by this signature.
    event_count: int32

    // Canonical digest of the previous signature (empty for root)
    previous_signature_digest: bytes

    // Canonical SHA-256 digest of the events in this range
    events_digest: bytes

    // The signing identity's X.509 certificate (PEM)
    certificate: bytes

    // Signature over: SHA-256(previous_signature_digest || events_digest)
    signature: bytes

    // The SPIFFE ID of the signing identity
    spiffe_id: string
}
```

There is no `signed_at` timestamp field. Instead, certificate validity is verified against the **timestamp of the last `HistoryEvent` in the signed range**. Since signing occurs immediately after event production as part of the same transactional state write, the last event's timestamp is the authoritative time of signing. This eliminates a potential manipulation vector - a separate `signed_at` field could be backdated (to impersonate a prior certificate epoch) or forward-dated (to extend validity beyond the certificate's lifetime). By deriving the signing time from the events themselves, which are part of the signed digest, the timestamp cannot be manipulated independently of the signature.

The events themselves are stored once - in the existing `OldEvents`/`NewEvents` fields. The `HistorySignature` is pure metadata that references those events by index range.

The chain works as follows:

1. **Root workflow creation**: When a workflow is created and the first `OrchestratorStarted` event is processed, the runtime computes the canonical digest of those events, signs it with the hosting identity's certificate, and persists a `HistorySignature` as `signature-000000` in the actor state store with `start_event_index=0`, `event_count=N`, and `previous_signature_digest = empty`.
2. **Subsequent orchestrator executions**: Each time the orchestrator is re-invoked (new events arrive), the runtime creates a new `HistorySignature` covering the new event range, chaining it to the previous signature via `previous_signature_digest`. The new signature is persisted as `signature-000001`, etc.
3. **Cross-app child workflow**: When a child workflow runs on a different app (different identity), it creates its own signature chained to the parent's last signature. Since signatures are already persisted, the parent's signing metadata is simply included when propagating.
4. **Verification**: A verifier walks the chain from root to leaf. For each signature, it recomputes the canonical digest of the referenced event range and checks it against `events_digest`, then verifies the cryptographic signature against the certificate. Certificate validity is checked against the timestamp of the last event in the signed range (which is covered by the digest and thus tamper-proof).

**Signing lifecycle within a single workflow execution:**

```
Execution 1 (OrchestratorStarted + ExecutionStarted + TaskScheduled):
  Transactional write to actor state store:
    history-000000 = e0, history-000001 = e1, history-000002 = e2
    signature-000000 = {start=0, count=3, prev_digest=empty, cert=AppA_cert_v1, ...}
    metadata = {historyLength=3, signatureLength=1}

Execution 2 (TaskCompleted + new TaskScheduled):
  Transactional write to actor state store:
    history-000003 = e3, history-000004 = e4
    signature-000001 = {start=3, count=2, prev_digest=digest(sig0), cert=AppA_cert_v2, ...}
    metadata = {historyLength=5, signatureLength=2}
  (Note: cert may have rotated between executions - this is fine,
   each signature uses the cert active at signing time)

Propagation to child workflow:
  -> Copy events [e0..e4] into PropagatedHistory.events
  -> Include [signature-000000, signature-000001] in PropagatedHistory.signatures
  -> Events are copied only at propagation time, not duplicated at rest
```

#### Canonical Digest Computation

To ensure deterministic signing across language SDKs, history events are canonicalized before hashing:

1. **Event Canonicalization**: Each `HistoryEvent` is converted to a canonical byte representation using the following algorithm:
   - Fields are serialized in proto field number order.
   - Default values are explicitly included.
   - `google.protobuf.Timestamp` is serialized as seconds (int64) + nanos (int32) in big-endian.
   - `google.protobuf.StringValue` is serialized as UTF-8 bytes with a length prefix (uint32, big-endian).
   - Map fields are serialized with keys in sorted lexicographic order.
   - The `HistoryEvent.router` field is excluded from canonicalization (it is routing metadata, not semantic history).

2. **Range Digest**: The canonical bytes of all events in the signature's range are concatenated in event order, then hashed with SHA-256.

3. **Signature Input**: `SHA-256(previous_signature_digest || events_digest)` is signed using the identity's private key.

**Signing Algorithm**: Ed25519 (preferred) or ECDSA P-256, matching the algorithm of the Dapr identity certificate issued by Sentry.

#### Protobuf Changes (durabletask-protobuf)

New messages in `orchestrator_service.proto`:

```protobuf
// Propagation scope for workflow history
enum HistoryPropagationScope {
  // No propagation (default)
  HISTORY_PROPAGATION_SCOPE_NONE = 0;
  // Propagate full history from current and all ancestor workflows
  HISTORY_PROPAGATION_SCOPE_FULL = 1;
  // Propagate only direct parent chain events
  HISTORY_PROPAGATION_SCOPE_PARENT_CHAIN = 2;
}

// Signing metadata for a contiguous range of history events.
// This is a metadata-only message - it does NOT contain the events themselves.
// Events are stored once in OldEvents/NewEvents; this message references
// them by index range and stores only the signing artifacts.
message HistorySignature {
  // Index of the first event covered by this signature (inclusive).
  // References events in the workflow's OldEvents/NewEvents arrays.
  int32 start_event_index = 1;

  // Number of events covered by this signature.
  int32 event_count = 2;

  // Canonical SHA-256 digest of the previous signature in the chain.
  // Empty bytes for the root signature (first identity in the chain).
  bytes previous_signature_digest = 3;

  // Canonical SHA-256 digest of the events in this range.
  bytes events_digest = 4;

  // X.509 certificate (PEM-encoded) of the signing identity.
  // This is the Dapr identity certificate (SVID) of the app that
  // produced and signed this range of events.
  bytes certificate = 5;

  // Cryptographic signature over SHA-256(previous_signature_digest || events_digest)
  // using the private key corresponding to the certificate.
  bytes signature = 6;

  // The SPIFFE ID of the signing identity (informational, verified via certificate).
  string spiffe_id = 7;

  // Note: there is no signed_at timestamp. Certificate validity is verified
  // against the timestamp of the last HistoryEvent in the signed range
  // (events[start_event_index + event_count - 1].timestamp). Since that
  // timestamp is covered by events_digest, it cannot be tampered with
  // independently of the signature.
}

// Propagated history attached to child workflows and activities.
// This is the only place where events are copied - at propagation time,
// not at rest. The events are included here because the child/activity
// does not have access to the parent's state store.
message PropagatedHistory {
  // The history events being propagated, copied from the parent's
  // OldEvents/NewEvents at propagation time.
  repeated HistoryEvent events = 1;

  // Ordered chain of signatures covering the propagated events.
  // Each signature references an index range within the events field above.
  repeated HistorySignature signatures = 2;

  // The propagation scope that was used to produce this history.
  HistoryPropagationScope scope = 3;
}
```

#### State Persistence Model

`OrchestrationRuntimeState` is a **runtime-only, in-memory construct** - it is NOT directly serialized to the state store. The actual persistence uses individual actor state keys:

```
Actor State Store:
├── {actorType}||{actorID}||inbox-000000    → HistoryEvent (protobuf)
├── {actorType}||{actorID}||inbox-000001    → HistoryEvent (protobuf)
├── {actorType}||{actorID}||history-000000  → HistoryEvent (protobuf)
├── {actorType}||{actorID}||history-000001  → HistoryEvent (protobuf)
├── {actorType}||{actorID}||customStatus    → StringValue (protobuf)
└── {actorType}||{actorID}||metadata        → WorkflowStateMetadata (protobuf)
```

History signatures MUST follow this same pattern, stored as separate actor state keys:

```
├── {actorType}||{actorID}||signature-000000  → HistorySignature (protobuf)
├── {actorType}||{actorID}||signature-000001  → HistorySignature (protobuf)
```

The `WorkflowStateMetadata` protobuf is extended with a `signatureLength` field to track the number of stored signatures, matching the existing `inboxLength` and `historyLength` pattern.

```protobuf
// Extend existing WorkflowStateMetadata
message WorkflowStateMetadata {
  // ... existing fields (inboxLength, historyLength, generation) ...

  // Number of HistorySignature entries stored.
  uint64 signature_length = 4;

  // Whether history signing is enabled for this workflow instance.
  bool history_signing_enabled = 5;
}
```

The in-memory `wfenginestate.State` struct is extended with a `Signatures []*HistorySignature` field, loaded via bulk get alongside inbox and history events, and saved via the same transactional state operation pattern with `addStateOperations(req, signatureKeyPrefix, ...)`.

The `OrchestrationRuntimeState` in-memory struct gains a corresponding `Signatures` field, populated from the loaded state when reconstructed.

Modifications to existing protobuf messages:

```protobuf
// Add to ExecutionStartedEvent
message ExecutionStartedEvent {
  // ... existing fields ...

  // Propagated history from parent workflow (opt-in).
  // Contains events and signing metadata from the parent's state store.
  // Present only when the parent workflow opted to propagate history.
  optional PropagatedHistory propagated_history = 15;
}

// Add to TaskScheduledEvent
message TaskScheduledEvent {
  // ... existing fields ...

  // Propagated history from the calling workflow (opt-in).
  // Contains events and signing metadata from the workflow's state store.
  // Present only when the workflow opted to propagate history
  // when scheduling this activity.
  optional PropagatedHistory propagated_history = 10;
}

// Add to CreateSubOrchestrationAction
message CreateSubOrchestrationAction {
  // ... existing fields ...

  // Opt-in: propagate workflow history to the child workflow.
  HistoryPropagationScope history_propagation_scope = 10;
}

// Add to ScheduleTaskAction
message ScheduleTaskAction {
  // ... existing fields ...

  // Opt-in: propagate workflow history to the activity.
  HistoryPropagationScope history_propagation_scope = 10;
}
```

#### SDK Surface (Go Example)

**Propagating history when calling a child workflow:**

```go
func MyOrchestrator(ctx *task.OrchestrationContext) (any, error) {
    // Call child workflow with full history propagation and signing
    var result string
    err := ctx.CallSubOrchestrator("ChildWorkflow",
        task.WithSubOrchestratorInput(input),
        task.WithHistoryPropagation(protos.HISTORY_PROPAGATION_SCOPE_FULL),
        task.WithSignHistory(true),
    ).Await(&result)
    return result, err
}
```

**Propagating history when calling an activity:**

```go
func MyOrchestrator(ctx *task.OrchestrationContext) (any, error) {
    // Call activity with parent-chain history propagation
    var result string
    err := ctx.CallActivity("MCPToolCall",
        task.WithActivityInput(input),
        task.WithHistoryPropagation(protos.HISTORY_PROPAGATION_SCOPE_PARENT_CHAIN),
        task.WithSignHistory(true),
    ).Await(&result)
    return result, err
}
```

**Reading propagated history in a child workflow:**

```go
func ChildWorkflow(ctx *task.OrchestrationContext) (any, error) {
    // Access propagated history (nil if not propagated)
    history := ctx.GetPropagatedHistory()
    if history != nil {
        // Inspect caller identities
        for _, sig := range history.Signatures() {
            fmt.Printf("Processed by: %s\n", sig.SpiffeID())
        }

        // Check for specific events in history
        for _, event := range history.AllEvents() {
            // Inspect events for authorization decisions
        }
    }

    // ... workflow logic ...
    return result, nil
}
```

**Reading propagated history in an activity:**

```go
func MCPToolCallActivity(ctx task.ActivityContext) (any, error) {
    // Access propagated history
    history := ctx.GetPropagatedHistory()
    if history == nil {
        return nil, fmt.Errorf("propagated history required for MCP tool calls")
    }

    // Verify required workflow steps occurred
    if !history.ContainsEventType("HumanApproval") {
        return nil, fmt.Errorf("human approval required before MCP tool call")
    }

    // Proceed with MCP tool call
    return executeMCPTool(ctx, input)
}
```

#### Runtime Implementation

**In `durabletask-go`:**

1. **`task/orchestrator.go`**: Extend `CallSubOrchestrator` and `CallActivity` to accept propagation options. When propagation is requested, the `CreateSubOrchestrationAction` and `ScheduleTaskAction` include the propagation scope.

2. **`backend/runtimestate/applier.go`**: Two responsibilities:

   **a. Signature creation:** After the orchestrator produces actions and the applier generates new history events, the applier creates a new `HistorySignature` covering the new events by index range. This signature is chained to the previous signature (if any) already in the in-memory `OrchestrationRuntimeState.Signatures` slice. The signature is computed using the current identity's certificate and appended to the in-memory state. No events are duplicated.

   **b. Assembling propagated history (at propagation time):** When processing `CreateSubOrchestrationAction` or `ScheduleTaskAction` with propagation enabled:
   - Copy the relevant events from the in-memory history into the `PropagatedHistory.events` field (this is the only time events are copied - for transmission to the child).
   - Include the corresponding `HistorySignature` entries, remapping event indices to match the copied events array.
   - If ancestor propagated history exists (this workflow itself received propagated history), prepend those events and signatures.
   - Apply the propagation scope filter to select which events/signatures to include.
   - Attach the resulting `PropagatedHistory` to the child's `ExecutionStartedEvent` or activity's `TaskScheduledEvent`.
   - No re-signing is needed - the signatures already exist.

3. **`task/activity.go`**: Extend `ActivityContext` interface with `GetPropagatedHistory()` method. The propagated history is extracted from the `TaskScheduledEvent`.

**In `dapr/dapr`:**

1. **`pkg/runtime/wfengine/`**: Pass the Dapr security provider to the workflow engine so it can access the identity certificate for signing. The security provider is used at state persistence time (every orchestrator execution), not just at propagation time.

2. **`pkg/runtime/wfengine/state/state.go`**:
   - Extend the `State` struct with a `Signatures []*HistorySignature` field.
   - On load (`LoadWorkflowState`): bulk-get `signature-000000` through `signature-{N}` keys using the `signatureLength` from metadata, same pattern as inbox/history loading.
   - On save (`GetSaveRequest`): call `addStateOperations(req, signatureKeyPrefix, s.Signatures, ...)` to include signature entries in the transactional write, same pattern as inbox/history saving.
   - Extend `WorkflowStateMetadata` with `signatureLength` and `historySigningEnabled`.

3. **`pkg/actors/targets/workflow/orchestrator/`**:
   - **State persistence**: After each orchestrator execution completes, in `saveInternalState()`, the new `HistorySignature` entries are included in the transactional state operation alongside history and inbox entries. Each signature is persisted as an individual actor state key (`signature-000000`, etc.).
   - **Propagation**: When building child workflow start events and activity task events, read signatures from the loaded state and copy the relevant events to assemble the `PropagatedHistory`.

3. **History Verification Utility** (`pkg/runtime/wfengine/historyverify/`):
   - `VerifySignatures(signatures []*HistorySignature, events []*HistoryEvent) error` - walks the signature chain, recomputes the canonical digest of each referenced event range, verifies it against the stored `events_digest`, then checks the cryptographic signature against the certificate and verifies certificate validity against the timestamp of the last event in each signed range.
   - For propagated history: `VerifyPropagatedHistory(history *PropagatedHistory) error` - convenience wrapper that passes `history.events` and `history.signatures`.
   - For at-rest verification: called on workflow load with the stored `history_signatures` and `OldEvents`/`NewEvents`.

#### Sequence Diagram: History Signing at Rest and Propagation

```
App A (Orchestrator)          Dapr Runtime A            State Store          Dapr Runtime B         App B (Activity)
       |                           |                         |                      |                      |
  [Execution 1: workflow starts]   |                         |                      |                      |
       |                           |                         |                      |                      |
       |-- orchestrator returns -->|                         |                      |                      |
       |   actions                 |                         |                      |                      |
       |                    [generate new history events]     |                      |                      |
       |                    [compute canonical digest of      |                      |                      |
       |                     events 0..2]                     |                      |                      |
       |                    [create HistorySignature:         |                      |                      |
       |                     start=0, count=3]                |                      |                      |
       |                    [sign with App A cert]            |                      |                      |
       |                           |-- transactional write -->|                      |                      |
       |                           |   history-000000 = e0   |                      |                      |
       |                           |   history-000001 = e1   |                      |                      |
       |                           |   history-000002 = e2   |                      |                      |
       |                           |   signature-000000=sig0 |                      |                      |
       |                           |   metadata (lengths)    |                      |                      |
       |                           |                         |                      |                      |
  [Execution 2: calls activity with propagation]             |                      |                      |
       |                           |                         |                      |                      |
       |-- CallActivity(input,     |                         |                      |                      |
       |   scope=FULL)             |                         |                      |                      |
       |                           |                         |                      |                      |
       |                    [generate new history events]     |                      |                      |
       |                    [create HistorySignature:         |                      |                      |
       |                     start=3, count=2, chained]      |                      |                      |
       |                    [sign with App A cert]            |                      |                      |
       |                    [copy events into PropagatedHistory                      |                      |
       |                     + include signature metadata]    |                      |                      |
       |                           |-- transactional write -->|                      |                      |
       |                           |   history-000003 = e3   |                      |                      |
       |                           |   history-000004 = e4   |                      |                      |
       |                           |   signature-000001=sig1 |                      |                      |
       |                           |   metadata (updated)    |                      |                      |
       |                           |                         |                      |                      |
       |                           |-- TaskScheduled -------------------------------->|                      |
       |                           |   (PropagatedHistory:                            |                      |
       |                           |    events=[e0..e4]                               |                      |
       |                           |    signatures=[sig1,sig2])                       |                      |
       |                           |                         |               [verify signatures]            |
       |                           |                         |               [pass to activity]             |
       |                           |                         |                      |                      |
       |                           |                         |                      |-- Execute(ctx) ----->|
       |                           |                         |                      |   (history in ctx)   |
       |                           |                         |                      |                      |
       |                           |                         |                      |   [GetPropagatedHistory()]
       |                           |                         |                      |   [check authorization]
       |                           |                         |                      |                      |
       |                           |                         |                      |<-- result -----------|
       |                           |<-- TaskCompleted --------------------------------|                    |
       |<-- result                 |                         |                      |                      |
```

#### Performance Implications

| Concern | Impact | Mitigation |
|---------|--------|------------|
| **Serialization overhead** | History events must be canonicalized and serialized for digest computation. | Canonical form is computed once per signing. Events are already serialized for storage. |
| **Cryptographic overhead** | SHA-256 hashing + Ed25519/ECDSA signing per event range. | Ed25519 signing is ~60,000 ops/sec on modern hardware. Negligible compared to workflow I/O. |
| **Payload size increase** | Propagated history copies events into the message plus ~500 bytes per signature for signing metadata. At rest, only the metadata is added (no event duplication). | Configurable scope filters, depth limits, and size limits. Opt-in only. |
| **Verification overhead** | Chain verification requires validating each signature and its certificate. | O(n) where n = number of signatures (i.e., number of orchestrator executions/hops). Certificate validation can be cached. |
| **Replay impact** | Propagated history is fixed at invocation time and does not change during replays, so it does not affect replay determinism. | No mitigation needed - this is a positive property. |

#### Security Considerations

1. **Certificate validity window**: Signatures are verified against the timestamp of the last event in the signed range. If the certificate was valid at that event's timestamp, the signature is considered valid even if the certificate has since expired or been rotated. Since signing occurs in the same transactional write as event persistence, the last event's timestamp is the authoritative signing time. This eliminates a separate `signed_at` field as a manipulation vector.

2. **History truncation attacks**: A malicious intermediate could attempt to drop signatures from the propagated history. Chain verification detects this because each signature's `previous_signature_digest` must match the actual digest of the prior signature. Dropping a signature breaks the chain.

3. **History replay attacks**: An attacker could capture a valid signed history and replay it in a different context. Mitigation: the `events_digest` covers event-specific data including instance IDs and timestamps, binding the history to a specific execution.

4. **State store tampering**: Since `HistorySignature` entries are persisted in the state store, an attacker with direct state store access could attempt to modify history events. However, any modification would invalidate the signature's `events_digest`. The runtime can optionally verify the stored signature chain integrity when loading workflow state, recomputing the canonical digest of each referenced event range and checking it against the stored digest. This detects tampering at rest without requiring any event duplication.

5. **Certificate rotation between executions**: A workflow may execute multiple times (due to replay), and the identity certificate may rotate between executions. Each signature is created with the certificate active at that execution's time. Verifiers check certificate validity against the last event's timestamp in each signed range, not the current time. Different signatures in the same workflow may use different certificates - this is expected and correct.

### Feature Lifecycle Outline

#### Alpha

- History propagation with `FULL` and `PARENT_CHAIN` scopes.
- History signing with chain-of-custody model.
- Go SDK support only.
- Feature gated behind `WorkflowHistoryPropagation` feature flag (disabled by default).
- Canonical digest specification finalized.
- No performance guarantees.
- API may change based on feedback.

#### Beta

- .NET and Python SDK support.
- History verification utilities in SDK.
- Feature flag enabled by default.
- Performance benchmarks published.
- Backward-compatible API stability.

#### Stable

- All SDK support (Go, .NET, Java, Python, JavaScript).
- Cross-language canonical digest verification tests.
- Performance targets met (< 1ms overhead per signing operation).
- API frozen.
- Documentation and tutorials complete.

### Compatibility

- **Backward compatible**: History propagation is fully opt-in. Existing workflows that do not opt in are unaffected. The new protobuf fields are `optional` and do not change existing serialization.
- **Forward compatible**: Older runtimes that do not understand `PropagatedHistory` will ignore the unknown fields (standard protobuf behavior). Child workflows on older runtimes will simply not see propagated history.
- **Cross-version**: A newer parent can propagate history to an older child. The older child ignores it. An older parent cannot propagate history to a newer child (the feature must be opted in by the caller).

### Drawbacks

1. **Complexity**: Adds new concepts (propagated history, history signatures, chain verification) to an already complex workflow system. Developers must understand when and why to use history propagation.

2. **Payload bloat**: Propagating full history of long-running workflows with many events could result in large payloads, increasing network and storage costs.

3. **Certificate coupling**: History signing is coupled to Dapr's identity certificate infrastructure. Environments that do not use Dapr's mTLS (e.g., running without Sentry) cannot use history signing.

4. **Partial history trust**: If a workflow propagates history with `PARENT_CHAIN` scope, the child cannot verify that sibling events were not omitted. The scope filter is applied by the sender, so the child trusts the sender's filtering.

## Acceptance Criteria

### Performance Targets

- History signing overhead: < 1ms per signature for event ranges with up to 1000 events.
- History verification overhead: < 5ms for a chain of 10 chunks.
- Canonical digest computation: < 0.5ms for 100 events.
- No measurable impact on workflows that do not opt into history propagation.

### Compatibility Requirements

- Cross-language canonical digest parity: Go, .NET, Python, Java, JavaScript must produce identical digests for the same history events.
- Signatures produced by any SDK must be verifiable by any other SDK.
- Existing workflows without propagation must have zero performance regression.

### Test Matrix

| Scenario | Test |
|----------|------|
| Propagation disabled (default) | Verify no propagated history on child/activity context |
| Full scope propagation | Parent history fully visible in child |
| Parent-chain scope propagation | Only direct ancestor events visible |
| Custom scope propagation | Filter predicate correctly selects events |
| Signing with Ed25519 | Signature created and verified correctly |
| Signing with ECDSA P-256 | Signature created and verified correctly |
| Cross-language digest parity | Same events produce same digest in all SDKs |
| Cross-language signature verification | Signature from Go SDK verified by .NET SDK (and all pairs) |
| Chain verification (3+ hops) | Multi-hop chain verified end-to-end |
| Tampered history detection | Modified events cause verification failure |
| Truncated chain detection | Removed chunks cause verification failure |
| Expired certificate at last event timestamp | Verification fails |
| Rotated certificate (valid at last event timestamp) | Verification succeeds |
| Large history with size limit | Propagation rejected when exceeding limit |
| Propagation depth limit | Propagation stops at configured depth |
| Mixed signing (some hops signed, some not) | Unsigned ranges are clearly identified |
| Cross-app propagation | History propagated across app boundaries via actor routing |
| Replay determinism | Propagated history does not affect orchestrator replay |
| At-rest signing persistence | Signed chunks persisted and loadable across workflow executions |
| At-rest tamper detection | Modified events in state store detected on load when verification enabled |
| Certificate rotation across executions | Different certs used for different chunks, all verifiable |
| Workflow rehydration with signed state | Workflow loads correctly with history signatures in state |

## Completion Checklist

- [ ] Protobuf changes in `durabletask-protobuf` (new messages, modified messages)
- [ ] Canonical digest specification document (byte-level format)
- [ ] `durabletask-go` implementation:
  - [ ] History propagation in orchestrator context
  - [ ] History propagation in activity context
  - [ ] `PropagatedHistory` reader API
  - [ ] Chain-of-custody signing at state persistence time
  - [ ] `HistorySignature` actor state key persistence (`signature-NNNNNN` keys)
  - [ ] `WorkflowStateMetadata` extension (`signatureLength`, `historySigningEnabled`)
  - [ ] State load/save in `wfenginestate.State` for signature entries
  - [ ] Chain verification (at rest and on propagated history)
  - [ ] Canonical digest computation
- [ ] Dapr runtime changes (`dapr/dapr`):
  - [ ] Wire security provider to workflow engine for signing
  - [ ] History assembly in orchestrator actor
  - [ ] Feature flag `WorkflowHistoryPropagation`
  - [ ] Size and depth limit enforcement
- [ ] SDK changes:
  - [ ] Go SDK: `WithHistoryPropagation`, `WithSignHistory`, `GetPropagatedHistory`, `VerifyChain`
  - [ ] .NET SDK (beta)
  - [ ] Python SDK (beta)
  - [ ] Java SDK (stable)
  - [ ] JavaScript SDK (stable)
- [ ] Cross-language canonical digest test suite
- [ ] Unit tests for signing and verification
- [ ] Integration tests for cross-app propagation
- [ ] E2E tests for MCP authorization scenario
- [ ] Performance benchmarks
- [ ] Documentation in `dapr/docs`
