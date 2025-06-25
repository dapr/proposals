﻿# Alternate Actor PubSub Implementation Proposal
- Author(s): @joshvanl, @whitwaldo

## Overview
This proposal introduces PubSub capabilities for Dapr Actors, enabling actor instances to subscribe to routed messages
directly regardless of what provided the messages. Actors will be spawned, if they are not already, when they receive
an actor PubSub message that matches their registered routing filters. Because actors are virtual, individual actor
instances can subscribe to a PubSub topic or queue, but they will only receive the messages sent through it following
that subscription.

This proposal differs from the [original Actors PubSub proposal](https://github.com/dapr/proposals/pull/74) in the
following ways:
- There is no designated PubSub component in actors
- Declared subscriptions to a whole actor type will only push messages to instances activated before that point
- Individual actor instances can set up their own subscriptions independent of type
- A given actor instance can subscribe to more than one PubSub component
- A given actor instance can subscribe to the same PubSub component more than once (e.g. to apply multiple filters)
- Actors can programmatically unsubscribe from streaming PubSub subscriptions
- Filtering is applied as part of the subscription so it's readily registered on the runtime for optimal routing
- Rather than change how events are sent into PubSub to specify where they go, this instead adds a subscription in 
actors that require no changes to inbound events. This allows actors to subscribe to any existing PubSub events without 
change even if not originally created with actor subscriptions in mind. This is useful later on when this functionality
is extended to trigger workflows and workflow events.

This proposal seeks to implement more of a broadcast capability facilitating decoupling between the publisher and
any subscribers. The publisher does not know who is subscribing to the message(s) and it's very possible that there's
more than one subscriber across actor types and instances allowing more than one actor to react to the same message.

This does not seek to implement actor message passing or fire-and-forget asynchronous communication. While valuable
in their own right, these ideas are reserved for a separate proposal.

## Background
Today, Dapr provides PubSub functionality for applications but lacks native PuSub integration for actors. This limitation
means that rather than have an actor react specifically to a message it's targeting, additional logic must be written to
observe the trigger and, in turn, activate the actor to perform any necessary downstream actions. By enabling Actor PubSub,
actors can publish and subscribe directly to filtered broadcast messages, improving scalability and communication
reliability.

### Considerations
There are a number of scenarios or use cases which could benefit from an "Actor PubSub" feature. These include implicitly
or explicitly subscribing to topics as an actor and enabling any Dapr-enabled service to publish messages that individual
actors can pick up and execute, potentially in bulk.

Other scenarios can be considered in future iterations.

## Implementation Details

### Component
This proposal diverges from the original in that there is no distinct actor PubSub component. Rather, any existing Dapr
PubSub component is valid to be used as-is without changes. This limits the amount of new work necessitated by developers
to use this new capability and repurposes already existing functionality to enable new use cases.

To be clear, this means that the original proposal's "actorpubsub" annotation on a single pubsub component would **NOT**
apply here as there would be no "central" PubSub component used. Rather, this proposal is a broadening of the existing
PubSub functionality paired specifically with existing Actor use cases.

### Actor Type Subscription
Like the original proposal, on actor type registration, if the specified PubSub component is available, the Dapr runtime
will subscribe to the specified queue/topic indicated. This topic string will take the format of:

```
dapr.actors||$namespace||$actorType
```

Namespace is included to allow for multi-tenancy support for each PubSub broker across namespaces. The app ID is not
included as actor types transcend app IDs in a namespace. Upon un-registration of an actor type, the Dapr runtime will
unsubscribe from all registered PubSub queues/topics for that actor type.

To address the issue of virtual actors theoretically always existing, I propose we take
[the approach of Orleans](https://learn.microsoft.com/en-us/dotnet/orleans/streaming/?pivots=orleans-7-0#stream-semantics)
(which similarly utilizes virtual actors) and specify that it's only when an actor type or instance subscribes to a
stream, once the subscription is resolved, that it will receive all events published after that subscription.

While Orleans' concept of rewindable streams to receive events prior to subscription is intriguing, it is outside
the scope of this proposal.

### Actor Instance Subscription
Actors can also create subscriptions on an individual instance level to improve performance. Rather than invoking
every one of a potentially enormous number of actors to receive a message, by limiting subscriptions to individual
instances, it can dramatically lighten the load of the Dapr runtime in serving these messages, as well as the load on
clients services, especially when paired with runtime-side event routing.

By allowing that individual actors opt-in to receiving not only any message, but allowing multiple
subscriptions even to the same PubSub source (potentially applying different filters), this simplifies the concept and
clearly identifies which actor instances need to be rehydrated when applicable messages are received.

Today, Dapr supports applications subscribing to PubSub queue and topic events via:
- `Programmatic subscriptions` by returning the subscription config on the app channel on app health ready
- `Declarative subscriptions` by specifying subscriptions via YAML manifests in Self-Hosted and Kubernetes modes
- `Streaming subscriptions` by dynamically registering new subscriptions for receipt via gRPC-based long-lived connections

After discussion with other maintainers in the last couple of sync calls, this proposal has been narrowed to only
support streaming subscriptions in its initial implementation. Future need for Programmatic or Declarative subscriptions 
can be evaluated for a future Dapr release.

### Streaming Actor Subscriptions
Like existing Dapr streaming PubSub subscriptions, such subscriptions are created to apply to actor code and the specific
implementation will vary by SDK. Unlike programmatic subscriptions, these subscriptions can subscribe a whole actor 
type or one or more actor IDs and as such, do not need to be within an actor or even host the actor type to be 
registered, but can be.

Such subscriptions are created via either an HTTP or gRPC implementation as described in the implementation section
later in this proposal and resemble the existing streaming pubsub subscription model with changes to support the latest 
versions of the [(WIP) CloudEvents subscription specification](https://github.com/cloudevents/spec/blob/main/subscriptions/spec.md)
and 1.0.2, of the [CloudEvents message specification](https://github.com/cloudevents/spec/blob/v1.0.2/cloudevents/spec.md).

## Publish Actor Event API
There is no proposed change from the existing Dapr PubSub implementation for publishing an event into PubSub 
irrespective of what is subscribed to the event.

### Dapr Subscription Manager
Rather than change how events are published through PubSub, my proposal simply changes how the subscriptions are
set up to facilitate the potentially massive scalability concerns this capability introduces. Consider this: Today
a service may have a dozen subscriptions apiece, so across a dozen such services there are a few hundred such
subscriptions registered with Dapr.

But by allowing subscriptions on a per-actor ID basis, a service with a dozen actor types and thousands (or millions)
of actor instances apiece, each subscribing to one or more PubSub endpoints would massively increase the number
of standing subscriptions.

To that end, I propose that Dapr build a Subscription Manager that's semi-compliant with the current CloudEvents
[subscription specification (working draft)](https://github.com/cloudevents/spec/blob/main/subscriptions/spec.md)

A given subscription manager is responsible for one or more subscriptions assigned it, though this assignment can be
optimized to group like sources and subjects to minimize unnecessary scans for inbound events.

While a previous iteration of this proposal suggested using the Common Expression Language already used with Dapr 
PubSub, it isn't quite as flexible as the proposed filtering mechanism in the above-referenced CloudEvents subscription
specification. Rather than mix and match standards and since the decision was previously made to support CloudEvents, I
propose that we wholeheartedly embrace the CloudEvents specification and drop use of the Common Expression Language (CEL)
routing we've previously used. The rest of this proposal has been updated to reflect this change. Support for 
[CloudEvents SQL expressions](https://github.com/cloudevents/spec/blob/main/cesql/spec.md) is outside of the scope of 
this proposal and is reserved as a future incremental improvement for this functionality.

A subscription, per this specification and using each of the required filter dialects, is modeled using JSON as follows:

```json
{
  // Required: The unique identifier of the subscription in the scope of the subscription manager
  // This is used to perform management operations on the subscription going forward (e.g. get, update, delete)
  "id": "[a subscription manager-scoped unique string]",
  // Optional: Provided by the user when creating the event
  "source": "/myApp/sensors/ww-12345/poll",
  // Optional: The types of events the subscriber is interested in receiving. If specified on a subscription request,
  // all events generated MUST have a CloudEvents type property that matches one of the provided values 
  "types": [
    "io.dapr.event.sent",
    "io.dapr.workflow.activity.started"
  ],
  // Required: Required only for Dapr purposes to provide source metadata for the event (component and topic)
  "config": {
    // Required: The name of the PubSub component the event is sourced from
    "component": "<pubsub_component_name>",
    // Required: The name of the PubSub topic/queue the event is sourced from
    "topic": "<pubsub_topic_name>"
  },
  // Optional: If specified, an array of at least one filter expression that evaluates to true or false. Delivery 
  // should be performedonly if ALL of the expressions evaluate as true.
  // Must support the following dialects:
  // - exact: The value of the matching CloudEvent attribute MUST exactly match the specified value (case-sensitive)
  // - prefix: The value of the matching CloudEvent attribute MUST exactly start with the specified value (case-sensitive)
  // - suffix: The value of the matching CloudEvent attribute MUST exactly end with the specified value (case-sensitive)
  // - all: Expressed as an array of at least one expression, all nested filter expressions must evaluate to true for the expression to be true
  // - any: Expressed as an array of at least one expression, any of the nested filter expressions must evaluate to true for the expression to be true
  // - not: The result of the filter expression is the inverse of the nested expression (e.g. if the expression evaluates as true, the result is false).
  "filters": [
    {"exact":  {"type":  "io.dapr.event.sent"}},
    {"prefix": {"type":  "io.dapr.event.", "subject":  "<pubsub_topic"} }
  ],
  // Required: The destination actor to which events MUST be sent if the filters evaluate as true
  // The last segment, "/<actor_id>" is optional and should only be present for a subscription that is specific to an actor instance
  "sink": "https://dapr.io/events/<actor_type>/<actor_method>/<actor_id>",
  // Required: The identifier of the delivery protocol used by Dapr
  "protocol": "HTTP"
}
```

Whether a PubSub subscription is implemented via the HTTP or gRPC protocols, it should result in the creation of the above
subscription so that there is no substantive difference in how one subscription is registered versus another, lessening
the burden on the runtime to juggle different approaches.

#### Subscription Management Operations
The following are the operations that should be exposed in the API for the management of subscriptions in Dapr. While a
previous iteration of this proposal provided for a REST-like HTTP API, this has since been modified following further
discussion to instead favor a gRPC-only implementation as it's a more natural fit for our goals here:
- A streaming connection requires that there be a specific client on the other end to receive messages
- It allows for long-lived bidirectional connections improving performance

##### Creating a Subscription
It is expected that at a minimum, an implementation of this proposal includes the capability to create a subscription.
More than one sink should be allowed to be specified as part of this message so that multiple actor types and/or actor
instance IDs can be subscribed to in a single request.

While at first glance, it looks like there's an opportunity for deduplication of subscriptions on this front, I would
urge caution in doing so. For example, different subscriptions may specify a sink pointing to a broad actor type and
still others may specify individual actor IDs of that type, after consultation with @joshvanl, I would propose that 
by default all deduplication should occur when a message is received by the runtime as part of the delivery mechanism.
This is discussed further in [this section](#event-message-routing). 

Moreover, while the presence of multiple sinks was considered, as these would be tied to the same `subscription_id` 
going forward and the sinks themselves not individually managed, I would propose that this over-complicates deduplication,
sourcing and management operations. Requiring a one-to-one relationship of one sink to one `subscription_id` simplifies
this on all fronts.
 

###### gRPC Specification
The following defines the gRPC prototypes necessary to implement this functionality at a minimum:

```protobuf
service Dapr {
  rpc SubscribeActorEventAlpha1(SubscribeActorEventRequestAlpha1) returns (SubscribeActorEventResponseAlpha1) {}
}

// The message containing the details for subscribing an actor to a topic via streaming
message SubscribeActorEventRequestAlpha1 {
  // Optional: The identifier of the subscription. If not specified, this will be set by the runtime.
  string subscription_id = 1;
  // Required: The name of the PubSub component the subscription applied to.
  string pubsub_name = 2;
  // Required: The name of the topic being subscribed to.
  string topic_name = 3;
  // Optional: The types of events the subscription is filtered to receiving (evaluated against the CloudEvent `type` property) - must match at least one
  repeated string types = 4;
  // Optional: The filters to apply to the subscription - all must evaluate as true to allow event delivery to subscription
  repeated oneof filters {
    SubscribeActorExactEventFilterAlpha1
    SubscribeActorPrefixEventFilterAlpha1
    SubscribeActorSuffixEventFilterAlpha1
    SubscribeActorAllEventFilterAlpha1
    SubscribeActorAnyEventFilterAlpha1
    SubscribeActorNotEventFilterAlpha1
  } = 5;
  // Required: The destination actor type or instance that is subscribing to the event
  string sink = 6;
}

// Identifies a key/value pair of a CloudEvent attribute key to retrieve the value to compare to the provided
// value with as part of the filter evaluation.
message SubscribeFilterMappingAlpha1 {
  // Required: The name of the CloudEvent attribute to retrieve the value from for filter evaluation
  string attribute_key = 1;
  // Required: The string value to perform the filter evaluation against
  string value = 2;
}

// To evaluate as true, the value of the matching CloudEvents attributes MUST all exactly match with the associated 
// string value specified (case-sensitive).
message SubscribeActorExactEventFilterAlpha1 {
  // Required: The source of each value that should be evaluated as part of this filter expression  
  repeated SubscribeFilterMappingAlpha1 filter;
}

// To evaluate as true, the value of the matching CloudEvents attributes must all exactly match the start of the 
// associated string value specified (case-sensitive).
message SubscribeActorPrefixEventFilterAlpha1 {
  // Required: The source of each value that should be evaluated as part of this filter expression  
  repeated SubscribeFilterMappingAlpha1 filter;
}

// To evaluate as true, the value of the matching CloudEvents attributes must all exactly match the end of the
// associated string value specified (case-sensitive). 
message SubscribeActorSuffixEventFilterAlpha1 {
  // Required: The source of each value that should be evaluated as part of this filter expression  
  repeated SubscribeFilterMappingAlpha1 filter;
}

// To evaluate as true, each of the filter expressions provided must evaluate as true. At least one filter expression
// is required to be provided in this filter.
message SubscribeActorAllEventFilterAlpha1 {
  // Required: The filters to evaluate.
  repeated oneof filters {
    SubscribeActorExactEventFilterAlpha1
    SubscribeActorPrefixEventFilterAlpha1
    SubscribeActorSuffixEventFilterAlpha1
    SubscribeActorAllEventFilterAlpha1
    SubscribeActorAnyEventFilterAlpha1
    SubscribeActorNotEventFilterAlpha1
  } = 1;
}

// To evaluate as true, at least one of the filter expressions provided must evaluate as true. At least one filter
// expression is required to be provided in this filter.
message SubscribeActorAnyEventFilterAlpha1 {
  // Required: The filters to evaluate.
  repeated oneof filters {
    SubscribeActorExactEventFilterAlpha1
    SubscribeActorPrefixEventFilterAlpha1
    SubscribeActorSuffixEventFilterAlpha1
    SubscribeActorAllEventFilterAlpha1
    SubscribeActorAnyEventFilterAlpha1
    SubscribeActorNotEventFilterAlpha1
  } = 1;
}

// This must include one nested filter expression where the result of this filter is to provide the inverse of
// that filter result. For example, if the nested expression returns `true`, this should then return `false`.
message SubscribeActorNotEventFilterAlpha1 {
  // Required: The filter to evaluate.
  oneof filter {
    SubscribeActorExactEventFilterAlpha1
    SubscribeActorPrefixEventFilterAlpha1
    SubscribeActorSuffixEventFilterAlpha1
    SubscribeActorAllEventFilterAlpha1
    SubscribeActorAnyEventFilterAlpha1
    SubscribeActorNotEventFilterAlpha1
  }
}

// The message containing the response to an actor subscription event request if the response returns a 202.
SubscribeActorEventResponseAlpha1 {
  oneof response {
    // Sent if the actor subscription is successfully registered
    SubscribeActorEventSubscriptionSuccesssResponseAlpha1
    // Sent if the actor subscription is not successfully registered
    SubscribeActorEventSubscriptionFailureResponseAlpha1
  } = 1
}

// The message containing information about the successfully registered actor pubsub subscription
SubscribeActorEventSubscriptionSuccesssResponseAlpha1 {
  // Required: The identifier of the registered subscription reflecting the value provided by the user or assigned by the runtime. 
  string id = 1
}

// The message containing information about the failure to register the actor pubsub subscription
SubscribeActorEventSubscriptionFailureResponseAlpha1 {
  // Optional : A message containing additional information about what went wrong. 
  string message = 1
}
```

##### Retrieving a Subscription
It is expected that at a minimum, an implementation of this proposal will include the capability to retrieve an existing
subscription.

###### gRPC Specification
The following defines the gRPC prototypes necessary to implement this functionality at a minimum:

```protobuf
service Dapr {
  rpc GetSubscribeActorEventAlpha1(GetSubscribeActorRequest) returns (GetSubscribeActorResponse) {}
}

// This message contains the request to retrieve a registered Actor PubSub subscription
message GetSubscribeActorRequest {
  // Required: The identifier of the Actor PubSub subscription.
  string subscription_id = 1;
}

// This message contains the response containing a registered Actor PubSub subscription
message GetSubscribeActorResponse {
  // Required: The identifier of the subscription. If not specified, this will be set by the runtime.
  string subscription_id = 1;
  // Required: The name of the PubSub component the subscription applied to.
  string pubsub_name = 2;
  // Required: The name of the topic being subscribed to.
  string topic_name = 3;
  // Optional: The types of events the subscription is filtered to receiving (evaluated against the CloudEvent `type` property) - must match at least one
  repeated string types = 4;
  // Optional: The filters to apply to the subscription - all must evaluate as true to allow event delivery to subscription
  repeated oneof filters {
    SubscribeActorExactEventFilterAlpha1
    SubscribeActorPrefixEventFilterAlpha1
    SubscribeActorSuffixEventFilterAlpha1
    SubscribeActorAllEventFilterAlpha1
    SubscribeActorAnyEventFilterAlpha1
    SubscribeActorNotEventFilterAlpha1
  } = 5;
  // Required: The destination actor type or instance that is subscribing to the event
  string sink = 6;
}
```

##### Deleting a Subscription
It is expected that at a minimum, an implementation of this proposal includes the capability to delete one or
more subscriptions at a time.

###### gRPC Specification
The following defines the gRPC prototypes necessary to implement this functionality at a minimum:
```protobuf
service Dapr {
  rpc DeleteSubscribeActorEventAlpha1(DeleteSubscribeActorEventRequestAlpha1) returns (google.protobuf.Empty) {}
}

// This message defines the request to make to delete one or more registered Actor PubSub subscriptions
message DeleteSubscribeActorEventRequestAlpha1 {
  // Required: The identifier(s) of the subscription(s) to delete
  repeated string subscription_id = 1;
}
```

##### Updating a Subscription
While not absolutely required for an initial implementation of this proposal, having a means of updating a subscription
would be ideal and would at least halve the number of requests otherwise necessary to do the same as a workaround 
(e.g. get a subscription, change the necessary properties, delete the subscription, create a subscription with the 
newly updated properties).

###### gRPC Specification
The following defines the gRPC prototypes necessary to implement this functionality at a minimum:

```protobuf
service Dapr {
  rpc UpdateSubscribeActorEventAlpha1(UpdateSubscribeActorEventRequestAlpha1) returns (SubscribeActorEventResponseAlpha1) {}
}

// The message containing the details for updating an existing actor pubsub subscription. Note that this differs
// from SubscribeActorEventRequestAlpha1 only in that the subscription_id property is required.
message UpdateSubscribeActorEventRequestAlpha1 {
  // Required: The identifier of the subscription being updated.
  string subscription_id = 1;
  // Required: The name of the PubSub component the subscription applied to.
  string pubsub_name = 2;
  // Required: The name of the topic being subscribed to.
  string topic_name = 3;
  // Optional: The types of events the subscription is filtered to receiving (evaluated against the CloudEvent `type` property) - must match at least one
  repeated string types = 4;
  // Optional: The filters to apply to the subscription - all must evaluate as true to allow event delivery to subscription
  repeated oneof filters {
    SubscribeActorExactEventFilterAlpha1
    SubscribeActorPrefixEventFilterAlpha1
    SubscribeActorSuffixEventFilterAlpha1
    SubscribeActorAllEventFilterAlpha1
    SubscribeActorAnyEventFilterAlpha1
    SubscribeActorNotEventFilterAlpha1
  } = 5;
  // Required: The destination actor type or instance that is subscribing to the event
  string sink = 6;
}
```

##### Query for a list of Subscriptions
While not absolutely required for an initial implementation of this proposal, having a means of getting a list of existing
subscriptions would be ideal and especially until we have more specialized states better capable of handling lists of 
values, would simplify management of since-unnecessary subscriptions. This needn't include any detailed filtering in the
initial release and instead should only filter by actor type and instance, if specified at all.

##### gRPC Specification
The following defines the gRPC prototypes necessary to implement this functionality at a minimum:

```protobuf
service Dapr {
  rpc QuerySubscribeActorEventAlpha1(QueryActorEventSubscriptionRequestAlpha1) returns (QueryActorEventSubscriptionResponseAlpha1) {}
}

// The message used to create a query for existing Actor PubSub subscriptions
message QueryActorEventSubscriptionRequestAlpha1 {
  // Optional: The actor type to filter the query by.
  string actor_type = 1;
  // Optional: The actor identifier to filter the query by.
  string actor_id = 2; 
}

// The message containing the response to a query for existing Actor PubSub subscriptions
message QueryActorEventSubscriptionResponseAlpha1 {
  // Required: Should include at least one subscription object, otherwise this query should have returned a 404 status
  repeated QueryActorEventSubscriptionIdentifierAlpha1 subscriptions = 1; 
}

// The message containing the subscription identifier and actor scoping metadata
message QueryActorEventSubscriptionIdentifierAlpha1 {
  // Required: The identifier of the subscription.
  string subscription_id = 1;
  // Required: The type of the actor the subscription is associated with.
  string actor_type = 2;
  // Optional: The identifier of the actor the subscription is associated with.
  string actor_id = 3;
}
```

### Event Message Routing
Upon the Daprd runtime receiving an actor PubSub message, the runtime will wrap the message with a CloudEvent envelope
as usual. The PubSub component used will be the one specified in the subscription (both component and queue/topic name).

If no matching component exists, an appropriately typed error will be returned to the client.

The Actor PubSub message CloudEvent envelope may look like the following example embodying JSON content:
```json
{
  "specversion": "1.0.2",
  "type": "io.dapr.event.sent",
  "source": "<pubsub_component>/<topic_name>",
  "subject": "orders",
  "id": "5929aaac-a5e2-4ca1-859c-edfe73f11565",
  "time": "1970-01-01T00:00:00Z",
  "datacontenttype": "application/json",
  "data": "{\"name\": \"something_happened\"}"
}
```

Here, this diverges from the original proposal in that it specifies the PubSub component and topic in the `source` 
field and leaves the subject and type fields optionally populated by the developer, compliant with the CloudEvent
[specification](https://github.com/cloudevents/spec/blob/v1.0.2/cloudevents/spec.md#subject) for each field.

Upon the daprd runtime receiving an actor PubSub message, the runtime will:
- Unwrap the CloudEvent envelope and evaluate which subscriptions are a match for the indicated `type` value
- Evaluate the `filters` for each matching subscription and discard those subscriptions that don't evaluate as true
for all provided expressions in the subscription filters property
- If there is more than one matching sink for successfully evaluated filters for any given actor ID (e.g. there exists
a subscription for both the actor ID and its actor type for the same method), the sinks should be deduplicated so the
message is sent to each actor ID only a single time per method.
- Send the message to the sink specified in each filtered subscription

The data payload should be serialized per the latest (v1.0.2) 
[CloudEvents specification](https://github.com/cloudevents/spec/blob/v1.0.2/cloudevents/spec.md).

#### Breaking changes from existing Dapr PubSub
When the message is successfully sent to all subscribers from daprd, the PubSub broker component should receive a
delivered response and Dapr resiliency policies should be relied on at that point to ensure that subscriptions receive
the message (or don't). This aligns with the
[delivery guarantees of Orleans](https://learn.microsoft.com/en-us/dotnet/orleans/implementation/messaging-delivery-guarantees).

If an attempted delivery is made to a sink that features an actor type that is not registered in the placement service,
the subscription should be automatically deleted to conserve resources and the action logged for posterity.

#### Unknown implementation details
I am not familiar with how Dapr currently manages streaming subscriptions in a distributed fashion, but I would suggest
augmenting that to support this implementation. Ideally, this should be implemented so that setting up and removing
subscriptions at runtime is a lightweight operation as these subscriptions may be quite short-lived.

As indicated in the proposed HTTP and gRPC specifications above, this implementation should diverge from the existing
Jobs API style in that creating new subscriptions with an ID that's been previously specified should not overwrite 
the existing subscription. Rather, subscriptions must be explicitly deleted or otherwise updated to make changes.

#### Final notes
- All filtering should occur on the runtime to avoid activating actors unnecessarily.
- If a registered sink activates an actor that hasn't previously been activated, it should activate it like any other 
  inbound request would.
- PubSub invocations should follow the turn-based limitations inherent to Dapr Actors already. As such, this may require
  the creation of an inbox of sorts for each actor to handle queued messages pending successful acknowledgement by the
  actor.

SDKs will need to be updated to support receiving Actor PubSub messages.
No changes to SDKs will need to be made to support _sending_ PubSub messages to actors or other endpoints.

## Feature Lifecycle Outline
- Add new prototype implementation to runtime
- Update SDKs to support new API and protos

## Changelog
| Date | Change |
| -- | -- |
| 3/31/2025 | Removed existing CEL filtering in favor of strictly using CloudEvent filtering and specification per the Subscriptions API ([0.1 working draft](https://github.com/cloudevents/spec/blob/main/subscriptions/spec.md)) |
| 3/31/2025 | Added HTTP and updated gRPC specification for subscription management 
| 3/31/2025 | Removed programmatic and declarative subscription types in favor of only supporting streaming subscriptions in this initial release |
| 4/8/2025 | Removed HTTP API implementation in favor of making this proposal strictly gRPC-based |
| 4/8/2025 | Added language clarifying event deduplication and why multiple sinks per subscription isn't part of proposal |
| 4/8/2025 | Supports bulk subscription deletion |