# Scheduler Job Failure Policy

* Author(s): Josh van Leeuwen (@joshvanl)
* Updated: 2024-09-19

## Overview

Proposal details a Scheduler queue and Job API extension to support controlling behaviour of Job triggering in the event of failure.

## Background

The [Scheduler](https://docs.dapr.io/concepts/dapr-services/scheduler/) (and [go-etcd-cron](https://github.com/diagridio/go-etcd-cron/) library) are responsible for managing and executing jobs of all target consuming types.
When a Job is triggered, it is sent on a gRPC streaming connection from Scheduler to a connected daprd that implements that [Job target](https://github.com/dapr/dapr/blob/da6fb0db46b4d2932640eeaaaccf8b76f248f388/dapr/proto/scheduler/v1/scheduler.proto#L115).
In the event that this fails, for example if the trigger itself fails or there are no daprd instances connected for that target, the Job will currently still be marked as "triggered" a.k.a. "ticked" on the queue backend.
While always ticking failed jobs can be desirable behaviour, this is not always the case- and applications often require the job trigger to be retried multiple times on that tick to ensure durability of the schedule.

## Expectations and alternatives

1. The [go-etcd-cron](https://github.com/diagridio/go-etcd-cron/) library is to be updated to support a new `FailurePolicy` mechanism to correctly re-schedule jobs in the event of trigger failure.
2. The Scheduler Job API will mirror the new options available in the cron library.
  The scheduler will correctly signal to the library in the event of a trigger failure.
3. The runtime Jobs API will be updated to support the new `FailurePolicy` options.
4. The workflow Actor Reminders will be updated so that they will be marked as only to be triggered once, with an appropriate retry failure policy when using Scheduler.


## Design

To begin with, we will support 3 failure policies:
1. `Drop`: the job trigger will not be retried and the job will be marked as ticked.
  This is the current behaviour and will continue to be the default behaviour for all jobs.
2. `Constant`: the job will be retried as a constant time interval, up to a maximum number of retries (which could be infinite).
3. `Schedule`: the job will be retried according to a [cron scheduler](https://github.com/diagridio/go-etcd-cron/blob/2a1c6747974627691165eb96a2ca0202285d71eb/proto/job.proto#L68), up to a maximum number of retries (which could be infinite).

### Future Design

Although not part of the proposal, in future we can extend Scheduler to include a staging queue which is dedicated for Jobs where no current stream implements its target.
This addition would mean that the Jobs with no target are not needlessly being attempted to be triggered, freeing up main queue resources and preserving the intended failure policy.
Upon trigger, if no target stream is connected, the job will be moved to this staging queue.
In the event a stream implementing the target is connected, the Job can be moved to the main queue and triggered immediately or on its proper schedule.

### go-etcd-cron

To support the new `FailurePolicy` options, we first need to keep track of the current number of attempts the current Job count has been triggered on a particular tick.
We also need to keep track of the time at which that attempt was made in order to calculate the next attempt time according to the `FailurePolicy` schedule.
Like the trigger time, the last attempt time is the virtual _correct_ time that the attempt was to be made, rather than the observed wall clock time.
This is to ensure durability of the Scheduler during events of slow down or down time/restarts.
The attempts and last attempt time will added to the Counter proto message, and managed by the trigger queue manager.
Attempts will be 0 and last attempted time will be `nil` for all new tick counts of a Job.

```proto
// Counter holds counter information for a given job.
message Counter {
  ...

  // attempts is the number of times the job has been attempted to be
  // triggered at this count.
  uint32 attempts = 4;

  // last_attempted is the timestamp the job was last attempted to be triggered.
  optional google.protobuf.Timestamp last_attempted = 5;
}
```

Below are the proto definitions for the new `FailurePolicy` options.
Both constant and cron policies include an optional max retries option to limit the number of retries according to the number of attempts.
If max retries is unset, the Job will be retried indefinitely.
The failure policy message is added as an optional field to the Job message.
If unset, the failure policy of a Job is `Drop`.

```proto
// FailurePolicy defines the policy to apply when a job fails to trigger.
message FailurePolicy {
  // policy is the policy to apply when a job fails to trigger.
  oneof policy {
    FailurePolicyDrop drop = 1;
    FailurePolicyConstant constant = 2;
    FailurePolicyCron cron = 3;
  }
}

// FailurePolicyDrop is a policy which drops the job tick when the job fails to
// trigger.
message FailurePolicyDrop {}

// FailurePolicyRetry is a policy which retries the job at a consistent
// delay when the job fails to trigger.
message FailurePolicyConstant {
  // delay is the constant delay to wait before retrying the job.
  google.protobuf.Duration delay = 1;

  // max_retries is the optional maximum number of retries to attempt before
  // giving up.
  // If unset, the Job will be retried indefinitely.
  optional uint32 max_retries = 2;
}

// FailurePolicyCron is a policy which retries the job according to a cron
// schedule.
message FailurePolicyCron {
  // schedule is the cron schedule at which to retry the job.
  // See the Job.schedule field for the format of schedule.
  // Must not be empty.
  string schedule = 1;

  // max_retries is the optional maximum number of retries to attempt before
  // giving up.
  // If unset, the Job will be retried indefinitely.
  optional uint32 max_retries = 2;
}
```

```proto
message Job {
  ...

  // failure_policy is the optional policy to apply when a job fails to
  // trigger.
  // By default, the failure policy is drop- meaning the Job tick will be
  // dropped or ignored in the event of failure.
  optional FailurePolicy failure_policy = 7;
}
```

## Dapr

The Scheduler Job API service mirrors the new `FailurePolicy` options available in the go-etcd-cron library.
Similarly, the runtime Jobs API will be updated to support the same new `FailurePolicy` options.

When using Scheduler, Workflows will change Actor Reminders to now be single shot Jobs with a constant failure policy of every 5 seconds, and a maximum attempts of 120 (10 minutes).
Making this change to workflow reminders is desirable for a number of reasons:

- The current 1 minute interval of workflow reminders is often not appropriate, as it is either too long or too short.
- Current implementation of "cancelling" a workflow reminder is fragile and often does not work as expected.
- Removes the Delete Reminder code in the workflow runtime which is adding another round trip.
- Ensures there is never a case of "double trigger" of a workflow reminder, which is a suspected current source of flakiness in workflows.
