# Actors gRPC Bi-Directional Streaming

* Author(s): @joshvanl

## Overview

This proposal introduces support for the Actor runtime API to enable bi-directional streaming in gRPC.
Client-initiated connections will make Actor requests to the Dapr sidecar, and the Dapr sidecar will stream app invocations back to the client over the same connection.
Each stream will accommodate a single Actor Type.

## Background

Currently, app Actor API calls are made over unary HTTP or gRPC calls.
All Daprd Actor invocations of the app occur over unary HTTP requests, including discovering hosted Actor Types and configuration.
This requires the app to both have an open port, and be routable by Daprd to receive these requests.
Additionally, there is no current mechanism for dynamic Actor Type discovery.

By implementing the Actor API over a gRPC bi-directional stream, the app does not need to open an HTTP port to receive Actor invocations.
This approach also enables dynamic Actor Type registration.

## Implementation Details

The protos that implement the new Actor stream runtime reside in the new proto package `dapr.proto.actors.v1`.

```proto
// Actors service provides APIs for user applications to interact with the Dapr Actor
// runtime and receive actor messages.
service Actors {
  // Stream is the bi-directional streaming RPC for actors to receive messages
  // and send responses.
  rpc Stream(stream Request) returns (stream Response) {}
}
```

### Initial Message

Each gRPC stream connected to Daprd will first register its Actor Type along with that type’s entity configuration.
Specifically, the following fields for that type will be included:

```go
type EntityConfig struct {
    ActorIdleTimeout           string
    DrainOngoingCallTimeout    string
    DrainRebalancedActors      bool
    Reentrancy                 ReentrancyConfig
    RemindersStoragePartitions int
}
```

Upon successful registration, which involves Daprd registering the Actor Type with the Actor runtime, the stream will be used to send and receive Actor invocations.
Registration involves adding that type to the internal actor table factories, targeting the gRPC app stream, and broadcasting the hosted actor type to placement.

Similar to the `SubscribeTopicEventsAlpha1` RPC, the Actors runtime will take a `oneof` request message, with an initial message resembling `[SubscribeTopicEventsRequestAlpha1](https://github.com/dapr/dapr/blob/776209e56e12fbff4bac68fad0c7f28e0eaf6ec2/dapr/proto/runtime/v1/dapr.proto#L450)`.

If multiple initial streams of the same Actor Type are registered, they must all contain the same initial message; otherwise, subsequent streams will be rejected.
The first message from a client on a stream must be the initial message.
The initial request will contain the following message:

```proto
// RequestInitial is the initial message containing details for
// configuring actor runtime APIs.
message RequestInitial {
  string type = 1; // Required: Actor Type
  optional google.protobuf.Duration idle_timeout = 2;
  optional bool drain_rebalanced = 3;
  optional google.protobuf.Duration drain_ongoing_call_timeout = 4;
  optional RequestInitialReentrancyConfig reentrancy_config = 5;
}

message RequestInitialReentrancyConfig {
  optional int32 max_stack_depth = 1;
}
```

### Actor API Requests

After the initial message, the client sends API requests to Daprd or processes received message events.

```proto
message Request {
  oneof actor_request_type {
    RequestInitial initial = 1;
    RequestRequest request = 2;
    RequestProcessed processed = 3;
  }
}

message RequestRequest {
  uint64 uid = 1; // Unique identifier per request
  RequestAPI request = 2;
}

message RequestAPI {
  oneof api {
    RequestInvokeActor invoke_actor = 1;
    RequestRegisterTimer register_timer = 2;
    RequestUnregisterTimer unregister_timer = 3;
    RequestRegisterReminder register_reminder = 4;
    RequestUnregisterReminder unregister_reminder = 5;
    RequestGetState get_state = 6;
    RequestExecuteStateTransaction execute_state_transaction = 7;
    RequestPublishMessage publish_message = 8; // Actor pubsub not yet implemented.
  }
}
```

Each outbound `RequestRequest` message includes a `uid` field, which is a simple incrementing `uint64`.
This field must start at `1` and increment by `1` for each new request.
If a message is received from Daprd with a `uid` that is not `+1` of the previous message, the stream will be terminated.

For all inbound messages, the client must send a corresponding processed event with the same `uid`.

```proto
message RequestProcessed {
  uint64 uid = 1;
  RequestProcessedResponse response = 2;
}

message RequestProcessedResponse {
  oneof response {
    RequestProcessedResponseInvokeActor invoke_actor = 1;
    RequestProcessedResponseTimer timer = 2;
    RequestProcessedResponseReminder reminder = 3;
    RequestProcessedResponsePubSubMessage pubsub_message = 4;
  }
}
```

### Actor Inbound Messages

Daprd sends messages containing responses to API requests and event messages over the bi-directional stream.
These similarly contain a `uid` to track the request-response pair.

```proto
message Response {
  oneof actor_response_type {
    ResponseInitial initial = 1;
    ResponseResponse response = 2;
    ResponseProcessed processed = 3;
  }
}

message ResponseResponse {
  uint64 uid = 1;
  ResponseAPI api = 2;
}

message ResponseAPI {
  oneof api {
    ResponseInvokeActor invoke_actor = 1;
    ResponseRegisterTimer register_timer = 2;
    ResponseUnregisterTimer unregister_timer = 3;
    ResponseRegisterReminder register_reminder = 4;
    ResponseUnregisterReminder unregister_reminder = 5;
    ResponseGetState get_state = 6;
    ResponseExecuteStateTransaction execute_state_transaction = 7;
    ResponseSubscribePubSub subscribe_pubsub = 8; // Actor pubsub not yet implemented.
    ResponsePublishPubSubMessage publish_pubsub_message = 9; // Actor pubsub not yet implemented.
  }
}

message ResponseProcessed {
  uint64 uid = 1;
  ResponseProcessedResponse response = 2;
}

message ResponseProcessedResponse {
  oneof response {
    ResponseProcessedResponseInvokeActor invoke_actor = 1;
    ResponseProcessedResponseTimer timer = 2;
    ResponseProcessedResponseReminder reminder = 3;
    ResponseProcessedResponsePubSubMessage pubsub_message = 4;
  }
}
```

### Feature Lifecycle Outline

The runtime needs to be updated to implement these APIs and integrate them into the existing Actor machinery.
All Actor-supported SDKs must also be updated to track request-response pairs using the `uid`.

Once implemented, the existing Actor API should be deprecated and eventually removed in favor of this new approach.
