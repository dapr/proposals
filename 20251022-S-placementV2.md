# Consolidate Placement into Scheduler, Improve Actors and Workflows Reliability and Performance

* Author(s): Cassie Coyle (cicoyle)
* State: Proposed
* Introduced: 10-31-2025
* Binding votes: dapr/dapr maintainers

## Overview

This proposal introduces "PlacementV2", a redesign that eliminates the standalone Placement service by moving its functionality into the Scheduler control plane service.

The motivation is to increase reliability and performance, reduce operational overhead, and simplify the control plane by unifying two stateful, 
leader-elected services (Placement and Scheduler) into one. The Scheduler already runs with embedded etcd and provides a job ownership table, 
consensus, and streaming APIs — all of which can serve actor placement without maintaining an additional custom Raft implementation.

This consolidation allows us to:
- Remove one control-plane StatefulSet and custom Raft subsystem.
- Reduce gRPC connections from Dapr sidecars (one fewer control-plane stream).
- Improve stability by removing the hard coded raft implementation for a simplified leadership algorithm.
- Improve latency and reliability for both Actors and Workflows (and Reminders) by colocating coordination in one process.
- Remove global locking and instead have per actor per namespace type locking.
- Remove resource waste with unnecessary heartbeats.

This will be feature gated via an opt-in (new) feature flag: `SchedulerPlacement`.

## Background

### Why Placement exists

The [Placement service](https://docs.dapr.io/concepts/dapr-services/placement/) coordinates [Dapr Actors](https://docs.dapr.io/reference/api/actors_api/), 
ensuring that there is exactly one active instance of each actor ID across all dapr sidecars. It maintains cluster membership, 
computes hash tables mapping actor IDs to sidecars, and pushes versioned tables to sidecars so they can route calls correctly.

### How Placement works today

1. **Membership:** Each sidecar connects to the Placement leader via a long-lived gRPC stream (`ReportDaprStatus`), sending its `appID`, `namespace`, `actor types`, and periodic heartbeats.
2. **Table Computation:** The leader builds a hash table per `(namespace, actorType)`. Whereby `host = fn(actorType, actorID, tableVersion)`.
3. **Locking:** The leader locks while rebuilding tables to ensure atomic updates (effectively a namespace-wide “stop-the-world” window where it `locks`, `updates`, `unlocks`).
4. **Dissemination:** The leader pushes versioned placement tables to sidecars, which swap their local routing tables upon receipt.
5. **Leader Election:** Placement replicas elect a leader via HashiCorp Raft. The leader maintains all in-memory membership and placement state.
6. **Persisted:** `membership`, `apiLevel`, `tableGeneration`, and global `config` are persisted in Raft log/snapshots.
7. **Placement rings themselves are _in-memory only_ on the leader** and are rebuilt from membership on every leader election. There is no durable ring storage — this is intentional to keep the critical path fast.

Sidecars:
- Cache the latest versioned table
- Compute ownership locally via hashing
- Activate actors on-demand if they own the ID

## Motivation

### Problems with the current design

- **Two control plane StatefulSets** (Placement + Scheduler) with similar logic running that could be combined.
- **Duplicate raft implementations** (custom built Raft in Placement, embedded etcd in Scheduler).
- **Two separate gRPC connections per sidecar** — one for Scheduler (jobs/reminders), one for Placement (actors).
- **Global namespace locks** during table rebuilds.
- **Difficult to reproduce bugs** from cluster events, connection issues, network partitions, or leader churn in Placement.
- **Wasteful Heartbeats** with per second heart beat calls per sidecar

### Proposed Benefits

By merging Placement into Scheduler we get:

- **Reliability**: One etcd-based Raft implementation instead of custom Raft implementation, simplify leadership election
- **Latency**: Sidecars maintain only one gRPC stream (to Scheduler) instead of two, reduce hops with Actor/Workflow/Reminder usage
- **Operational Overhead**: One fewer StatefulSet, less config and logs, fewer moving parts
- **Performance**: Per actor type updates and locking, no global locks
- **Maintainability**: Remove lines of legacy leadership and custom Raft code

## Related Items

### Related issues

There have been a ton of issues in dapr/dapr over the years regarding issues with Placement and Placement's leadership algorithm, grpc
connections, placement client connection errors cause issues with scheduler connections, custom raft logic, deadlocks that are 
difficult for maintainers to reproduce and nearly impossible to pin down and fix as a result of the lack of ability to reproduce.

## Expectations and alternatives

* What is in scope for this proposal?
  * Full functional parity with current Placement service with feature flag, but with improved performance
  * Per type dissemination instead of global locking
* What will not change / what is out of scope for this proposal?
  * Actor runtime semantics: activation/state/timers/etc
  * Client-side hashing for actor routing (remains in the sidecar)
  * Placement tables (remain in-memory) -> as per‑type rings (in‑memory), this is _still_ **not** persisted
  * Actor feature requests like actor pubsub, cross namespace actor invocation, etc
    * Hard stickiness can be done in the future via an rpc on the Scheduler, but is outside the scope of this proposal
* What alternatives have been considered, and why do they not solve the problem?
  * Keep Placement and Scheduler separate and fix existing Placement issues: 
    * Hand rolled raft, means 2 leaderships to manage
    * This would maybe solve some of the Placement issues, but lack the performance improvements gained by combining the control plane services into a single process. 
      This would also entail spending countless hours trying to reproduce very difficult to reproduce issues.
  * Gossip/peer streams among Schedulers: 
    * We don't really need this. Scheduler 'leader' will have the table and others will be notified immediately when the leader goes down to elect the new leader and rebuild the table.
  * Kubernetes leases for leadership & simple KV store for persisted state: 
    * By choosing the first element in the job 'ownership' scheduler table, we don't need a more complicated leadership election process.
    * We don't need to persist anything. Everything can be derived from the streaming connections.
  * Utilize etcd raft leadership under the hood to determine Scheduler leadership that is equivalent to the placement leader
    * Want to decouple this from etcd

## Implementation Details

### Design

#### Technical Comparison: What exists today?

##### Placement

- Leader-elected (HashiCorp Raft).
- Sidecars open one bidirectional stream to the leader via `ReportDaprStatus`.
- Sidecars send Host heartbeats; leader applies membership to an FSM (persisted via Raft snapshots).
- Leader computes rings per namespace/actorType in memory.
- Dissemination: lock → update(full PlacementTables snapshot, version=generation, api_level) → unlock.
- Followers replicate state.
- Full table push on every change.

![PlacementDiagram](./resources/20251022-S-placementV2/placementDiagram.png)

##### Scheduler

- All are peers, runs embedded etcd which runs the raft consensus algorithm and leadership under the hood for us.
- There is a job ownership/"leadership" table to keep track of which scheduler owns which jobs for triggering.
- Streams: `WatchHosts` (server-stream), `WatchJobs` (bidirectional for triggers).
- CRUD for jobs; no actor placement.
- Not placement aware.

![SchedulerDiagram](./resources/20251022-S-placementV2/schedulerDiagram.png)

##### Proposed Architectural Solution

###### Remove Placement + Implement Per-Type Dissemination & Locking

Goal: Remove Placement service + eliminate namespace-wide batching and global activation pauses

We should tweak the `WatchHosts` Scheduler rpc to include a simple flag: `leader=true/false`, allowing sidecars to reuse the same connection to the leader for the `ReportDaprStatus` rpc.
The existing Scheduler partition ownership table in etcd can serve to determine a 'scheduler leader' analogous/equivalent to the placement leader. 

Although all Schedulers are peers without a true leader (all receive and trigger jobs), sidecars connect to all Schedulers since job ownership is distributed and we don't know which scheduler
owns which jobs. Using the `leader=true/false` flag from `WatchHosts`, sidecars can identify which connection to use for `ReportDaprStatus`.

```protobuf
service Scheduler {
  ...
  // WatchHosts is used by the daprd sidecar to retrieve the list of active
  // scheduler hosts so that it can connect to each. Receives an updated list
  // on leadership changes.
  rpc WatchHosts(WatchHostsRequest) returns (stream WatchHostsResponse) {}
}

// WatchHostsRequest is the message used by the daprd sidecar to connect to the
// Scheduler and watch for hosts.
message WatchHostsRequest {}

// WatchHostsResponse is the response message to convey the details of a host.
message WatchHostsResponse {
  // hosts is the list of current active scheduler hosts in the cluster.
  repeated Host hosts = 1;
}

// Host is the message used to represent a host.
message Host {
  // address is the address of the host.
  string address = 1;
  bool leader = 2; // true for the single elected leader
}
```

To avoid coupling leadership to etcd, we can simply treat the first Scheduler instance in the ownership table (once quorum is reached) as the leader.

![PartitionOwnershipTable](./resources/20251022-S-placementV2/partitionOwnershipTable.png)

While placement persists some data, like membership, with scheduler we should not persist data, nor do we need a migration path, as we will keep all data in memory and everything can be
determined dynamically by the streaming connections. We should remove the `api_level` and `table_generation`, the `raft` pkg, extra unused code like load and bounded loads algorithm 
functions that are not used/called, and the heartbeat logic and base this off stream connections. We can rebuild the placement table entirely in-memory and not store anything since we 
have a streaming connection of all sidecars to all schedulers.

The current Placement service exposes a [State API](https://docs.dapr.io/reference/api/placement_api/) for debugging, but it is disabled by default.  
With added metrics (detailed near the end of the proposal), we will have full observability into membership changes, per‑type ring versions, and ring rebuilds.  
Therefore, we will **not** expose the Placement State API from the Scheduler — reducing complexity without sacrificing debuggability.

If the leader or all scheduler instances go down, the system can simply start over at per‑type ring version `0`. Because everything is derived dynamically from active streaming connections, the
placement table can be rebuilt immediately without data migration or persistence.

Rebuild and disseminate only the actor types whose eligible hosts changed, with per‑type lock/update/unlock. Other types continue serving traffic uninterrupted.

Some definitions:
- Actor Types: the actor types a sidecar hosts. They come from the app/SDK registering actors; daprd’s actor subsystem exposes this, and daprd reports it.
- Ring: the per actor type consistent hashing structure that maps (actorType, actorID) → host, aka the “consistent hash”.
- PlacementTable: a namespace snapshot containing a map of actorType → ring plus metadata. In Phase 2 we send partial updates (only the changed types).

**Previously**, Placement would perform a namespace‑scoped full‑table swap, like the following example explains:

```text
Event: dapr-2 disconnects (it hosted T1 and T2)
1) Placement Leader applies MemberRemove(dapr-2) to its state/FSM (rings are per type internally).
2) Placement Leader disseminates namespace‑wide:
   Lock(ns) → dapr-0, dapr-1
   Update(full snapshot: all types) → dapr-0, dapr-1
   Unlock(ns) → dapr-0, dapr-1
   Sidecars: clear all entries on Lock; rebuild all rings on Update; resume on Unlock.
Implications:
   - Brief global pause for all actor types in the namespace.
   - Sends full table even if one type changed.
```

**Proposed solution:** type-scoped partial update:

```text
Event: dapr-2 disconnects (it hosted only T2)
1) Scheduler notes Actor_Types(dapr-2) included T2 → EligibleHosts[T2] changed.
2) NamespaceController:
   Routes events to ActorTypeController, for namespaced operations
3) ActorTypeController[T2]:
   Recompute Ring[T2], bump Version[T2].
   Disseminate to dapr-0, dapr-1:
   Lock(types=[T2])
   Update(Entries={T2}, Versions={T2})
   Unlock(types=[T2])
Sidecars: pause lookups only for T2, merge T2’s ring, resume T2. T1 unaffected.
```

T1, T2 come from actor types registered in the SDK via [RegisterActorImplFactoryContext](https://github.com/cicoyle/test-apps/blob/main/scheduler-actor-reminders/server/player-actor.go#L35).
`daprd` detects them using [actorTable.SubscribeToTypeUpdates(...)](https://github.com/dapr/dapr/blob/master/pkg/actors/table/table.go#L216) and reports them as `actor_types` in `ReportDaprStatus`.

**High Level Controllers for how this will work:**
Controllers will live in Scheduler, essentially logical processes or go routines for example:
- NamespaceController
  - Scopes state and streams by namespace
- ConnectionController
  - Watches sidecar streams, tracks actor_types per sidecar, emits SidecarAdded/Removed/Updated events by namespace.
- ActorTypeController[T]
  - On events that affect T: recompute Ring[T], increment Version[T], disseminate type‑scoped Lock/Update/Unlock.
  - Retries per target, does not block other types.

![PerTypeDissemination](./resources/20251022-S-placementV2/perTypeDissemination.png)

What this means we need:
- Type‑scoped locking: implement a lock keyed by actor type. On Lock(types): block lookups only for those types, others run unaffected.
- Partial merge: on Update with per type tables.Entries, rebuild/replace hashTable.Entries[T] only for provided T, do not clear the whole map.
- Targeted drain: after updating types [T...], drain only local actors of those types whose new owner is remote. Do not call a global “halt all”.
- Per‑type versioning/readiness:
  - Track versionByType[T]. No need for a global version.
  - During initial dapr sidecar startup, the sidecar shouldn’t accept actor calls until it has a ring for every actor type it locally reports (its Actor_types). 
  - After that, subsequent changes are per-type: only the affected types must pause, update, and resume.

Sidecar Startup vs steady-state
- On new stream: Scheduler sends LOCK(all) → UPDATE(full snapshot: all types, versions per type) → UNLOCK(all). 
- On changes to type T: send LOCK([T]) → UPDATE(entries={T}, versions={T}) → UNLOCK([T]).
- On new type: same as change, send only that type.
- On cold reconnect: treat as startup again (full once, then deltas).

HaltNonHosted vs HaltHosted
- HaltNonHosted([T…]): for each local actor with type in [T…], recompute owner with updated ring; if new owner is remote, deactivate/migrate it. 
If still local, leave it running.
- Do not implement HaltHosted, stopping actors that remain local causes avoidable churn. Meaning, only stop actors that the ring says moved away.
  If an actor’s ownership didn’t change, let it continue running - essentially soft stickiness.
- No global drain.

This enforces soft stickiness: per‑type updates do not stop actors that still map to the local sidecar, which avoids 
unnecessary churn and short global pauses.
So, during LOCK([T]) → UPDATE([T]) → UNLOCK([T]), only actors of types [T] that moved to a remote owner are drained and 
all others continue running. No namespace‑wide drain.

`ReplicationFactor` aka, virtual nodes (vnodes) remain:
Number of virtual positions per host used to build the ring. This helps to smooth key distribution and reduce movement when hosts join/leave.
To support this, Scheduler will need to expose the equivalent flag:
```go
fs.IntVar(&opts.ReplicationFactor, "replicationFactor", defaultReplicationFactor, "sets the replication factor for actor distribution on virtual nodes")
```

Improvements:
- No global pause: only the types in flight are briefly locked, others continue (soft stickiness).
- Less network: payloads contain only the changed types.
- Less CPU: only rebuild rings for changed types.
- Better concurrency: different types can disseminate in parallel.

Protos (slightly tweaked from original from placement), change `ReportDaprStatus` -> `ReportActorTypes`
```protobuf
...
    // sidecar reports its presence and actor_types;
    // (now) scheduler sends lock/update/unlock orders
    rpc ReportActorTypes(stream Host) returns (stream PlacementOrder) {}
...

message Host {
  string name = 1;
  int64 port = 2;
  string appID = 3;
  string namespace = 4;
  repeated string actor_types = 5; // actor types hosted
}

message PlacementOrder {
  enum Operation { LOCK = 1; UPDATE = 2; UNLOCK = 3; }
  
  Operation operation = 1;
  string namespace = 2;

  // Required for lock/unlock
  // empty for all types upon startup
  // this is not needed for update since its derived from tables
  repeated string actor_types = 3; // scope

  // Per-type versions, changed on update
  map<string, uint64> versions = 4; // for UPDATE

  // Partial tables allowed, only changed types.
  PlacementTables tables = 5; // for UPDATE
}

message PlacementTables {
  map<string, PlacementTable> entries = 1; // key: actor type
  // vnodes per host, for building rings (global default)
  int64 replication_factor = 2;
}

// One actor-type ring
message PlacementTable {
  map<string, TableHost> hosts = 1; // host key -> below details
}

message TableHost {
  string name = 1;
  int64  port = 2;
  string appID = 3;
}
```

Pseudo example for reference:

Sidecar starts up and connects to scheduler
```json
{ "operation":"LOCK", "namespace":"ns", "actor_types":[] }
{ "operation":"UPDATE", "namespace":"ns",
  "actor_types":[],
  "versions":{"T1":1,"T2":1},
  "tables":{
    "replication_factor":64,
    "entries":{
      "T1":{"hosts":{"10.0.0.1:3500":{"name":"10.0.0.1:3500","port":3500,"appID":"app"}}},
      "T2":{"hosts":{"10.0.0.3:3500":{"name":"10.0.0.3:3500","port":3500,"appID":"app"}}}
    }
  },
}
{ "operation":"UNLOCK", "namespace":"ns", "actor_types":[] }
```

Delta (T2) (ring change)
```json
{ "operation":"LOCK", "namespace":"ns", "actor_types":["T2"] }
{ "operation":"UPDATE", "namespace":"ns",
  "actor_types":["T2"],
  "versions":{"T2":7},
  "tables":{
    "replication_factor":64,
    "entries":{
      "T2":{
        "hosts":{
          "10.0.0.1:3500":{"name":"10.0.0.1:3500","port":3500,"appID":"app"},
          "10.0.0.3:3500":{"name":"10.0.0.3:3500","port":3500,"appID":"app"}
        },
      }
    }
  },
}
{ "operation":"UNLOCK", "namespace":"ns", "actor_types":["T2"] }
```

### Feature lifecycle outline

* Compatibility guarantees
  * No need for placement -> scheduler migration
  * No regressions
* Deprecation
  * Placement Service should be deprecated after the feature flag process, false at first, then it becomes default, after 2 releases deprecate
* Feature flags
  * `SchedulerPlacement`

### Acceptance Criteria

How will success be measured?

* Performance targets
  * Performance improvements when combined into a single process, to be validated with perf tests
  * Fewer pauses/faster dissemination
* Compatibility requirements
  *   No regressions
* Suggested Metrics to Add/Have
  * more should be added in general, things like the following would be useful:
    * scheduler metrics
      * ring version by namespace, actor type
      * ring rebuilds total by namespace, actor type, reason
      * ring rebuild duration by namespace, actor type
      * dissemination total count by namespace, actor type, operation
      * dissemination latency by namespace, actor type
      * in-flight locks by namespace, actor type
    * dapr sidecar metrics
      * actor ring version by namespace, actor type
      * update apply duration by actor type
      * lock duration by actor type
      * halt non-hosted updates total count by actor type
      * halt non-hosted actors per update by actor type
      * actor activations total count by actor type
      * actor deactivations total count with reason by actor type
      * active actors by actor type
      * actor routing visibility: local/remote by actor type
      * dissemination total count by type of operation: lock/update/unlock by actor type
      * update acked total by namespace, actor type
      * actor activation latency by actor type
      * number of actors drained per update with reason by namespace, actor type
      * numbers of actors drained total by namespace, actor type

## Completion Checklist

* Above proposed code changes
* Tests added
* Documentation