# Alternate Actor PubSub Implementation Proposal
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
apply here as there would be no "central" pubsub component used. Rather, this proposal is a broadening of the existing
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
subscriptions even to the same pubsub source (potentially applying different filters), this simplifies the concept and
clearly identifies which actor instances need to be rehydrated when applicable messages are received.

Today, Dapr supports applications subscribing to PubSub queue and topic events via:
- `Programmatic subscriptions` by returning the subscription config on the app channel on app health ready
- `Declarative subscriptions` by specifying subscriptions via YAML manifests in Self-Hosted and Kubernetes modes
- `Streaming subscriptions` by dynamically registering new subscriptions for receipt via gRPC-based long-lived connections

Each of these existing concepts remain consistent through this proposal as well and will be demonstrated below.

### Declarative Actor Subscriptions
Declarative subscriptions are described in YAML and provided both at application startup and via "hot reload" thereafter
without needing a restart. These subscriptions can be made to whole actors per the aforementioned restrictions or
to individual actor IDs.

The file format is very similar to the style used for typical PubSub declarative subscriptions, but adds the actor type,
method and optional instance.

This example uses a YAML component file named `my-subscription.yaml`.

```yaml
apiVersion: dapr.io/v2alpha1
kind: ActorSubscription
metadata:
  name: order
spec:
  topic: orders
  routes:
    actor_type: myactor  
    method: processorders
    actor_id:
      - 1
      - 2
      - 3
      - 4
  pusubname: pubsub
```

Here, the subscription is called `order` and:
- Uses the PubSub component called `pubsub` to subscribe to the topic called `orders`
- Defines the routes to the actors by sending all topic messages to the actor type `myactor` by invoking the method
  `processorders` but only on the actor IDs `1`, `2`, `3`, and `4`.
- If the `scopes` field were set, it could further scope this subscription for access only by the apps with the specified
  identifier, even if actor types existed in a broader set of apps.

The implementation on the actor will require annotations provided by the SDKs that register the appropriate routes and
metadata at startup to accommodate these requests. This is described in the following `Programmatic` section - to be
clear, while declarative subscriptions can be modified after runtime by hot-reloading the YAML components, the use
of the static programmatic routes themselves absent this declarative component will not themselves be dynamic. This
is identical to the existing PubSub paradigm in Dapr.


### Programmatic Actor Instance Subscriptions
Programmatic subscriptions are declared within the actors' code and the specific implementation will vary by SDK.
Here, I describe a proposed implementation using the Dapr .NET SDK. It takes an approach similar to that of
subscriptions using the existing PubSub API in order to promote consistency and leverage existing developer investment
in Dapr familiarity.

Because the subscription is executed only when a specific actor instance is executed, a programmatic subscription
cannot subscribe an entire actor type to the published messages. Rather, it will subscribe only the actor ID of the
instance it's running as when initialized.

Accordingly, these subscriptions are intended to be used only within types inheriting from `Actor`.

```cs
[Topic("pubsub", "orders")]
public async Task<Order> ProcessOrders(Order order)
{
    //Logic
    return order;
}
```

Like the declarative subscription, this identifies a route on the actor that:
- Maps to the `pubsub` component
- Subscribes to the `orders` topic or queue name
- Handles subscriptions using the `ProcessOrders` method implemented on the actor

When used without a paired declarative subscription, these routes are identified exclusively at startup and are **not**
to be considered dynamic. This is the
[same guidance](https://docs.dapr.io/developing-applications/building-blocks/pubsub/subscription-methods/#programmatic-subscriptions)
currently given for Dapr PubSub programmatic subscriptions as well, so nothing has changed here.

### Streaming Actor Subscriptions
Like programmatic actor subscriptions, streaming subscriptions are created within the actor code and the specific
implementation will vary by SDK. Here, rather than provide an example of the downstream implementation, I'll stick to
the prototype signature. Unlike programmatic subscriptions, these subscriptions can subscribe a whole actor type or
one or more actor IDs and as such, do not need to be implemented within an actor or host the actor type to use, but
can be.

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
  // Optional: If specified, an array of at least one ilter expression that evaluates to true or false. Delivery 
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
The following are the operations that should be exposed in the API for the management of subscriptions in Dapr.

##### Creating a Subscription
It is expected that at a minimum, an implementation of this proposal includes the capability to create a subscription.

###### HTTP Specification
Create a subscription using HTTP by defining the information contained in the subscription object defined above.

```cURL
POST http://localhost:<daprPort>/v1.0/actor-subscriptions/create
```

Note that the subscription ID can only contain alphanumeric characters, underscores and dashes. Rather than include
redundant parameters in the POST request, all information about the PubSub component name, the topic and other necessary
information will be derived from the request body instead.

```json
{
  // Optional: The unique identifier of the subscription in the scope of the subscription manager. If not set by
  // the developer, it should be set by the runtime when registering the subscription.
  // This is used to perform management operations on the subscription going forward (e.g. get, update, delete)
  "id": "[a subscription manager-scoped unique string]",
  // Required: Identifies the PubSub component and topic from which the event is sourced from
  "source": "<pubsub_component_name>/<pubsub_topic_name>",
  // Optional: The types of events the subscriber is interested in receiving. If specified on a subscription request,
  // all events generated MUST have a CloudEvents type property that matches one of the provided values 
  "types": [
    "io.dapr.event.sent",
    "io.dapr.workflow.activity.started"
  ],
  // Optional: If specified, an array of at least one ilter expression that evaluates to true or false. Delivery 
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
  "sink": "https://dapr.io/events/<actor_type>/<actor_method>/<actor_id>"
}
```

The following are the HTTP response codes that should be provided in response to this request:

| Code | Description | Notes                                                                                                                                                                                                      |
| -- | -- |------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| 202 | Accepted | -                                                                                                                                                                                                          |
| 400 | Request was malformed | Where possible, the runtime should seek to provide information in the body of the response as a string value indicating what about the request was malformed so the SDK can share this with the developer. |
| 409 | ID provided is already in use by another registered subscription | Changing the ID alone is not a guarantee that the request isn't otherwise malformed and won't throw a 400 on the next attempt. |                                                                            |                                                                                                                                                                                                         |                                                                                                                                                                                                        |
| 500 | The request appears to be formatted correctly, but there was an error in the Dapr runtime. |

Additionally, if the runtime returns a `202 Accepted`, the body should contain the following as a JSON value so that the 
SDKs can extract the ID of the subscription even if it was not specified:
```json
{
  "id": "<subscription_identifier>"
}
```

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

###### HTTP Specification
```cURL
GET http://localhost:<daprPort>/v1.0/actor-subscriptions/manage/<actor-subscription-id>
```

Note that a valid subscription ID can only contain alphanumeric characters, underscores and dashes. 

The following are the HTTP response code that should be provided in response to this request:
| Code | Description | 
| -- | -- |
| 200 | Returns the subscription object in the response body as a JSON object |
| 404 | Unable to locate a subscription object with the specified actor subscription identifier |
| 500 | The request appears to be formatted correctly, but there was an error in the Dapr runtime. |

If the runtime returns a `200 OK`, the body should contain the following as a JSON value matching the value of the subscription
as presently registered.
```json
{
  "id": "[a subscription manager-scoped unique string]",
  "source": "<pubsub_component_name>/<pubsub_topic_name>", 
  "types": [
    "io.dapr.event.sent",
    "io.dapr.workflow.activity.started"
  ],
  "filters": [
    {"exact":  {"type":  "io.dapr.event.sent"}},
    {"prefix": {"type":  "io.dapr.event.", "subject":  "<pubsub_topic"} }
  ],
  "sink": "https://dapr.io/events/<actor_type>/<actor_method>/<actor_id>"
}
```

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
It is expected that at a minimum, an implementation of this proposal includes the capability to delete a subscription.

###### HTTP Specification
Delete an existing Actor PubSub subscription by making a `DELETE` request to the following endpoint:

```cURL
DELETE http://localhost:<daprPort>/v1.0/actor-subscriptions/manage/<actor-subscription-id>
```

Note that the subscription ID can only contain alphanumeric characters, underscores and dashes.

The following are the HTTP response codes that should be provided in response to this request:

| Code | Description                                              |
| -- |----------------------------------------------------------|
| 202 | The subscription has been deleted with immediate effect. |
| 404 | The specified subscription identifier could not be located, so no changes were made. |
| 500 | The request appears to be formatted correctly, but there was an error in the Dapr runtime. |

No body should be returned in the response for any of the possible HTTP status codes.

###### gRPC Specification
The following defines the gRPC prototypes necessary to implement this functionality at a minimum:
```protobuf
service Dapr {
  rpc DeleteSubscribeActorEventAlpha1(DeleteSubscribeActorEventRequestAlpha1) returns (google.protobuf.Empty) {}
}

// This message defines the request to make to delete a registered Actor PubSub subscription
message DeleteSubscribeActorEventRequestAlpha1 {
  // Required: The identifier of the subscription to delete
  string subscription_id = 1;
}
```

##### Updating a Subscription
While not absolutely required for an initial implementation of this proposal, having a means of updating a subscription
would be ideal and would at least halve the number of requests otherwise necessary to do the same as a workaround 
(e.g. get a subscription, change the necessary properties, delete the subscription, create a subscription with the 
newly updated properties).

###### HTTP Specification
Update an existing Actor PubSub subscription by replacing the existing subscription with updated properties through a
PUT request with the changed values. Ideally, this should be preceded with a GET request to retrieve the latest 
properties of the request, but this should not be enforced or validated by the runtime as it's outside of scope and 
can be added later if there's appropriate demand (e.g. etag support).

```cURL
PUT http://localhost<daprPort>/v1.0/actor-subscriptions/manage/<actor-subscription-id>
```

Note that the subscription ID can only contain alphanumeric characters, underscores and dashes. The following details
an example of the expected body of this PUT request:

```json
{
  "id": "[a subscription manager-scoped unique string]",
  "source": "<pubsub_component_name>/<pubsub_topic_name>", 
  "types": [
    "io.dapr.event.sent",
    "io.dapr.workflow.activity.started"
  ],
  "filters": [
    {"exact":  {"type":  "io.dapr.event.sent"}},
    {"prefix": {"type":  "io.dapr.event.", "subject":  "<pubsub_topic"} }
  ],
  "sink": "https://dapr.io/events/<actor_type>/<actor_method>/<actor_id>"
}
```

The following are the HTTP response codes that should be provided in response to this request:

| Code | Description |
| 202 | Accepted and will take immediate effect |
| 400 | Request was malformed | Where possible, the runtime should seek to provide information in the body of the response as a string value indicating what about the request was malformed so the SDK can share this with the developer. |
| 404 | Unable to locate an existing subscription for the indicated actor subscription ID so no changes made. This should not register a new subscription as a fallback. |
| 500 | The request appears to be formatted correctly, but there was an error in the Dapr runtime. |


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

###### HTTP Specification
Query the existing Actor PubSub subscriptions by making a GET request to the following endpoint:
```cURL
// List all
GET http://localhost:<daprPort>/v1.0/actor-subscriptions/query

// List by actor type
GET http://localhost:<daprPort>/v1.0/actor-subscriptions/query/actor/<actor-type>

// List by actor instance
GET http://localhost:<daprPort>/v1.0/actor-subscriptions/query/actor/<actor-type>/instance/<actor-id>
```

The following are the HTTP response codes that should be provided in response to this request:
| Code | Description |
| -- | -- |
| 200 | Returns the one or more subscription objects associated with the query in the response body as JSON. |
| 404 | Unable to locate any registered subscriptions for the specified endpoint filter. |
| 500 | The request appears to be formatted correctly, but there was an error in the Dapr runtime. |

If the runtime returns a `200 OK`, the body should contain the following as a JSON array comprised of each of the
subscription objects:

```json
[
  {
    "id": "[a subscription manager-scoped unique string]",
    "source": "<pubsub_component_name>/<pubsub_topic_name>",
    "types": [
      "io.dapr.event.sent"
    ],
    "filters": [
      {"exact":  {"type":  "io.dapr.event.sent"}},
      {"prefix": {"type":  "io.dapr.event.", "subject":  "<pubsub_topic"} }
    ],
    "sink": "https://dapr.io/events/<actor_type>/<actor_method>/<actor_id>"
  },
  {
    "id": "[another subscription manager-scoped unique string]",
    "source": "<pubsub_component_name>/<pubsub_topic_name>",
    "types": [
      "io.dapr.workflow.activity.started"
    ],
    "filters": [
      {"exact":  {"type":  "io.dapr.event.sent"}},
      {"prefix": {"type":  "io.dapr.event.", "subject":  "<pubsub_topic"} }
    ],
    "sink": "https://dapr.io/events/<actor_type>/<actor_method>/<actor_id>"
  }
]
```

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
- Add `SubscribeActorEventAlpha1` API
- Update SDKs to support `SubscribeActorEventAlpha1` API

## Changelog
| Date | Change                                                                                                                                                                                                                |
| -- |-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| 3/31/2025 | Removed existing CEL filtering in favor of strictly using CloudEvent filtering and specification per the Subscriptions API ([0.1 working draft](https://github.com/cloudevents/spec/blob/main/subscriptions/spec.md)) |
| 3/31/2025 | Added HTTP and updated gRPC specification for subscription management                                                                                                                                                 |