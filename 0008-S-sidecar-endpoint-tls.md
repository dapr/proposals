# Dapr endpoint env and TLS support in SDKs 

* Author(s): Artur Souza (@artursouza)
* State: Ready for Implementation
* Updated: 06/27/2023

## Overview

This is a design proposal to [support remote or shared Dapr APIs](https://github.com/dapr/dapr/issues/6035).

This will allow applications to talk to a remote or shared sidecar, without having to rely on localhost sidecar running per app instance. It means the communication will likely require TLS communication.

## Background

### Motivation
- Applications to communicate to Dapr APIs without a local sidecar.

### Goals
- Dapr users can talk to a remote Dapr API without using CLI or any other tool, by just running the application with environment variables.
- System administrators don't need to have different configurations per application based on programming language, meaning the same environment variables will work with every SDK - exception is when SDK only supports HTTP or GRPC, but sysadmin can simply always setup environment variables for both protocols to guarantee consistency.

### Current Shortfalls
- Inconsistency on setting up Dapr's sidecar endpoint on each SDK.
- Not every SDK support a secure endpoint.

## Related Items

### Related proposals 

Formalizing the proposal here from [this issue](https://github.com/dapr/dapr/issues/6035).

## Expectations and alternatives

* What is in scope for this proposal?
- SDKs to support a consistent pair of environment variables to setup Dapr API
- SDKs to support TLS endpoints for Dapr API

* What is deliberately *not* in scope?
- SSL certificate pinning
- Have consistency of other environment variables for SDK (`DAPR_HOST`, `DAPR_SIDECAR_IP`, etc)
- Have consistency of how Dapr client is instanciated on each SDK


* What alternatives have been considered, and why do they not solve the problem?
1. Leave every SDK as-is:
  - Not every SDK offers an environment variable to configure Dapr endpoint, forcing configuration in code
  - Environment variables per SDK, forcing sysadmin to know about each application's language use
  - Not every SDK supports TLS endpoint
2. Add TLS support only, giving each SDK room to decide on how to expose it to the user
  - Not every SDK offers an environment variable to configure Dapr endpoint, forcing configuration in code
  - Environment variables per SDK, forcing sysadmin to know about each application's language use

* Are there any trade-offs being made? (space for time, for example)
1. Leaving existing environment variables for host and port as-is per SDK, but driving consistency on this new way.
2. Not changing Dapr's DAPR_HOST (or equivalent), DAPR_HTTP_PORT and DAPR_GRPC_PORT.

* What advantages / disadvantages does this proposal have? 
Pros:
- Bring consistency in Dapr API endpoint configuration cross SDKs
- Add support for TLS endpoint

Cons:
- Does not address existing inconsistencies in client instantiation and env variables
- Needs to define a priority between new env variables and old ones

## Implementation Details

### Design

* `DAPR_GRPC_ENDPOINT` defines entire endpoit for gRPC, not just host: `https://dapr-grpc.mycompany.com`
* `DAPR_HTTP_ENDPOINT` defines entire endpoit for HTTP, not just host: `https://dapr-http.mycompany.com`
* Port is parsed from the URL (`https://dapr.mycompany.com:8080`) or via the default port of the protocol used in the URL (80 for `http` and 443 for `https`)
* `DAPR_GRPC_ENDPOINT` and `DAPR_HTTP_ENDPOINT` can be set at the same time since some SDKs (Java, as of now) supports both protocols at the same time and app can pick which one to use.
* `DAPR_GRPC_ENDPOINT` and `DAPR_HTTP_ENDPOINT` must be parsed and the protocol will be used for SDK to determine if communication is over TLS (if not done automatically). In summary, `https` means secure channel.
* Initially, only `http` and `https` protocols should be supported. Other protocols can be added in the future depending on each language support.
* `DAPR_GRPC_ENDPOINT` and `DAPR_HTTP_ENDPOINT` have priority over existing `DAPR_HOST` and `DAPR_HTTP_PORT` or `DAPR_GRPC_PORT` environment variables. Application's hardcoded values passed via constructor takes priority over any environment variable. In summary, this is the priority list (highest on top):
  1. Values passed via constructor or builder method.
  2. `DAPR_GRPC_ENDPOINT` and `DAPR_HTTP_ENDPOINT`
  3. Existing `DAPR_HOST` (or equivalent, defaulting to `127.0.0.1`) + `DAPR_HTTP_PORT` or `DAPR_GRPC_PORT`

#### Example of implementation

https://github.com/dapr/java-sdk/blob/76aec01e9aa4af7a72b910d77685ddd3f0bf86f3/sdk/src/main/java/io/dapr/client/DaprClientBuilder.java#L172C3-L192

### Feature lifecycle outline

* Compatability guarantees
This feature should allow localhost definition too `http://127.0.0.1:3500`, for example.

* Deprecation / co-existence with existing functionality
This feature takes priority over existing (inconsistent) environment variables from each SDK. If app provides a hardcoded value for Dapr endpoint (via constructor, for example), it takes priority.
Use of existing `DAPR_API_TOKEN` environment variables is highly encouraged for remote API but not required.

* Feature flags
N/A

### Acceptance Criteria

How will success be measured? 

* Performance targets
N/A

* Compabitility requirements
Same environment variables work with any SDK - except if protocol is not supported by given SDK.

* Metrics
N/A

## Completion Checklist

What changes or actions are required to make this proposal complete?

* SDK changes
  * Add support for new environment variable
  * Add integration testing on each SDK when possible
* Documentation

