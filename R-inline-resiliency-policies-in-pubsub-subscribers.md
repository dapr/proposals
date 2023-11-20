# Inline resiliency policies in pubsub subscriber declarations

* Author(s): Oliver Tomlinson (@olitomlinson)
* State: Draft
* Updated: 18/11/2023

## Overview

I propose that declarative and programmatic PubSub subscriptions should be extended to include an optional resiliency policy which is applied only to the inbound process of delivering the message  (sidecar to app subscription endpoint). The resiliency policy is scoped to just the subscription it was declared into / against.

This should only impact the runtime, but I could see potential work in the SDKs to support inline resiliency policies i.e. extending any convenience methods/attributes for registering subscribers.

## Background

### Motivation
- There isn't currently an low-friction/lightweight solution to have a specialised resiliency policy on a per subscriber basis, which means policies do not work well for some subscribers which would benefit from their own policy.

### Goals
- Empower Developers (and Operators) to provide resiliency policies which are specialised to the needs of the message handler.

### Current Shortfalls
- See motivation
  
## Related Items

### Related proposals 

- A general proposal to introduce resiliency policies via a Callback API https://github.com/dapr/dapr/issues/5264

### Related issues 

- See [here](https://github.com/dapr/dapr/issues/7184) for the problem that this proposal is aiming to address

## Expectations and alternatives

* What is in scope for this proposal?
  * Support for programmatic and declarative subscribers
* What is deliberately *not* in scope?
  * ???
* What alternatives have been considered, and why do they not solve the problem?
  * I think it would be possible to change the existing resiliency policy yaml struture to include named topics & subscribers, but I do not think this provides the _best_ solution as it doesn't easily empower developers to craft their policies in the place in which they are needed (the subscriber code its self)
* Are there any trade-offs being made? (space for time, for example)
  * Operators may loose some visibility of specialised retry policies when programmatic subscribers are used, but I think on balance this is trade-off is acceptable.
* What advantages / disadvantages does this proposal have?
  * Advantages
    * Reuses the existing resiliency concepts / yaml so this should feel really comfortable to users who are already familiar with resiliency policies
  * Disadvantages
    * Operators may not be able to _easily_ change resiliency policies which are attached to programmatic subscribers (as that would be a code change)
      * Counter : Users who care about this should be encouraged to use the declarative model instead.


## Implementation Details

### Design

#### Declarative Subscriber 

```yaml
apiVersion: dapr.io/v2alpha1
kind: Subscription
metadata:
  name: order-subscriber
spec:
  pubsubname: myPubsubComponent
  topic: newOrder
  routes:
    default: /order
  deadLetterTopic: poisonMessages
  resiliency:
    policies:
      timeouts:
        general: 5s
      retries:
        important:
          policy: constant
          duration: 5s
          maxRetries: 30
      circuitBreakers:
        pubsubCB:
          maxRequests: 1
          interval: 8s
          timeout: 45s
          trip: consecutiveFailures > 8
    components:
      myPubsubComponent:
        topic:
          newOrder:
            subscriber:
              order-subscriber:
                timeout: general
                retry: important
                circuitBreaker: pubsubCB
```

**Slim alternative** of the above. (personally, I prefer this slim version)

```yaml
apiVersion: dapr.io/v2alpha1
kind: Subscription
metadata:
  name: order-subscriber
spec:
  pubsubname: myPubsubComponent
  topic: newOrder
  routes:
    default: /order
  deadLetterTopic: poisonMessages
  resiliency:
    timeout: 5s
    retry: 
      policy: constant
      duration: 5s
      maxRetries: 30
    circuitBreaker:
      maxRequests: 1
      interval: 8s
      timeout: 45s
      trip: consecutiveFailures > 8
```

#### Programmatic Subscriber

continuing with the **slim** theme, this is how the `/dapr/subscribe` response could look...

```json
[
    {
        "pubsubname": "myPubsubComponent",
        "topic": "newOrder",
        "route": "/orders",
        "deadLetterTopic": "poisonMessages",
        "resiliency": {
            "timeout": "5s",
            "retry": {
                "policy": "constant",
                "duration": "5s",
                "maxRetries": "30"
            },
            "circuitBreaker": {
                "maxRequests": "1",
                "interval": "8s",
                "timeout": "45s",
                "trip": "consecutiveFailures > 8"
            }
        }
    }
]
```


### Feature lifecycle outline

* Expectations
  * ???
* Compatability guarantees
  * N/A
* Deprecation / co-existence with existing functionality
  * This would have to co-exist with resiliency policies that are already applied to pubsub components. I would expect that the specialised policies would **override** any component-level policies.
* Feature flags
  * No

### Acceptance Criteria

How will success be measured? 

* By allowing specialised resiliency policies per subscriber across all topics and apps.

## Completion Checklist

What changes or actions are required to make this proposal complete? Some examples:

* Code changes
  * N/A
* Tests added (e2e, unit)
  * N/A
* SDK changes (if needed)
  * TBD
* Documentation
  * N/A


