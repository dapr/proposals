# Actor PubSub Implementation Proposal

- Author(s): @joshvanl

## Overview

This proposal introduces PubSub capabilities for Dapr Actors, enabling actors to subscribe to messages for their actor type.
Additionally, actors of a that type have the ability to publish messages to that type, to be delivered to  particular actor IDs.
A single designated PubSub component will be marked as the actor PubSub for that namespace, similar to how an actor state store is defined.
There can be only one actor PubSub Component per namespace.
Actors will be spawned, if they are not already, when they receive an actor PubSub message.

## Background

Today, Dapr provides PubSub functionality for applications but lacks native PubSub integration for actors.
This limitation means that actors cannot communicate efficiently via publish-subscribe messaging patterns within their type.
By enabling Actor PubSub, actors can publish messages to specific actor IDs, improving scalability and communication reliability.

### Considerations

There are a number of scenarios or use cases which could benefit from an "Actor PubSub" feature.
These include implicitly or non-implicitly subscribing to topics as an actor, whether publishing messages will spawn actors, if the message is broadcast to a single or all active actors, and if multiple topics can exist within a single actor type domain.
This proposal describes a single topic per actor type that is implicitly subscribed to by all actors of that type.
Other scenarios can be considered in future iterations.

## Implementation Details

### Component

A single PubSub component will be designated as the actor PubSub, similar to how an actor state store is configured.
It is intentional there can only be a single PubSub to a namespace to ensure that infrastructure is completely abstracted from actor developer consumers.
It is also the case having multiple PubSubs would complicate subscribing, forcing actors to watch for PubSub resources and resubscribe as necessary.
It seems logical to keep parity of single Component for both state and PubSub messaging.

The metadata configuration will follow the existing state store pattern:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
 name: joshvanl
spec:
 type: pubsub.joshvanl
 version: v1
 metadata:
 - name: actorpubsub # Added metadata field for all PubSubs.
   value: "true"
```

Only one PubSub component can be marked as the actor PubSub.
Any PubSub type can be used for actors.

### Actor Subscription

On actor type registration, if the Actor PubSub is available, the Dapr runtime will subscribe to the actor PubSub topic for that actor type.
This topic string will take the format of:

```
dapr.actors||$namespace||$actorType
```

Namespace is included to allow for multi-tenancy support of a single PubSub broker across namespaces.
The app ID is not included as actor types transcend app IDs in a namespace.
Upon un-registration of an actor type, the Dapr runtime will unsubscribe from the actor type's PubSub topic.

### API

To facilitate publishing a message to a actor ID, the following client API will be added to the Dapr runtime.
Notice that the message type is similar to the [existing PublishEvent API](https://github.com/dapr/dapr/blob/955436f45f783e52c9af8c2ae32f7f82a287c39c/dapr/proto/runtime/v1/dapr.proto#L381), but with fields dedicated to actor routing and endpoint invocation.
The metadata field will have the same functionality and uses as the [existing PublishEvent API](https://github.com/dapr/dapr/blob/955436f45f783e52c9af8c2ae32f7f82a287c39c/dapr/proto/runtime/v1/dapr.proto#L381).
A client does not need to be an actor or host that actor type to use this API.
The only requirement is that the actor PubSub is configured.

```proto
service Dapr {
  // PublishActorEventAlpha1 publishes an event to an actor ID.
  rpc PublishActorEventAlpha1(PublishActorEventRequestAlpha1) returns (google.protobuf.Empty) {}
}

// PublishActorEventRequestAlpha1 is the message to publish event data to an
// actor ID.
message PublishActorEventRequestAlpha1 {
  // actor_type is the type of the actor to publish to.
  string actor_type  = 1;

  // actor_id is the ID of the actor_type to publish to.
  string actor_id = 2;

  // method is the endpoint of the actor to invoke with the published data.
  string method = 3;

  // The data which will be published to topic.
  bytes data = 4;

  // The content type for the data (optional).
  optional string data_content_type = 5;

  // The metadata passing to pub components
  //
  // metadata property:
  // - key : the key of the message.
  optional map<string, string> metadata = 6;
}
```

### Event Message Routing

Upon the Daprd runtime receiving an actor PubSub message, the runtime will wrap the message with a CloudEvent envelope as usual.
The PubSub used will be the static PubSub Component marked as the actor PubSub, the topic string built using the format above.
If no actor PubSub is defined, an appropriate typed error will be returned to the client.

The Actor PubSub message CloudEvent envelope will have the following fields added to allow for routing of the message on delivery.

```
dapr.actors.id
dapr.actors.method
```

An example of the CloudEvent envelope:

```json
{
  "specversion": "1.0",
  "type": "com.dapr.event.sent",
  "source": "joshvanl",
  "id": "5929aaac-a5e2-4ca1-859c-edfe73f11565",
  "time": "1970-01-01T00:00:00Z",
  "datacontenttype": "application/json",
  "data": {
    "message": "Hello, World!"
  },
  "dapr.actors.id": "MyActorID",
  "dapr.actors.method": "MyActorMethod"
}
```

Upon the daprd runtime receiving an actor PubSub message, the runtime will unwrap the CloudEvent envelope and route the message to the appropriate actor instance.
The actor type and ID to route is easily extracted from the topic the message was received from, and the ID from the CloudEvent envelope.
The data payload will be serialized just as is for normal subscriptions today.
The method field will be used to invoke the actor at that method with the data payload.
The result of that invocation will determine the response code sent back to the PubSub broker component.

All Daprds who share a hosted actor type will subscribe to the same PubSub topic.
These Daprds will receive messages from the subscription for that type, destined for IDs which they often will not be currently hosting.
These messages will be routed (proxied) to the correct daprd host, based on the placement table hash.
Proxying requests like this is already implemented and used in the actor runtime engine today, so no work needs to be added to support this.

SDKs will need to be updated to support publishing actor PubSub messages.
No changes to SDKs need to be made to support _receiving_ actor PubSub messages.

## Feature Lifecycle Outline

- Add Subscription on Actor Type Registration & PubSub Component Configuration
- Add PublishActorEventAlpha1 API
- Update SDKs to support `PublishActorEventAlpha1` API
