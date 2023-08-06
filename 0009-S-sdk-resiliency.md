# Resiliency in Dapr SDKs

* Author(s): Artur Souza (@artursouza)
* State: Ready for Implementation
* Updated: 08/06/2023

## Overview

This is a design proposal to [support resiliency when SDKs invoke remote Dapr APIs](https://github.com/dapr/dapr/issues/6609).

This will allow applications to talk to a remote or shared sidecar, without having to rely on custom retry and timeout logic in the user's application.

## Background

### Motivation
- Applications to communicate to remote Dapr APIs when there is communication degradation.
- Applications to communicate to sidecar when there is degradation of the sidecar's health.

### Goals
- Dapr users can talk to a remote Dapr API without having to implement resiliency logic in their app.
- System administrators don't need to have different configurations per application based on programming language, meaning the same configuration will work with every SDK.

### Current Shortfalls
- Applications need to implement resiliency (retry and timeout) on top of existing SDK.

## Related Items

### Related proposals 

Formalizing the proposal here from [this issue](https://github.com/dapr/dapr/issues/6609).

## Expectations and alternatives

* What is in scope for this proposal?
- SDKs to support a consistent (and small) set of environment variables to configure resiliency on SDKs
- Consistent set of retriable errors for gRPC and HTTP APIs.

* What is deliberately *not* in scope?
- Circuit Breaking
- A highly configurable spec for resiliency policies (like the CRD in runtime)


* What alternatives have been considered, and why do they not solve the problem?
1. Leave every SDK as-is:
  - Undetermined behavior when sidecar is down or too slow. For example, the Java SDK simply gets stuck forever if there is no response from the sidecar (tested with ToxiProxy).
  - Timeout and retry needs to be implemented at the user's application.
2. Add retry only
  - Undetermined behavior when sidecar is down or too slow. For example, the Java SDK simply gets stuck forever if there is no response from the sidecar (tested with ToxiProxy).
3. Let each SDK decide how to handle this.
  - Inconsistent behavior and configuration for resiliency, requiring system admins to know specifics of each SDK.

* Are there any trade-offs being made? (space for time, for example)
1. Simplification of retry policy, having an opinionated setting for most configuration.
2. No support for Circuit Breaking or API health check prior to calling the Dapr API.

* What advantages / disadvantages does this proposal have? 
Pros:
- Bring consistency and simple set of configuration points that work cross SDKs
- Document expected behavior for SDKs regarding timeout and retries

Cons:
- See trade-offs mentiond above.

## Implementation Details

### Design

* `DAPR_API_MAX_RETRIES` defines the maximum number of retries, SDKs can determine which strategy will be implemented (linear, exponential backoff, etc). `0` is the default value and means no retry. `-1` or any negative value means infinite retries.
* `DAPR_API_TIMEOUT_MILLISECONDS` defines the maximum waiting time to connect and receive a response for an HTTP or gRPC call. Defaults to `0`. `0` (or negative) are handled as "undefined" and calls might hang forever on the client side. This setting is the timeout for each API invocation and not the timeout of the aggregated time for retries. This setting can be used without retries.
* All environment variables can be overwritten via parameters to the Dapr client or at a per-request basis, in the following order (higher priority on top):
  1. Parameter when instantiating a Dapr client object
  2. Properties or any other language specific configuration framework.
  3. Environment variables
* SDK to retry if error is on connection.
* SDK to retry in case of the following retriable codes:
  * gRPC: DEADLINE_EXCEEDED, UNAVAILABLE.
  * HTTP: 408, 429 (respect `Retry-After` header), 500, 502, 503, 504
* The same client should still be usable if the API goes down but is restored after any arbitrary amount of time. In other words, the unavailability of the Dapr API should not require the application to restart.

#### Example of implementation

https://github.com/dapr/java-sdk/pull/889

### Feature lifecycle outline

* Compatability guarantees
Retries and timeouts should be disabled by default.

* Deprecation / co-existence with existing functionality
If customers prefer to have a more fine tuned resiliency logic, they can still achieve so by disabling the SDK resiliency and use a 3rd party library to handle retries with custom logic.

* Feature flags
Retries and timeouts are disabled by default with the value `0`.

### Acceptance Criteria

How will success be measured? 

* Performance targets
N/A

* Compabitility requirements
Same environment variables work with any SDK.
SDKs to pass a new compatibility test (in runtime).

* Metrics
N/A

## Completion Checklist

What changes or actions are required to make this proposal complete?

* SDK changes
  * Add support for new environment variable
  * Add new parameters when instantiating a new Dapr client
  * Add integration testing on each SDK when possible (can use ToxiProxy)
* Compatibility tests
  * Implement a compatibility test in runtime (similar to what was done for actor invocation)
* Documentation

