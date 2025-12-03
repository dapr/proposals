# Scheduler Broadcast Job Type

* Author(s): Josh van Leeuwen (@joshvanl)
* Updated: 2025-04-29

## Overview

This proposal describes adding a new Scheduler Job type - `Broadcast`.
This job type is designed to leverage the durability of Scheduler and its reliable connection to all daprd runtimes to broadcast objects to all relevant clients to allow for durable message broadcasting.
The initial design focuses on two initial use cases, `actor pubsub` and `durable actor IDs`, but this job type's functionality can be extended to other domains in future.

## Related Items

https://github.com/dapr/proposals/pull/78

## Background

It is the case in Dapr that some features desire or require the need to be able to durably broadcast messages to some set of runtimes in the cluster.
Such examples include the need to broadcast [Actor pubsub subscription](https://github.com/dapr/proposals/pull/78) manifests to all actor type implementations, or signal that an actor ID should be invoked, and life cycle extend beyond the typical actor deactivation timeout or dissemination events.
It is imperative that these objects/messages are preserved across restarts/dissemination/crash events to ensure their durability.
It is also the case that these messages need a reliable way to be broadcast to all the relevant hosts.

While these two feature do have access to reliable durable storage in the form of the Actor State Store, implementing these require durable service discovery and co-ordination _at the runtime level_.
Not to mention that storage is often not available outside of the context of Actors.
Implementing these at the runtime level would be brittle as the co-ordination mechanism itself would require significant machinery.
The Scheduler service already has application typed context service discovery and connectivity with durable storage access, so is the perfect place to implement this functionality.

Users today are required to use "normal" Scheduler jobs to ensure message durable execution, and implement their own broadcast or message routing functionality.
Typically creating a job which has an infinite schedule that triggers at a fixed small interval.
Moving this functionality as a first concept to Dapr improves performance and improves the developer experience in creating complex distributed systems.

### Design

The Scheduler service will implement a new `Broadcast` typed job.
This new Job type is an extension on the existing "typical" `Job` type, with the difference being that instead of the Job being invoked at some schedule to a _single_ client, the Job itself will be Broadcast to _all_ clients which implement that type.

The Scheduler Job handler will be updated to special case these Broadcast job types to be extended with the following functionality:
1. Instead of Scheduler sending the Job trigger event to a single client, it will be sent to all clients which implement that job type.
2. Upon connection of a client which implements that type, Scheduler will send the job to each implementing client with the `BROADCAST_JOB_EVENT_PUT` flag set.
3. For each Job trigger event, each client will receive the event with the `BROADCAST_JOB_EVENT_TRIGGER` flag set.
4. At any point during the Job's life cycle it is deleted (not expired according to its schedule), the Scheduler will send a `BROADCAST_JOB_EVENT_DELETE` event to all clients which implement that job type.

## Implementation Details

#### go-etcd-cron

In order for Scheduler to track the existing Broadcast jobs which are created, exist during a restart, or are deleted, a new hook will be implemented in the cron Informer.
This hook will be called during informer events, and the Scheduler will use these events to manage the in memory Broadcast job state and manage broadcasting those events.

#### Scheduler

The Scheduler service will be updated to include a new Job handler code path to build a Broadcast Job manager.
This manager will take informer events from go-etcd-cron, sending PUT and DELETE flagged job events to connected clients during CRUD events, as well as when appropriate clients connect.
The Scheduler will trigger all appropriate connected clients concurrently during trigger events.
It is not expected that clients would return any response code to a trigger execution other than SUCCESS, as these jobs are generally intended (at least for the initial use cases) to provide object manifests- not triggering some execution.
A client failing to return the response of a trigger should be considered as a failed connection.
In any case, to account for future use cases, the trigger response codes should considered a logical AND whereby failed calls will be retried on each failed client call according to the Jobs Failure Policy.

Broadcast jobs can either expire according to its schedule, or be deleted by a client.

Broadcast Jobs will continue to adhere to the staging queue, like existing Jobs today.

#### Keep Alive

The gRPC keep alive options for both the Scheduler server and daprd clients need to be updated so that they are more aggressive.
This ensures greater durability of the gRPC job streams for Scheduler broadcast job manager.
https://pkg.go.dev/google.golang.org/grpc/keepalive

#### protos

The following protos will be added to the Scheduler service to implement the Broadcast job type.
Note that both the Actor Subscription and Durable Actor ID feature use cases are included as example of API shape.

```proto
// JobTargetType is the type of the job target.
enum JobTargetType {
  // JOB_TARGET_TYPE_JOB is the job target type.
  JOB_TARGET_TYPE_JOB = 0;

  // JOB_TARGET_TYPE_ACTOR_REMINDER is the actor reminder target type.
  JOB_TARGET_TYPE_ACTOR_REMINDER = 1;

  // JOB_TARGET_TYPE_ACTOR_SUBSCRIPTION is the broadcast actor subscription
  // object target type.
  JOB_TARGET_TYPE_ACTOR_SUBSCRIPTION = 2;

  // JOB_TARGET_TYPE_DURABLE_ACTOR_ID is the broadcast durable actor ID target type.
  JOB_TARGET_TYPE_DURBALE_ACTOR_ID = 3;
}

// JobTargetMetadata holds the typed metadata associated with the job for
// different origins.
message JobTargetMetadata {
  oneof type {
    TargetJob job = 1;
    TargetActorReminder actor_reminder = 2;
    TargetBroadcast broadcast = 3;
  }
}
```

```proto
syntax = "proto3";

package dapr.proto.scheduler.v1;

import "google/protobuf/any.proto";

option go_package = "github.com/dapr/dapr/pkg/proto/scheduler/v1;scheduler";

// BroadcastJobEventType is the type of the event why this job is being
// broadcast.
enum BroadcastJobEventType {
  // BROADCAST_JOB_EVENT_PUT is the event type for a job being created or
  // updated.
  BROADCAST_JOB_EVENT_PUT = 0;

  // BROADCAST_JOB_EVENT_DELETE is the event type for a job being deleted.
  BROADCAST_JOB_EVENT_DELETE = 1;

  // BROADCAST_JOB_EVENT_TRIGGER is the event type for a job being triggered.
  BROADCAST_JOB_EVENT_TRIGGER = 2;
}

// BroadcastJobDataWrapper is a canonical wrapper for the data of a broadcast
// job which is sent to clients from the Scheduler. Contains CRUD events for
// the broadcast job.
// This message will set on the data field of the job when being broadcast from
// Scheduler to compatible clients.
message BroadcastJobDataWrapper {
  // type is the type of the event which is causing this job to be send to the
  // client.
  BroadcastJobEventType type = 1;

  // data contains the actual data of the job. It may be wrapped in another
  // message, depending on the type of broadcast job.
  google.protobuf.Any data = 2;
}

// TargetBroadcast are job types which are "broadcast", meaning they are sent
// to all clients who support this target on connection, and when the job is
// created. On delete, the job is signalled to all matching clients that the
// job has been deleted with an empty data payload.
message TargetBroadcast {
  oneof broadcast {
    // TargetActorSubscription is the message which holds broadcast component spec
    // for an Actor Subscription.
    TargetActorSubscription actor_subscription = 1;

    // TargetDurableActorID is the message which holds broadcast durable actor
    // ID for an Actor Subscription.
    // The data field of the job must be of type `common.InvokeRequest`, where
    // the `.HttpExtension.Verb` will always be ignored, in favour of a POST
    // when the durable actor is invoked.
    TargetDurableActorID durable_actor_id = 2;
  }
}

// TargetActorSubscription is the message which holds broadcast component spec
// for an Actor Subscription.
message TargetActorSubscription {
  // id is the actor ID.
  string id = 1;

  // type is the actor type.
  string type = 2;
}

// TargetDurableActorID is the message which holds broadcast durable actor ID
// jobs.
message TargetDurableActorID {
  // id is the actor ID.
  string id = 1;

  // type is the actor type.
  string type = 2;
}
```

## Use case examples

### Actor PubSub Subscription

As detailed in the [Proposal](https://github.com/dapr/proposals/pull/78), the Actor PubSub subscription object which is manifest which details how a subscription should be defined for receiving messages to a single actor ID.
In order for this manifest to be durable, it needs to be broadcast to all clients which implement that Actor type.
Upon receiving the object, or during dissemination, each actor runtime can determine which set of subscription manifests are assigned to that host based on an Actor ID placement table lookup.
The object subscription object itself should have no definitive lifetime, and is only created and deleted by the application.
It could be the case that these objects also have a corresponding TTL associated.

The actor runtime will receive events about the job;
- on put events the actor runtime will create the underlying subscription, or replace one that is already existing.
- on delete events the actor runtime will delete the subscription.
- there should not be any trigger events.

The Actor Subscription broadcast job will take the following form.
Notice the schedule is set to infinite as the Job should never be triggered.
The name of the Job object can be any application defined name.
This job type will return an `already exists` error if a PUT is attempted on an existing Actor Subscription job.

Schedule:
```json
{
  "name": "my-actor-subscription", // application defined name.
  "metadata": {
    "app_id": "foo",
    "namespace": "bar",
    "target": {
      "broadcast": {
        "actor_subscription": {
          "id": "my-actor-id",
          "type": "my-actor-type"
        },
      },
    },
  },
  "job": {
    "due_time": "2562047h",
    "data": {} // the proto encoded manifest of the actor subscription.
  }
}
```

Trigger:
```json
{
  "name": "my-actor-subscription",
  "metadata": {
    "app_id": "foo",
    "namespace": "bar",
    "target": {
      "broadcast": {
        "actor_subscription": {
          "id": "my-actor-id",
          "type": "my-actor-type"
        },
      },
    },
  },
  "data": {
    "type": "BROADCAST_JOB_EVENT_PUT",
    "subscription": {}
  }
}
```

### Durable Actor IDs

It is the case that applications require actor IDs to be durably instantiated for periods of time, including across dissemination events and restarts.
To get around this today, applications are forced to use work arounds like creating a reminders that are triggered repeatedly to keep the actor alive.
To make durable actor IDs a first class concept and improve performance, the Scheduler will implement a durable actor ID broadcast job type.
Upon receiving the job or after dissemination events, the actor runtime will check whether the actor ID is associated with that host, then invoke the actor according to the invocation request encoded.
The existence of the job will override the actor deactivation timeout, and the actor will be kept alive until the job is deleted.

The actor runtime will receive events about the job;
- on put events the actor runtime will invoke the underlying durable actor ID with the
  encoded invocation request.
- on delete events the actor runtime will delete the durable actor ID.
- there should not be any trigger events.

The Durable Actor ID broadcast job will take the following form.
Notice that the due time may be set to infinite (the actor ID can only be deleted via a delete event), or an optional TTL can be set to have automatic deletion.
Since the job will be created abstractedly from the application by the runtime, the name will be generated with the following form; `__internal__$actor-type__$actor-id`.
This job type will return an `already exists` error if a PUT is attempted on an existing durable Actor ID.
The data field of the job will include a `common.InvokeRequest` proto message, detailing how the actor will be invoked.
The HTTP Verb of the app Actor invocation request defined will be ignored, in favour of always using a POST or DELETE.

Schedule:
```json
{
  "name": "__internal__my-actor-id__my-actor-type",
  "metadata": {
    "app_id": "foo",
    "namespace": "bar",
    "target": {
      "broadcast": {
        "actor_subscription": {
          "id": "my-actor-id",
          "type": "my-actor-type"
        },
      },
    },
  },
  "job": {
    "due_time": "10h",
    "data": {  // the proto encoded common.InvokeRequest of the actor invocation.
      "invoke_request": {
        "method": "/foo",
        "data": 1234
      }
    }
  }
}
```

Trigger:
```json
{
  "name": "__internal__my-actor-id__my-actor-type",
  "id": 1234,
  "metadata": {
    "app_id": "foo",
    "namespace": "bar",
    "target": {
      "broadcast": {
        "actor_subscription": {
          "id": "my-actor-id",
          "type": "my-actor-type"
        },
      },
    },
  },
  "data": {
    "type": "BROADCAST_JOB_EVENT_PUT",
    "invoke_request": {
      "method": "/foo",
      "data": 1234
    }
  }
}
```
