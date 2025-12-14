# Add Config to Disable Sidecar Requests to /dapr/config and /dapr/subscribe

* Author(s): Tom Flenner
* State: Waiting for approval to start implementation
* Updated: 20250611

## Overview

This proposal introduces a new configuration option that allows disabling automatic HTTP requests to `/dapr/config` and `/dapr/subscribe` during Dapr sidecar initialization.

Areas affected:

>/area runtime

>/area docs

What is being proposed:

Based on Josh van Leeuwen thoughts :

Add support for a new `--disable-init-endpoints` flag in `dapr run`, as well as a corresponding `dapr.io/disable-init-endpoints annotation`. This flag will accept a string slice of endpoints (e.g., `["config", "subscribe"]`) to disable the automatic loading behavior per sidecar instance.

## Background

Dapr makes automatic HTTP requests to `/dapr/config` and `/dapr/subscribe` endpoints during initialization. While this behavior is helpful for applications relying on programmatic configuration or subscriptions, it introduces several issues for applications that do not use these features:

- Log Noise: Unnecessary requests produce warning logs, leading to confusion for developers who do not expect or understand these failures.

- Unexpected Failures: Under certain network conditions or misconfigured routes, these automatic requests can block declarative subscriptions from being loaded correctly. This results in broken functionality even though the app only intends to use declarative configuration.

As such, it is desirable to make these behaviors configurable and opt-in to better support a variety of deployment and usage patterns.

## Related Items

### Related proposals 
N/A

### Related issues 
https://github.com/dapr/dapr/issues/8224

## Expectations and alternatives

#### In Scope
- New CLI flag: `--disable-init-endpoints` (string slice, e.g., `--disable-init-endpoints=subscribe,config`)
- New sidecar annotation: `dapr.io/disable-init-endpoints: "subscribe,config"`
- Runtime logic to skip calling `/dapr/subscribe` and/or `/dapr/config` as configured
- Documentation updates to describe new options

#### Not In Scope
- Granular error handling improvements beyond skipping the calls

#### Alternative
- Separate booleans for each endpoint: leads to flag sprawl, harder to maintain.
- No change

#### Trade -offs
- N/A

## Implementation Details

### Design

A new CLI flag and annotation will be introduced. Internally, the Dapr runtime will check this configuration before sending requests to the respective endpoints.

Example CLI usage:
```bash
dapr run --app-id myapp --disable-init-endpoints=subscribe,config
```

Example sidecar annotation:
```yaml
annotations:
  dapr.io/disable-init-endpoints: "subscribe,config"
```

In the Dapr runtime, this will translate into logic similar to:
```go
if disabledEndpoints.Has("config") {
    return
}
if disabledEndpoints.Has("subscribe") {
    return
}
```


### Feature lifecycle outline

- Initial implementation will allow opt-out behavior only.
- Backward-compatible: default behavior remains unchanged (both endpoints enabled).
- No deprecation is required.

### Acceptance Criteria

- CLI and annotation config are both supported.
- Behavior is clearly documented and default behavior remains unchanged.
- Sidecars correctly skip calls based on configuration.
- Unit tests added for new logic paths.

## Completion Checklist

What changes or actions are required to make this proposal complete? Some examples:

- [ ] Code changes to runtime
- [ ] CLI support for --disable-init-endpoints
- [ ] Sidecar annotation parsing
- [ ] Unit and integration tests
- [ ] Documentation updates (CLI, annotations, init behavior)
- [ ] Changelog and release notes

Release Note:
```
dapr runtime: Added `--disable-init-endpoints` flag and `dapr.io/disable-init-endpoints` annotation to optionally disable automatic requests to /dapr/config and /dapr/subscribe during sidecar startup. Useful for apps that rely solely on declarative config and subscriptions.
```