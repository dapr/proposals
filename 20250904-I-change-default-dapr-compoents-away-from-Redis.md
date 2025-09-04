# Title of proposal 

* Author(s): Oliver Tomlinson
* State: Proposal
* Updated: 2025/9/4

## Overview

Remove Redis as the default component when installed via `dapr init`, and replace with better alternative/s.

Redis has a growing number of disadvantages which make it no longer suitable as the component for the default experience (for PubSub/State/Actors/Workflows).

## Background

**With regards to PubSub**

Redis for PubSubs behaviour lies somewhere in a grey area between a standard message broker and a streaming broker, which often creates confusion when dapr adopters start experimenting with Dapr PubSub for the first time. 

In the case of long-running subscriber processes, there is an increasing likelihood that message timeouts will be exceeded which will lead to unpredictable retry behaviour etc. 

This unpredictable retry behaviour is exacerbated when Developers place debug breakpoints in their subscription handler code, which naturally causes the message timeout to be exeeded in the Redis component (which the developer is not aware of!), however the subscription handler code is still running.

My recommendation here would be to consider **kafka** as the default PubSub component that comes installed with dapr init for several reasons.

Kafka is currently the most widely adopted open-source message broker in the world (with RabbitMQ as second) therefore we immediately benefit from an increased likelihood that dapr adopters have some understanding of whats going on under the hood (Unlike Redis Streams, which doesn't have that much market penetration in comparison to Kafka).
 
Kafka also offers a much simpler-to-reason-about implementation than Redis. For example, Redis can get into a state where the same message is being processed by many subscriber handlers in parallel during a retry, which can't happen with Kafka. Redis streams has an explicit timeout, which, requires tuning for different workloads (this means having to declare many redis components with different configs, which is not amazing). Kafka does not have this, and as such Kafka is much more accepting than Redis Streams for accommodating different durations of workloads (Kafka is protected from the debug breakpoint problem mentioned above too) -- all this to say it gives a much better Developer Experience and first impression than Redis, which IMO is vital, as people will often misunderstand where things are going wrong and assume its dapr **creating** a problem, which is rarely the case**

**With regards to State Store**

Redis for State Store is convenient, but given we offer advice not to use Redis for State Store for Actors, Workfloads and transactional workloads in production, it doesn't feel like a great choice. Redis is first and foremost a cache, it's not a persistent store unless you go through the hoops of making it so. So Redis is not a perfect fit for the permanent data store that is the Dapr State Store component.

My recommendation would be Postgres, it goes without saying it's the most robust open source db, and has wide developer reach (even if you've never used postgres before, theres a chance you've used mySQL, MSSQL, Oracle, so you're already familiar with ANSI based SQL tech. However, With Redis, theres no real synergy with other tech, you're either familar with how it works, or not)

With the imminent licensing changes to Redis, there is no zero-cost production pathway for dapr adopters, and that feels like a missed opportunity.

Obviously it's not the responsibility of the Dapr project to **tell** dapr adopters what components to use in production, but, I do feel we should optimise for a zero-cost pathway for those that do just want to get something into production quickly. I'm talking about falling into the pit of success here. The quicker dapr adopters can get something into production, the more likely they are to reflect with positivity. If production roll-out becomes challenged or protracted with licensing and misleading/confusing developer experiences, then dapr adoption will naturally drop-off.

## Related Items

### Related proposals 

N/A

### Related issues 

N/A


## Expectations and alternatives

**What is in scope for this proposal?**

In scope for this proposal is the introduction of an new `dapr init` experience, which, does not use the Redis components, but instead uses alternatives that are open-source, with a permissive production license and offer a better developer experience OOTB (["falling into the pit of success"](https://english.stackexchange.com/questions/77535/what-does-falling-into-the-pit-of-success-mean))

In-order to phase the rollout of this new `dapr init` experience, I recommend we use a new flag e.g

> dapr init --install-os-components

The existing (Redis-based installation) would still be the default experience via `dapr init` until we are ready to make the swap.

We should consider swapping the default once the following are true :
 - Community feedback of the new experience is good (no recurring instances of the same problems)
 - All docs / quickstarts have been modified to support the the new default experience using the new open-source components
 - Any CI/CD that uses the Redis-based installation has been swapped to the new installation
 - The older Redis-based installation could still be triggered by providing a flag e.g. `dapr init --install-legacy-components`


**What is deliberately *not* in scope?**

N/A

**What alternatives have been considered, and why do they not solve the problem?**

Other open-source components can be considered and reasoned about during the discourse of this proposal. For example, some may feel that `RabbitMQ` and `MySQL` may offer a better default experience, over `Kafka` and `Postgres`.

**Are there any trade-offs being made? (space for time, for example)**

Yes, the trade-off here is that the new init mode is naturally more complex, as it must launch many components, not just one.

**What advantages / disadvantages does this proposal have?**

N/A

## Implementation Details

> [!NOTE]
> Implementation details to be agreed once the proposal is accepted
