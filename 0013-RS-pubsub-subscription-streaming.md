# PubSub Subscription Streaming

* Author(s): @joshvanl
* State: Ready for Implementation
* Updated: 2024-03-05

## Overview

This is a design proposal to implement a new Dapr runtime gRPC and HTTP API for subscription streaming.
This new gRPC & HTTP API will allow an application to subscribe to a PubSub topic and receive messages through this RPC.
Applications will be able to dynamically subscribe and unsubscribe to topics, and receive messages without opening a port to receive incoming traffic from Dapr.

## Background

Dapr supports applications subscribing to PubSub topic events.
These subscriptions can be configured either:
- `programmatically` by returning the subscription config on the app channel server on app health ready, or
- `declaratively` via Subscription yaml manifests in Self-Hosted or Kubernetes mode.

Today, it is not possible to dynamically update the subscription list without restarting Daprd, though hot reloading for Subscription manifests is [planned](https://github.com/dapr/dapr/issues/7139).
It is common for users to want to dynamically subscribe and unsubscribe to topics inside their applications based on runtime conditions.
In the cases where Dapr is not running as a sidecar, users often do not want to open a public port or create a tunnel in order to receive PubSub messages from Dapr.

A streaming Subscription API will allow applications to dynamically subscribe to PubSub topics and receive messages without opening a port to receive incoming traffic from Dapr.

## Expectations and alternatives

This proposal outlines the gRPC & HTTP streaming API for subscribing to PubSub topics.
This proposal does _not_ address any hot-reloading functionality to the existing programmatic or declarative subscription configuration.
Using a gRPC streaming API is the most natural fit for this feature, as it allows for first class long-lived bi-directional connections to Dapr to receive messages.
A supplementary WebSocket based HTTP API is useful for applications which do not have a gRPC client available or HTTP WebSockets are preferred.
These messages are typed RPC giving the best UX in each SDK.
Once implemented, this feature will need to be implemented in all Dapr SDKs.

## Solution

### gRPC

Rough gRPC PoC implementation: https://github.com/dapr/dapr/commit/ed40c95d11b78ab9a36a4a8f755cf89336ae5a05

The Dapr runtime gRPC server will implement the following new RPC and messages:

```proto
service Dapr {
  // SubscribeTopicEventsAlpha1 subscribes to a PubSub topic and receives topic events
  // from it.
  rpc SubscribeTopicEventsAlpha1(stream SubscribeTopicEventsRequestAlpha1) returns (stream TopicEventRequestAlpha1) {}
}

// SubscribeTopicEventsRequest is a message containing the details for
// subscribing to a topic via streaming.
// The first message must always be the initial request. All subsequent
// messages must be event responses.
message SubscribeTopicEventsRequestAlpha1 {
  oneof subscribe_topic_events_request_type {
    SubscribeTopicEventsSubscribeRequestAlpha1 request = 1;
    SubscribeTopicEventsResponseAlpha1 event_response = 2;
  }
}

// SubscribeTopicEventsSubscribeRequest is the initial message containing the
// details for subscribing to a topic via streaming.
message SubscribeTopicEventsSubscribeRequestAlpha1 {
  // The name of the pubsub component
  string pubsub_name = 1 [json_name = "pubsubName"];

  // The pubsub topic
  string topic = 2 [json_name = "topic"];

  // The metadata passing to pub components
  //
  // metadata property:
  // - key : the key of the message.
  optional map<string, string> metadata = 3 [json_name = "metadata"];

  // dead_letter_topic is the topic to which messages that fail to be processed
  // are sent.
  optional string dead_letter_topic = 4 [json_name = "deadLetterTopic"];

  // max_in_flight_messages is the maximum number of in-flight messages that
  // can be processed by the subscriber at any given time.
  // Default is no limit.
  optional max_in_flight_messages = 5 [json_name = "maxInFlightMessages"];
}

// SubscribeTopicEventsResponse is a message containing the result of a
// subscription to a topic.
message SubscribeTopicEventsResponseAlpha1 {
  // id is the unique identifier for the subscription request.
  string id = 1 [json_name = "id"];

  // status is the result of the subscription request.
  TopicEventResponseAlpha1 status = 2 [json_name = "status"];
}
```

When an application wishes to subscribe to a topic it will initiate a stream with `SubscribeTopicEventsRequest`, and `Send` the initial request `SubscribeTopicEventsSubscribeRequest` containing the options for the subscription.
Daprd will then setup the machinery to add this gRPC RPC stream to the set of subscribers.
The request contains no route or path matching configuration as all events will be sent on this stream.
Subscription gRPC streams are the highest priority when Daprd determines which publisher a message should be sent to.
Only a single PubSub Topic pair may be subscribed at a single time with this API.
If the first message sent to the server is not the initial request, the RPC will return an error.
If any subsequent messages are not `SubscribeTopicEventsResponse` messages, the RPC will return an error.

When a message is published to the topic, Daprd will send a `TopicEventRequest` message on the stream containing the message payload and metadata.
After the application has processed the message, it will send to the server a `SubscribeTopicEventsResponse` containing the `id` of the message and the `status` of the message processing.
Since multiple messages can be sent and processed in the application at the same time, the event `id` is used by the server to track the status of each individual event.
An event topic response will follow the timeout resiliency as currently exist for subscriptions.

Client code:

```go
	stream, _ := client.SubscribeTopicEventsAlpha1(ctx)
	stream.Send(&rtv1.SubscribeTopicEventsRequestAlpha1{
		SubscribeTopicEventsRequestTypeAlpha1: &rtv1.SubscribeTopicEventsRequest_RequestAlpha1{
			Request: &rtv1.SubscribeTopicEventsSubscribeRequestAlpha1{
				PubsubName: "mypub", Topic: "a",
			},
		},
	})

	client.PublishEvent(ctx, &rtv1.PublishEventRequest{
		PubsubName: "mypub", Topic: "a",
		Data:            []byte(`{"status": "completed"}`),
		DataContentType: "application/json",
	})

	event, _ := stream.Recv()
	stream.Send(&rtv1.SubscribeTopicEventsRequestAlpha1{
		SubscribeTopicEventsRequestType: &rtv1.SubscribeTopicEventsRequest_EventResponseAlpha1{
			EventResponse: &rtv1.SubscribeTopicEventsResponseAlpha1{
				Id:     event.Id,
				Status: &rtv1.TopicEventResponse{Status: rtv1.TopicEventResponse_SUCCESS},
			},
		},
	})

	stream.CloseSend()
```

### HTTP (WebSockets)

Along with a gRPC based streaming API, a WebSocket based HTTP equivalent API will be implemented.
Much like the gRPC API, the HTTP based WebSocket API will follow an initial request-response handshake, followed by a stream of messages to the client with status responses by the client, indexed by the message ID.
The same proto types as using in the gRPC API (but in JSON blobs) will be used for the HTTP API.
The server WebSocket implementation will be based on the [gorilla/websocket](https://github.com/gorilla/websocket) package, as this seems well used, understood and maintained.

The HTTP streaming API will be available at the following endpoint.
As the pubsub and topic information is in the request body, no request configuration is given in the URL.

```
GET: /v1.0-alpha1/subscribe
```

```json
INITIAL_REQUEST (to server) = {
  "pubsubName": "mypub",
  "topic": "a",
  "metadata": {
    "key": "value"
  },
  "deadLetterTopic": "dead-letter-topic",
  "maxInFlightMessages": 10
}

TOPIC_EVENT_REQUEST (to application) = {
  "id": "123",
  "source": "asource",
  "type": "atype",
  "spec_version": "1.0",
  "data_content_type": "application/json",
  "data": "abc",
  "topic": "a",
  "pubsub_name": "mypub",
  "path": "/"
}

TOPIC_EVENT_RESPONSE (to server) = {
  "id": "123",
  "status": {
    "status": "SUCCESS"
  }
}
```

## Completion Checklist

- [ ] gRPC server implementation in daprd
- [ ] API documentation
- [ ] SDK implementations
  - [ ] .Net
  - [ ] Java
  - [ ] Go
  - [ ] Python
  - [ ] JavaScript
