# Actor Metadata in Dapr SDKs

* Author(s): Artur Souza (@artursouza)
* State: Draft
* Updated: 11/21/2024

## Overview

Actor invocations are done by Dapr SDKs and also handled by Dapr SDKs at the server side, unless the app decides to implement the HTTP or gRPC handles directly. This proposal is to allow Actor SDKs to include custom HTTP headers or gRPC metadata to actor method invocations that the app can choose to handle via custom handlers or interceptors.

## Background

### Motivation
- Applications might have custom logic to authorize calls on the server side of actor invocation, requiring calls from a client to include the app's identifier or token.

### Goals
- Dapr users can invoke Dapr actors with custom HTTP headers or gRPC metadata for custom logic on the server-side.

### Non-goals
- Handle custom HTTP headers or gRPC metadata in SDK's runtime.

### Current Shortfalls
- Applications need to wrap custom metadata as attributes of the input object of the actor method, for example:
```java
public class Order {
    private String orderId;
    private String customerName;
    private double orderTotal;
    private Map<String, String> headers;

   ...
}
```

## Expectations and alternatives

* What is in scope for this proposal?
- SDKs to support a map of HTTP headers or gRPC metadata to be added in all actor method invocations.

* What is deliberately *not* in scope?
- Server-side handling of the metadata in SDKs - that is expected to be handled by app.

* What alternatives have been considered, and why do they not solve the problem?
1. Leave every SDK as-is:
  - Users need to include the custom headers or metadata as extra attributes in their business object models.
2. Add headers per request
  - Need to change how actors are invoked by supporting more than 1 param in methods, also breaking the protocol agnostic nature of the actors API.

* Are there any trade-offs being made? (space for time, for example)
1. The HTTP headers or gRPC metadata will be per client, meaning all calls from that client will include the metadata from them.

* What advantages / disadvantages does this proposal have? 
Pros:
- Allows actor clients to include HTTP headers or gRPC metadata while keeping the protocol agnostic nature of the Actor SDK runtime.

Cons:
- Does not support metadata values per request.

## Implementation Details

### Design

When instantiating an Actor client or Actor proxy on any of the SDKs, the user might optionally pass a map of string of strings with headers or metadata to be passed in every actor invocation coming from that actor client instance.

#### Example of implementation

TBD

### Feature lifecycle outline

* Compatability guarantees
This change does not require the Actor implementation to process custom HTTP headers or gRPC metadata as those are expected to be handled by the app's logic.

* Deprecation / co-existence with existing functionality
If not handles, the custom HTTP headers or gRPC metadata are simply ignored.

* Feature flags
Not needed.

### Acceptance Criteria

How will success be measured? 

* Performance targets
N/A

* Compabitility requirements
Cross SDK calls work with or without custom HTTP headers or gRPC metadata.

* Metrics
N/A

## Completion Checklist

What changes or actions are required to make this proposal complete?

* SDK changes
  * Add support for include HTTP headers or gRPC metadata in gRPC client.

