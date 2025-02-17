# Actor PubSub Implementation Proposal

- Author(s): @joshvanl

## Overview

This proposal introduces PubSub capabilities for Dapr Actors, enabling actors to subscribe to messages within their type for a particular topic.
Additionally, actors of a specific type will have the ability to publish messages to a topic.
A single designated PubSub component will be marked as the actor PubSub, similar to how an actor state store is defined.
There can be only one actor PubSub state store per Dapr deployment.

## Background

Currently, Dapr provides PubSub functionality for applications but lacks native PubSub integration for actors.
This limitation means that actors cannot communicate efficiently via publish-subscribe messaging patterns within their type.
By enabling Actor PubSub, actors can directly subscribe to and publish messages to specific topics, improving scalability and communication efficiency.

## Implementation Details

### Actor PubSub Registration

A single PubSub component will be designated as the actor PubSub, similar to how an actor state store is configured.
The metadata configuration will follow the existing pattern:

```yaml
metadata:
  - name: actorpubsub
    value: "true"
```

Only one PubSub component can be marked as the actor PubSub.

### Actor Subscriptions

Actors will be able to subscribe to messages within their type for a particular topic.
Subscriptions will be dynamically declared, and messages for that topic will be routed through the established bi-directional gRPC stream.

Upon deactivation, either during shutdown or actor rebalancing, the subscription to all topics by the actor will be removed.
Similarly, if an actor expires via it's idling window, actor is deactivated, and all subscriptions are removed.
The idling timeout is reset on every subscription message received by the actor.

```proto
message ActorSubscriptionRequest {
  // The pubsub topic
  string topic = 1;

  // The metadata passing to pub components
  //
  // metadata property:
  // - key : the key of the message.
  map<string, string> metadata = 2;

  // dead_letter_topic is the topic to which messages that fail to be processed
  // are sent.
  optional string dead_letter_topic = 3;
}
```

When an actor subscribes to a topic, Dapr will internally create a subscription to:

```
dapr.actors||$namespace||$actortype||$topic-name
```

### Actor Publishing

Actors will be able to publish messages to topics within their type.
The following API will be used for message publishing:

```proto
message ActorPublishRequest {
  // The pubsub topic
  string topic = 1;

  // The data which will be published to topic.
  bytes data = 2;

  // The content type for the data (optional).
  string data_content_type = 3;

  // The metadata passing to pub components
  //
  // metadata property:
  // - key : the key of the message.
  map<string, string> metadata = 4;
}
```

Publishing follows the same topic naming convention:

```
dapr.actors||$namespace||$actortype||$topic-name
```

Messages published by an actor will be delivered to all actors of the same type that are subscribed to the given topic.

### Actor PubSub Message Handling

Subscriptions and message handling will be processed through the existing bi-directional gRPC stream for actor communication.
Messages delivered to subscribed actors will be sent over the established stream with the same payload as [today](https://github.com/dapr/dapr/blob/8cefe6a312867d8517a750d2dc300e10821bdf3a/dapr/proto/runtime/v1/appcallback.proto#L96).

The client response is similarly the same format as [today](https://github.com/dapr/dapr/blob/8cefe6a312867d8517a750d2dc300e10821bdf3a/dapr/proto/runtime/v1/appcallback.proto#L136).

### Non-Actor Publishing

Non-actor clients are able to publish messages to actor subscribers via a new `PublishActorEvent` RPC.
This RPC is close to the existing `PublishEvent` rpc, accept that it takes an `actor_type` field rather than a `pubsub_name` field.
This is because their is 1. only one pubsub associated with actors in that namespace, and 2. the fully qualified topic name is derived via the actor type.

Publishing messages to actors will not invoke, and therefore initialize the existence of an actor, since an actor needs to be subscribing to a topic to receive messages.

```proto
  rpc PublishActorEvent(PublishActorEventRequest) returns (google.protobuf.Empty) {}
}

message PublishActorEventRequest {
  // actor_type is the type of actor to which the message is sent.
  string actor_type = 1;

  // The pubsub topic
  string topic = 2;

  // The data which will be published to topic.
  bytes data = 3;

  // The content type for the data (optional).
  string data_content_type = 4;

  // The metadata passing to pub components
  //
  // metadata property:
  // - key : the key of the message.
  map<string, string> metadata = 5;
}
```

## Feature Lifecycle Outline

This feature should be implemented after the actor bi-directional streaming feature is completed.
The bi-directional stream will serve as the transport mechanism for delivering messages to actors.
