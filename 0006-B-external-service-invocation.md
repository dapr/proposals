# External Service Invocation 

* Author(s): Samantha Coyle (@sicoyle)
* State: Ready for Implementation
* Updated: 04/06/2023

## Overview

This is a design proposal for the requested [external service invocation feature](https://github.com/dapr/dapr/issues/4549).

The goal of this feature enhancement is to provide developers with a way to invoke any service of their choosing,
using the existing building blocks provided by Dapr.

## Background

### Motivation
We want Dapr users to be able to invoke external,
non-Daprized services with ease and flexibility.

### Goals
Implement a change into `dapr/dapr` that facilities a seamless Dapr UX to allow for external service invocation using existing building blocks and feature sets.

### Current Shortfalls
Currently, we have the service invocation API that allows for Dapr users to use the invoke API on the Dapr instance.
This provides many features as part of the service invocation building block such as HTTP & gPRC service invocation,
service-to-service security, resiliency, observability, access control, namespace scoping, load balancing, and service discovery.
However, the current implementation does not allow for external service invocations - which is a real bummer for many Dapr users.

To remind everyone of the work around many Dapr users use, there is the HTTP binding.
Dapr users can create an HTTP binding with their external URL specified,
but this approach has many downfalls that yield a less-than-desirable developer experience.

For additional background information,
please refer to the [external service invocation feature request](https://github.com/dapr/dapr/issues/4549).

## Related Items

### Related proposals 

Formalizing the proposal here from [this issue](https://github.com/dapr/dapr/issues/4549).

## Expectations and alternatives

* What is in scope for this proposal?
Feature enhancement to enable external service invocation
using the existing service invocation building block allowing service communication using HTTP protocol.

* What is deliberately *not* in scope?
gRPC invocation as well as additional authentication, to include OAuth2,
is not within scope of this initial proposal and implementation.


* What alternatives have been considered, and why do they not solve the problem?
1. Expanding the existing HTTP Binding.
2. Creation of another HTTP Binding explicitly dedicated to external service invocation keeping in mind the current pain points.

Moving forward with the alternative approaches goes against the motivation and goal of this proposal,
as Dapr users would be missing crucial service invocation features,
be restricted on numerous avenues,
and be forced to continue abiding by an awkwardly clunky workaround.
Additional pros/cons may be found in the [linked issue's discussion](https://github.com/dapr/dapr/issues/4549#issuecomment-1414841151).

* Are there any trade-offs being made? (space for time, for example)
N/A

* What advantages / disadvantages does this proposal have? 
This proposal allows service invocation to be enabled for non-Dapr endpoints.

Pros:
- Extends existing service invocation implementation.
- Same feel as current user invocation process.
- Can leverage existing service invocation features like resiliency, security practices, observability.
- Leveraging a new CRD would keep our CRD setup less cluttered and easier to adjust and add to moving forward.
- With a new CRD you can add/rm endpoints programmatically via kubectl.
- Allows for user overrides such as base URL and related request information at invocation time.

Cons:
- Creation of an additional CRD, thus increasing the duplication of boilerplate code for its setup.
- Need to know external base URL ahead of time to configure, but that may not always be easy for end users.
- Would need to change the Dapr Operator to notify on edits for external endpoints.

## Implementation Details

### Design

How will this work, technically?

Allow configuration of pieces needed for external service invocation through creation of new CRD titled `HTTPEndpoint`.
It is HTTP specific in it's `Kind`.
This has benefits in being obvious upfront that it supports only `http`,
and makes it to where we do not need `spec.allowed.protocols`.
However, it would have the drawback of needing additional CRDs in the future for supporting other protocols such as `gRPC`.
The sample `yaml` file snippet below represents the proposed configuration.

```
apiVersion: dapr.io/v1alpha1
kind: HTTPEndpoint
metadata:
  name: "github"
spec:
  baseUrl: "http://api.github.com"
  authScheme: "token" # or BASIC as default
  headers:
  - name: "Accept-Language"
    value: "en-US"
  - name: "Content-Type"
    value: "application/json"
  - name: "Authorization"
    # Option #1
    secretKeyRef:
      name: "my-secret"
      key: "mymetadataSecret"
    # Option #2
    #value: "base64encodedUser:base64encodedPWD" # put in header as is in this case
auth:
  secretStore: "my-secretstore"
```

Noteworthy caveat:
If `Authorization` header specified,
then we would need validation on `authScheme` to become mandatory to know what to use for the prefix.
This allows for headers to match with the existing HTTP header schema,
thus leading to a better user experience that is straightforward to use.
The term `authScheme` comes from the HTTP headers syntax found [here](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Authorization).

Implementation for external service invocation will sit alongside the existing service invocation building block implementation with API changes to support external invocation.

User facing changes include overriding the URL when calling Dapr for service invocation.
Users will use the existing service invocation API, but instead of using an app ID,
they can use an external URL and  optionally overwrite fields at the time of invocation.

To summarize, there would be two ways of working with external service invocations:
1. The URL format programatically.
This allows for convenience and includes a single HTTP call.
2. HTTPEndpoint resource creation declaratively,
where the `HTTPEndpoint.Name` would be used as the AppId in the existing service invocation URL.

#### Examples

1. URL format overwritten:
`http://localhost:${daprPort}/v1.0/invoke/http://api.github.com/method/`

2. HTTPEndpoint resource creation declaratively using the HTTPEndpoint resource definition above.
`http://localhost:${daprPort}/v1.0/invoke/github/method/`


### Feature lifecycle outline

* Compatability guarantees
This feature is fully compatible with the existing service invocation API.

* Deprecation / co-existence with existing functionality
This feature will require support for external service invocations that will sit alongside and make changes to expand the existing service invocation API.

* Feature flags
N/A

### Acceptance Criteria

How will success be measured? 

* Performance targets
N/A

* Compabitility requirements
This feature will need to be fully compatible with existing service invocation API.
In the case that a user tries to add an `HTTPEndpoint` with the same name as an AppId in the same namespace, 
then the `HTTPEndpoint` will fail to create as names must be unique.

* Metrics
Existing service invocation tracing and metrics capabilities when calling external enpoints will be fully functional.

## Completion Checklist

What changes or actions are required to make this proposal complete?

* Code changes
* Secret resolution
* Tests added (e2e, unit)
* SDK changes (if needed)
* Documentation

