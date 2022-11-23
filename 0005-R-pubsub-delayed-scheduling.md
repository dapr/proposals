# Title of proposal 

* Author: Yaron Schneider (@yaron2)
* State: Ready for Implementation
* Updated: 23/11/2022

## Overview

This is a design proposal for the widely requested [delayed message scheduling for pub/sub feature request](https://github.com/dapr/dapr/issues/2675).

The goal of this feature enhancement is to provide developers with the ability to send messages to a pub/sub topic at a given, pre-configured time.

## Background

For background, see this [issue](https://github.com/dapr/dapr/issues/2675). Delayed message scheduling is a common feature in message brokers that helps in postponing essential workloads to a later time, where consumers can then pick them up either after peak hours or when the correct conditions for processing have been met. The use cases for delayed message sending are many, from delaying email sending to processing orders when enough computational capacity on the receiving end has been met to throttle prevention.

### Related issues 

https://github.com/dapr/dapr/issues/2675

## Expectations and alternatives

* What is in scope for this proposal?

This proposal outlines a new metadata key for the [Dapr publish endpoint](https://docs.dapr.io/reference/api/pubsub_api/#publish-a-message-to-a-given-topic) to apply for both HTTP and gRPC APIs that allows developers to set the target date and time for a message to be published to a given topic. 

* What is deliberately *not* in scope?

Using component specific implementations is deliberately not in scope, as it offers no tangible benefits and carries heavy maintenance burden for contributors, approvers and maintainers.

* What alternatives have been considered, and why do they not solve the problem?

The only alternative for users is to create separate topics for delayed messages together with boilerplate code to orchestrate sending/receiving messages from said topics within their code, and then chaining the message flow to the correct topics after the conditions for sending delayed messages have been met. In addition, user code will need to become aware of Dapr specific mechanisms and mirror it in user code.

* What advantages / disadvantages does this proposal have?

This proposal allows delayed message scheduling to be enabled for:

1. Every pub/sub component supported by Dapr without requiring component specific changes
2. Messages sent using Dapr CloudEvents
3. Messages sent using user provided CloudEvents
4. Messages sent using raw payloads
5. Message sent/received by HTTP and gRPC subscribers

## Implementation Details

### Design

The Dapr runtime will use an intermediary topic to store messages that are scheduled for late delivery. Users will specify their intent for a message to be delayed using the existing metadata mechanism (specified as a query param on HTTP1.1 or on the request object for gRPC), the same way we use metadata for TTLs in pub/sub and for many other features. The proposed metadata key to be used is `scheduledSendTime` with the value being a date time string adhering to the ISO-8601 format, the same format supported by Dapr actors reminders and timers.

HTTP example:

```bash
curl -X "POST" http://localhost:3500/v1.0/publish/pubsub/singularity?metadata.scheduledSendTime=2045-11-05T08:15:30-05:00 -H "Content-Type: application/json" -d '{"execute-order": "66"}'
```

Upon receiving a delayed message, the Dapr runtime will examine the due date for the message and publish it to the target topic if time is due. If the time isn't due, the Dapr runtime will hold the message until the time is right to send it without ACKing back to the broker. If the Dapr instance crashes, the message will remain in the broker and be consumed by a different instance.

#### Configuring delayed messaging

Users can enable delayed messaging by adding the following pub/sub component metadata (other fields specific for Redis removed for brevity):

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: pubsub
spec:
  type: pubsub.redis
  version: v1
  metadata:
  - name: "enableDelayedMessaging"
    value: true
```

If not set, the default value is `false` and the feature is bypassed completely.


#### Consuming delayed messages

On startup, the Dapr runtime will subscribe to a topic in the following format: `<app-id>-delayed`. This topic holds all delayed messages for the given application, and has the following benefits:

1. Topics messages are scoped per application and not per component
2. Enables infrastructure transparency in that operators can examine the underlying broker, discover the delayed messages topic and optionally look into the message queue if supported
3. Of secondary importance, this enables users to optionally consume messages directly from the configured topic for advanced scenarios

##### Interaction with namespaced consumer groups

If [namespaced consumer groups](https://docs.dapr.io/developing-applications/building-blocks/pubsub/howto-namespace/#with-namespace-consumer-groups) are enabled, the topic format will result in `<app-id><namespace>-delayed` to allow for a delayed message topic that is scoped both to the app id and the namespace it is running in.

#### Publishing delayed messages

When Dapr receives a message that has a metadata key of `scheduledSendTime`, it will publish it to the delayed message topic in the format described above, if `enableDelayedMessaging` has been enabled on the component.
Since this feature supports all pub/sub components, Dapr will return an error if a metadata key is present and `enableDelayedMessaging` is false or not set on the component.

### Message format

Dapr instances that receive a delayed message need to know the topic and other request level details (content type, etc.). To faciliate this requirement and support both CloudEvents and raw payloads, Dapr will publish a well known, typed proto message to the delayed messages topic in the following structure:

```proto
message DelayedMessage {
  string topic = 1;
  bytes message = 2;
  string contentType = 3;
  map<string, string> metadata = 4;
  bool bulkPublish = 5;
}
```

This provides the Dapr runtime all the information it needs to create a publish request and send it to the message broker.

#### Bulk publish support

If the `DelayedMessage` has the `bulkPublish` field set to `true`, the `message` field will be deserialized as a [BulkMessageEntry](https://github.com/dapr/components-contrib/blob/677663e81ab292d140021e671841ca5f613aaaea/pubsub/requests.go#L26) type and will be sent via the bulk publish route.

#### Components with topic management disabled

Some Dapr pub/sub components allow users to override the creation of topics. In that case, documentation will state to users they need to pre-create the delayed message topic using the correct format.

### Feature lifecycle outline

* Expectations

This feature will start as a preview feature.

* Compatibility guarantees

This feature is backward compatible with previous Dapr versions.

* Feature flags

This feature requires the Preview feature flag.

### Acceptance Criteria

How will success be measured? 

* Performance targets

N/A

* Compabitility requirements

Implementation needs to be compliant with previous versions of Dapr when bypassed (disabled) in terms of API endpoints, behavior, payloads and error codes.

* Metrics

N/A

## Completion Checklist

What changes or actions are required to make this proposal complete?

* Code changes
* Tests added (e2e, unit)
* Documentation