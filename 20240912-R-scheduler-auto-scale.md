# Auto-Scale the Scheduler service

* Author(s): Cassie Coyle (@cicoyle)
* State: Review
* Updated: 2024-09-12

## Overview

This proposal outlines a design for auto-scaling the Schedulers, which is needed for the Scheduler service to be Stable.
The Scheduler's internal cron scheduler library code will need to be refactored to be dynamic. Runtime will need to be
updated to also account for dynamically connecting to Schedulers when they are scaled up or down. The Scheduler server
code will need to be updated such that it optionally runs embedded etcd on different Scheduler instances.


## Background

In Dapr v1.14 we introduced the Scheduler control plane service to support the Jobs API along with scaling the Actor
reminder system and the Workflow API (since it uses Actor reminders under the hood). The Scheduler encompasses running
itself as a new process, an embedded etcd instance, and an internal cron scheduler. There are a few things needed to
make the Scheduler a stable control plane service. This proposal will be detailing how to support the auto-scaling of
Schedulers and secondly enhancing the Schedulers such that we optionally run etcd instances in the Schedulers under
certain circumstances that make sense. When scaling beyond 5 etcd instances, there are little performance improvements, 
so it makes sense for Schedulers to become stateless at this point. For example, we could have 20 Schedulers, but only
5 of them would contain the embedded etcd.

## Related Items

### Related issues

[Tracking Issue: Scheduler Stable](https://github.com/dapr/dapr/issues/7796) &&
[Auto Scale Scheduler Issue](https://github.com/dapr/dapr/issues/7912).

## Expectations and alternatives

There will be 3 changes needed for this feature to be complete. 

1. **New control loop** for the [go-etcd-cron library](https://github.com/diagridio/go-etcd-cron), such that the leadership table is able to be dynamic. This 
means we need to watch the leadership table to observe a new Scheduler starting up or going down, stopping all Schedulers 
job triggers for that moment until the leadership table is updated and synchronized. We will need to stop the job queue 
and rebuild it again by reshuffling all the jobs to evenly split up the job ownership amongst Schedulers. Refactoring is
needed here such that the queue logic can be stopped and restarted. The leadership table will be updated to 
include the host address as the data such that the key in the table remains the id, but the value becomes the host address.

- An optimization we can do after the initial work is done, is to keep all jobs in memory. Having a local cache 
would optimize building the queue faster, since we are using etcd this should be feasible. 

2. **A new WatchHosts RPC** is added to `scheduler.proto`, allowing the dapr sidecar to connect to any available Scheduler 
instance via a ClusterIP service, rather than needing to manually track and connect to each instance as in the current 
headless service model. This `WatchHosts` RPC dynamically provides the current set of active Scheduler instances to the 
sidecar, keeping it updated as Schedulers scale up or down.

- For **backward compatibility**, we will check to determine whether the `WatchHosts` RPC is available or not. If it 
exists, Dapr will use it to manage Scheduler connections dynamically. If not, Dapr will continue using the existing 
static connection logic, maintaining compatibility with earlier versions by sending the full list back of available 
Schedulers.

```protobuf
service Scheduler {
  ...
  rpc WatchHosts(WatchHostsRequest) returns (stream WatchHostsResponse) {}
}

message HostPort {
  string host = 1;
  uint32 port = 2;
}

message WatchHostsResponse {
  repeated HostPort hosts = 1;
}

message WatchHostsRequest {}

```

3. **1/3/5 Schedulers with respective embedded etcd instances, beyond 5 instances run Schedulers stateless**. We want to
optionally run embedded etcd in Scheduler for cases where it makes sense. Cases to consider:
- running 1 Scheduler means running it with embedded etcd
- running 2 Schedulers means running the first with embedded etcd, the second Scheduler will be stateless and connect to
the existing etcd
- running 3 Schedulers means running all with embedded etcd
- running 4 Schedulers means running 3 Schedulers with embedded etcd, and the last one will be stateless and connect to
an existing etcd. Note, here there is no gain to round-robin-ing between the existing etcd instances due to the way 
etcd works with raft under the hood.
- running 5 Schedulers means running 5 Schedulers with embedded etcd.
- running beyond 5 Schedulers means running 5 Schedulers with embedded etcd, and any Scheduler beyond 5 will be 
stateless and connect to an existing etcd instance.

![Stateless Schedulers Example](./resources/20240912-R-scheduler-auto-scale/statelessSchedulersExample.png)

etcd must run with an odd number of replicas. In dapr, 5 will be our practical limit while running inside Schedulers. 
With the cron scheduler library and embedded etcd already separate processes in Scheduler, we just need to optionally 
run the embedded etcd.

Regarding [etcd's cluster reconfiguration operations guide](https://etcd.io/docs/v3.3/op-guide/runtime-configuration/#cluster-reconfiguration-operations),
for adding members, or etcd instances, we will need to set the `initial-cluster-state` to be existing for the new 
instances. We do not need to update the configuration of existing members manually, as once the new members join the 
cluster, it will automatically be integrated into the consensus and the cluster's state will include the new member. 

As noted in their documentation, we _**should**_ set the `strict-reconfig-check` flag on etcd such that we reject
reconfiguration requests if the number of started members will be less than a quorum. This flag will ensure we avoid
quorum loss as we dynamically add or remove stateful Schedulers.

## Completion Checklist

- [ ] Write the new control loop for the go-etcd-cron library to make the leadership table & job queue dynamic
- [ ] Add WatchHosts to scheduler.proto & update code to use it while ensuring backwards compatibility
- [ ] 1/3/5 Scheduler etcd problem where Schedulers might be stateless
- [ ] Tests
- [ ] Document that some Schedulers might be stateless