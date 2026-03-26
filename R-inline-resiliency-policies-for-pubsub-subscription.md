# Inline resiliency policies for pubsub subscription

* Author(s): Oliver Tomlinson (@olitomlinson)
* State: Draft
* Updated: 18/11/2023

## Overview

I propose that declarative and programmatic PubSub subscriptions should be _extended_ to include an _optional_ resiliency policy which is applied only to the inbound process of delivering the message  (sidecar to app subscription endpoint). The resiliency policy is scoped to just the subscription it was declared into / against.

This should only impact the runtime, but I could see potential work in the SDKs to support inline resiliency policies i.e. extending any convenience methods/attributes for registering subscriptions.

## Background

### Motivation
- There isn't currently an low-friction/lightweight solution to have a specialised resiliency policy on a per subscription basis, which means policies do not work well for some subscriptions which would benefit from their own policy.

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
  * Support for extending programmatic and declarative subscriptions.
* What is deliberately *not* in scope?
  * complete Resiliency policy overhauls /v2
* What alternatives have been considered, and why do they not solve the problem?
  * I think it would be possible to change the existing resiliency policy yaml struture to include named topics & subscriptions, but I do not think this provides the _best_ solution as it doesn't easily empower developers to craft their policies in the place in which they are needed (the subscriber code its self)
* Are there any trade-offs being made? (space for time, for example)
  * Operators may loose some visibility of specialised retry policies when programmatic subscriptions are used, but I think on balance this is trade-off is acceptable.
* What advantages / disadvantages does this proposal have?
  * Advantages
    * Reuses the existing resiliency concepts / yaml so this should feel really comfortable to users who are already familiar with resiliency policies
  * Disadvantages
    * Operators may not be able to _easily_ change resiliency policies which are attached to programmatic subscriptions (as that would be a code change)
      * Counter : Users who care about this should be encouraged to use the declarative model instead.


## Implementation Details

### Design

![image](https://github.com/olitomlinson/proposals/assets/4224880/f7d881d1-7511-430b-bf5a-5a5203b5b005)


#### Declarative Subscription 

```yaml
apiVersion: dapr.io/v2alpha1
kind: Subscription
metadata:
  name: order-subscription
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
            subscription:
              order-subscription:
                timeout: general
                retry: important
                circuitBreaker: pubsubCB
```

**Slim alternative** of the above. (personally, I prefer this slim version)

```yaml
apiVersion: dapr.io/v2alpha1
kind: Subscription
metadata:
  name: order-subscription
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

#### Programmatic Subscription

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
---
#### Potential implementation (on the surface this looks like it could work, but not validated!)

**As it stands...** 

when a Subscription is bound to a Topic in the runtime, the component inbound resiliency policy is loaded for the target pubsub component. https://github.com/dapr/dapr/blob/70f5fd6982f5e068f2e93614ebb6542dc84cb771/pkg/runtime/processor/pubsub/topics.go#L56

```go
policyDef := p.resiliency.ComponentInboundPolicy(name, resiliency.Pubsub)
```
At the point of the message being dispatched to the app channel, a policy runner is created and loaded with the resiliency policy.
https://github.com/dapr/dapr/blob/70f5fd6982f5e068f2e93614ebb6542dc84cb771/pkg/runtime/processor/pubsub/topics.go#L179C3-L179C3
```go
policyRunner := resiliency.NewRunner[any](ctx, policyDef)
		_, err = policyRunner(func(ctx context.Context) (any, error) {
			var pErr error
			if p.isHTTP {
				pErr = p.publishMessageHTTP(ctx, sm)
```

**I propose the above is modifed...** 

Such that the _new_ subscription policy (if one has been supplied) is **swapped** in place of the component inbound resiliency policy
```go
policyDef := p.resiliency.ComponentInboundPolicy(name, resiliency.Pubsub)
subscriptionPolicyDef, ok := p.resiliency.SubscriptionResiliencyPolicy(route.Resiliency)

if ok {
    policyDef = subscriptionPolicyDef
}
```

---

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

* By allowing specialised resiliency policies per subscription across all topics and apps.

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


