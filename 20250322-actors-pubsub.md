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
- Subscribes to the `orders` topic or queue nme
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
[subscription specification](https://github.com/cloudevents/spec/blob/main/subscriptions/spec.md)

A given subscription manager is responsible for one or more subscriptions assigned it, though this assignment can be
optimized to group like sources and subjects to minimize unnecessary scans for inbound events.

The design choice has been previously made to use the Common Expression Language for routing and this proposal does
not diverge from this decision. Accordingly, as this **DOES** differ from the CloudEvents specification, the example
below changes the `filter` property to `rules` and updates the signature accordingly and puts the sink in the
`rule` body accordingly along with an optional default endpoint to which the delivery is made irrespective of any
other rules.

A subscription, per this specification, is modeled using JSON as follows:

```json
{
  // Required: The unique identifier of the subscription in the scope of the subscription manager
  "id": "[a subscription manager scoped unique string]",
  // Required: The source to which the subsription is related (in Dapr, the pubsub component name
  "source": "<pubsub_component_name>",
  // Required: The topic the subscription is configured for
  "subject": "<pubsub_topic_name>",
  // Optional: An array of filter expressions that evaluate to true or false. Delivery should be performed to the 
  // identified sink only if the CEL expression evaluates to true. Rules are evaluated indepednently of one another.
  "routes": {
    "rules": [
      // If either of the `match` or `sink` properties are provided, both must be
      {
        // Required: The CEL expression that must evaluate as true to match on this route 
        "match": "",
        // Required: The destination actor to which events MUST be sent if the expression is true 
        // The last segment, "/<actor_id>" is optional and should only be present for a subscription that is specific to an actor instance
        "sink": "https://dapr.io/events/<actor_type>/<actor_method>/<actor_id>"
      },
      // Optional: The destination actor to which events MUST be sent if the expression is true
      // The last segment, "/<actor_id>" is optional and should only be present for a subscription that is specific to an actor instance
      {
        "default": "https://dapr.io/events/<actor_type>/<actor_method>/<actor_id>"
      }
    ]
  },
  // Required: The identifier of the delivery protocol, whether HTTP or gRPC
  "protocol": "HTTP"
}
```

Whether a PubSub subscription is implemented via the HTTP or gRPC protocols, it should result in the creation of the above
subscription so that there is no substantive difference in how one subscription is registered versus another, lessening
the burden on the runtime to juggle different approaches.

#### Streaming Subscriptions API
New actor event subscriptions should be registered using the following proto:

```proto
service Dapr {
    rpc SubscribeActorEventAlpha1(SubscribeActorEventRequestAlpha1) returns (google.protobuf.Empty) {}
}

// The message containing the details for subscribing an actor to a topic via streaming or otherwise
message SubscribeActorEventRequestAlpha1 {
    oneof subscribe_topic_events_request_type {
        SubscribeActorTopicEventsRequestInitialApha1 initial_request = 1;
        SubscribeActorTopicEventsRequestProcessedAlpah1 event_processed = 2;        
    }
}

// The initial message containing the details for subscribing an actor to a topic via streaming or otherwise
message SubscribeActorTopicEventsRequestInitialAlpha1 {
    // The name of the PubSub component the subscription applies to
    // Required. 
    string pubsub_name = 1;
    // The name of the topic being subscribed to.
    // Required.
    string topic_name = 2;
    // Registers the sink to deliver the messages to.
    // Required.    
    repeated oneof subscribe_actor_event_request_filter_type {
      // Registers a filtered subscription with a CEL expression that must evaluate as true
      SubscribeActorTopicEventsRequestFilterAlpha1
      // Registers a non-filtered subscription
      SubscribeActorTopicSinkAlpha1 
    } = 6;
}

// Describes the CEL expression and sink to pass an actor event to when the CEL evaluates as true
message SubscribeActorTopicEventsRequestFilterAlpha1 {
    // The CEL expression to evaluate
    // Required.
    string expression = 1;
    
    // The destination to send data to when the expression evaluates to true
    // Required.
    SubscribeActorTopicSinkAlpha1 sink = 2;
}

// Describes the actor endpoint to deliver the actor event to
message SubscribeActorTopicSinkAlpha1 {
    // The type of the actor to publish to.
    // Required.
    string actor_type = 1;
    
    // The endpoint of the actor to invoke with the published data
    // Required.
    string actor_method = 2;
    
    // The optional list of one or more actor identifiers subscribing to the event (optional).
    repeated string actor_id = 3; 
}

// The message containing the subscription to a topic
message SubscribeActorTopicEventsRequestProcessedAlpha1 {
    // The unique identifier of the subscription from the subscription manager
    string id = 1;
    
    // The status of the result of the subscription request
    TopicEventResponse status = 2;
}
```

### Event Message Routing
Upon the Daprd runtime receiving an actor PubSub message, the runtime will wrap the message with a CloudEvent envelope
as usual. The PubSub component used will be the one specified in the subscription (both component and queue/topic name).

If no matching component exists, an appropriately typed error will be returned to the client.

The Actor PubSub message CloudEvent envelope may look like the following example embodying JSON content:
```json
{
  "specversion": "1.0",
  "type": "io.dapr.event.sent",
  "source": "pubsub",
  "subject": "orders",
  "id": "5929aaac-a5e2-4ca1-859c-edfe73f11565",
  "time": "1970-01-01T00:00:00Z",
  "datacontenttype": "application/json",
  "data": {
    "message": "Hello, World!"
  }
}
```

Here, this diverges from the original proposal in that it specifies the PubSub component in the `source` field, but also
puts the name of the queue or topic in the `subject` field, compliant with the CloudEvent
[specification](https://github.com/cloudevents/spec/blob/v1.0.2/cloudevents/spec.md#subject) for this field.

Upon the daprd runtime receiving an actor PubSub message, the runtime will unwrap the CloudEvent envelope and evaluate
the message against the registered subscriptions in the subscription manager to filter which subscriptions are registered
to this PubSub source and subject.

At that point:
- Send the message to any actor sinks that specified a default route
- Evaluate each rule in each matching subscription and if true, send the message to the sink specified

The data payload will be serialized just as it is for normal subscriptions today.

#### Breaking changes from existing Dapr PubSub
When the message is successfully sent to all subscribers from daprd, the PubSub broker component should receive a
delivered response and Dapr resiliency policies should be relied on at that point to ensure that subscriptions receive
the message (or don't). This aligns with the
[delivery guarantees of Orleans](https://learn.microsoft.com/en-us/dotnet/orleans/implementation/messaging-delivery-guarantees).

#### Unknown implementation details
I am not familiar with how Dapr currently manages streaming subscriptions in a distributed fashion, but I would suggest
augmenting that to support this implementation. Ideally, this should be implemented so that setting up and removing
subscriptions at runtime is a lightweight operation as these subscriptions may be quite short-lived.

#### Final notes
- All filtering should occur on the runtime to avoid activating actors unnecessarily.
- If a registered sink activates an actor that hasn't previously been activated, it should activate it like any other inbound
  request would.
- PubSub invocations should follow the turn-based limitations inherent to Daor Actors already. As such, this may require
  the creation of an inbox of sorts for each actor to handle queued messages pending successful acknowledgement by the
  actor.

SDKs will need to be updated to support receiving Actor PubSub messages.
No changes to SDKs will need to be made to support _sending_ PubSub messages to actors or other endpoints.

## Feature Lifecycle Outline
- Add `SubscribeActorEventAlpha1` API
- Update SDKs to support `SubscribeActorEventAlpha1` API