# Support Multiple App Protocols (HTTP and gRPC) in a Single Dapr Application

- Author(s): Tom FLENNER
- State: Draft
- Updated: 2026-02-03

## Overview

This proposal enables Dapr applications to expose both HTTP and gRPC protocols simultaneously for Service Invocation. Currently, each Dapr application is limited to a single `app-protocol`. This change affects the Dapr runtime, CLI, and SDKs.

Related issue: [dapr/dapr#8897](https://github.com/dapr/dapr/issues/8897)

## Background

In Dapr's Service Invocation architecture, communication between sidecars always uses gRPC. The `app-protocol` setting only controls how the **local sidecar communicates with its application**.

```
[App A] <--app-protocol--> [Sidecar A] <==gRPC==> [Sidecar B] <--app-protocol--> [App B]
```

Currently, developers must choose a single protocol per application:

```bash
dapr run --app-protocol http --app-port 3000 ...
# OR
dapr run --app-protocol grpc --app-port 50001 ...
```
****
This limitation forces developers to either duplicate APIs or deploy multiple instances when they need both protocols.

### Use Cases

1. **Hybrid APIs**: Expose HTTP REST for external consumers and gRPC for internal high-performance calls.
2. **Incremental Migration**: Support both protocols during HTTP-to-gRPC transitions.
3. **Protocol-Optimized Endpoints**: Use gRPC for streaming, HTTP for simple REST operations.

## Related Items

### Related proposals

None

### Related issues

- [dapr/dapr#8897](https://github.com/dapr/dapr/issues/8897)

## Expectations and alternatives

### In Scope

- Runtime support for multiple app protocols per application
- CLI support for specifying multiple protocols
- SDK support for protocol selection during invocation
- Backward compatibility with single-protocol configurations

### Out of Scope

- Automatic protocol negotiation or fallback
- Per-endpoint protocol enforcement
- Protocol-level authorization policies

### Alternatives Considered

| Alternative                | Why Rejected                                  |
| -------------------------- | --------------------------------------------- |
| Deploy multiple instances  | Infrastructure cost and complexity            |
| Use protocol proxy/gateway | Adds latency and defeats Dapr's unified model |
| Keep single-protocol model | Doesn't address hybrid use cases              |

## Implementation Details

### Design

#### CLI and Runtime Changes

Accept multiple protocols via comma-separated values:

```bash
dapr run --app-protocol http,grpc --app-port 3000 --app-grpc-port 50001 ...
```

**Validation rules:**

- `http` requires `--app-port`
- `grpc` requires `--app-grpc-port`
- Single protocol continues to use `--app-port` for backward compatibility

#### SDK Changes

SDKs will allow protocol selection during service invocation:

```go
// Go SDK
resp, err := client.InvokeMethod(ctx, "my-app", "method", dapr.WithProtocol("grpc"))
```

```csharp
// .NET SDK
var response = await client.InvokeMethodAsync<Response>(
    "my-app", "method",
    new InvocationOptions { Protocol = "grpc" });
```

**Default behavior**: Use the first protocol specified in `--app-protocol`.

### Feature lifecycle outline

This feature adds support for multiple app protocols simultaneously. Existing single-protocol configurations remain unchanged and fully supported.

### Acceptance Criteria

- [ ] Runtime accepts comma-separated protocols via `--app-protocol`
- [ ] CLI validates protocol/port combinations
- [ ] SDKs can specify protocol during invocation
- [ ] Single-protocol configurations continue to work
- [ ] Clear error messages for misconfigurations

## Completion Checklist

**Runtime (dapr/dapr):**

- [ ] Parse multiple protocols from `--app-protocol`
- [ ] Validate required ports per protocol
- [ ] Unit and integration tests

**CLI (dapr/cli):**

- [ ] Support comma-separated `--app-protocol` values
- [ ] Validation and error messages
- [ ] Unit tests

**SDKs:**

- [ ] Protocol selection API
- [ ] Unit and integration tests

**Documentation (dapr/docs):**

- [ ] Update Service Invocation docs
- [ ] Add multi-protocol configuration examples
