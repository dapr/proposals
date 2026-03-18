# Workflow External Event Timer Tracking

Author: Albert Callarisa @acroca
Status: Proposed
Introduced: 2026-03-18

## Overview

This proposal adds structured metadata to timers created during `waitForExternalEvent` calls in Dapr Workflows, and enforces that a timer is **always** created when waiting for an external event, including indefinite waits. The change spans the durabletask-protobuf schema, all Dapr Workflow SDKs, and the runtime.

Affected areas: Runtime, SDKs.

## Background

When a workflow calls `waitForExternalEvent`, it suspends execution until a named external event is delivered. Callers may optionally specify a timeout after which the wait expires. Internally, this timeout is implemented as a `CreateTimerAction` scheduled by the SDK.

### Problem

Today it is not possible to determine from the outside which workflow instances are currently blocked waiting for an external event, or which event they are waiting for. This makes it difficult to:

- Answer the question "who is waiting for event X?" without replaying full workflow history.
- Build tooling, dashboards, or SLO monitors that surface stuck or long-waiting workflows.
- Identify event names associated with in-flight timers when inspecting state.

There are two gaps in the current implementation:

1. **No reliable way to identify the origin of a timer.** Some SDKs already set the timer name to the external event name, but this is not enough to differentiate a timer created by `waitForExternalEvent` from a regular timer that happens to have the same name. There is no structured way to tell them apart.

2. **No timer when there is no timeout.** SDKs currently skip creating a timer entirely when `waitForExternalEvent` is called without a timeout. Indefinite waits are completely invisible in the timer store.

## Expectations and Alternatives

### In scope

- Proto schema changes to `CreateTimerAction` and `TimerCreatedEvent`.
- SDK changes to populate the new `origin.external_event` field.
- SDK changes to always emit a timer, using an infinite duration for indefinite waits.

### Out of scope

- A new API or CLI for listing workflows waiting for a given event. This is a natural follow-up but is not part of this proposal.

### Alternatives considered

**Add a new event type to the workflow history.** This would grow the history size, which we want to keep as lean as possible. Attaching the origin to the existing timer event is a better fit.

**Query workflow history directly.** This is not possible. Replaying history requires executing the workflow code, which runs in the SDK on the customer side. We have no access to it, so we cannot determine the current state by replaying history alone.

## Implementation Details

### Proto Schema

Two messages in `durabletask-protobuf` are extended with a `origin` oneof:

```protobuf
// orchestrator_actions.proto
message CreateTimerAction {
    google.protobuf.Timestamp fireAt = 1;
    optional string name = 2;

    oneof origin {
        TimerOriginExternalEvent external_event = 3;
    }
}

// history_events.proto
message TimerCreatedEvent {
    google.protobuf.Timestamp fireAt = 1;
    optional string name = 2;
    optional RerunParentInstanceInfo rerunParentInstanceInfo = 3;

    oneof origin {
        TimerOriginExternalEvent external_event = 4;
    }
}

// Indicates the timer was created as a timeout for a waitForExternalEvent call.
message TimerOriginExternalEvent {
    // The name of the external event being waited on, matching EventRaisedEvent.name.
    string name = 1;
}
```

The `oneof` is used to keep the field extensible for future timer origin types (e.g. child workflow timeouts) without breaking existing consumers.

### SDK Behavior Changes

All Dapr Workflow SDKs MUST be updated as follows:

#### 1. Always populate `origin.external_event`

When constructing a `CreateTimerAction` as part of a `waitForExternalEvent` call, the SDK MUST set:

```
createTimerAction.origin.external_event.name = <event_name>
```

where `<event_name>` is the name passed to `waitForExternalEvent`.

#### 2. Always create the timer

SDKs MUST always emit a `CreateTimerAction` when `waitForExternalEvent` is called, regardless of whether a timeout was provided by the caller.

- **With timeout:** behavior is unchanged except for the new `origin` field.
- **Without timeout:** create a timer with `fireAt` set to the maximum representable future timestamp (e.g. `9999-12-31T23:59:59Z`). This timer effectively never fires, but its presence in the timer store makes the wait observable.

### Backwards Compatibility

The `origin` field is an `optional` `oneof`, so existing consumers that do not understand it will ignore it. The change is backwards compatible on the wire.

Existing workflows that resume after an SDK upgrade will not have the new field on previously-created timers. This is acceptable.

SDKs that previously skipped timer creation for indefinite waits will now emit a timer. Consumers of workflow history MUST tolerate timers that were not previously present for these waits.

### Feature Lifecycle

| Phase  | Criteria |
|--------|----------|
| Alpha  | Proto updated, at least one SDK updated, field populated on timer creation |
| Beta   | All SDKs updated, existing tests pass, new tests cover the field and infinite-timer case |
| Stable | Shipped in a minor Dapr release, no regressions observed across SDK test suites |

### Acceptance Criteria

- [ ] `CreateTimerAction.origin.external_event.name` is set to the event name on every timer created by `waitForExternalEvent`.
- [ ] A timer is emitted for `waitForExternalEvent` calls with no timeout, using a far-future `fireAt` timestamp.
- [ ] All SDKs have test coverage for both cases (with timeout, without timeout).
- [ ] Existing `waitForExternalEvent` tests continue to pass.

## Completion Checklist

- [ ] `durabletask-protobuf`: update proto schema
- [ ] Runtime: propagate `origin` from `CreateTimerAction` to `TimerCreatedEvent`
- [ ] SDK: Go
- [ ] SDK: .NET
- [ ] SDK: Java
- [ ] SDK: Python
- [ ] SDK: JavaScript
- [ ] Documentation: update `waitForExternalEvent` reference to note that a timer is always created
