# Multi-App Workflows

* Author(s): Josh van Leeuwen (@joshvanl)

## Overview

This design proposal describes the implementation for Workflows to be executed across multiple Dapr applications (app IDs).

## Background

It is the case that users want to execute some of a Workflow pipeline across both applications and machines.
This can be due to different parts of the pipeline being implemented by different applications, execution isolation, or hardware requirments such as GPUs in cases of AI/ML workflow activities.
To achieve this, Dapr Workflows will introduce cross-application Workflow support, whereby a single Workflow can be executed across multiple app IDs.
Developers can define Orchestrations and Activities as normal, and then optionally specify remote app IDs for Activities or Orchestrations to be executed on.

## Related Items

https://github.com/dapr/dapr/issues/8578
https://github.com/dapr/dapr/issues/8556

## Implementation Details

### Design

#### SDK

Dapr Workflow SDKs will be updated to include new optional fields for Orchestration and Activity calls.
Both client, and in-orchestration or activity blocks can define a target app ID for execution.

Below are some examples in Go:

```go
client.ScheduleNewOrchestration(ctx, "foo", api.WithAppID("app1"))
```

```go
	reg.AddOrchestratorN("xyz", func(ctx *task.OrchestrationContext) (any, error) {
		 ctx.CallActivity("abc", task.WithActivityInput("hello world"), task.WithAppID("app3")).Await(nil))
		return nil, nil
	})
```

```go
_, err = client.WaitForOrchestrationCompletion(ctx, id, api.WithAppID("app3"))
```


#### durabletask

The durabletask protobuf API will be updated with a new optional `appID` field on all messages where an existing `instanceId` is defined.
No other durabletask changes need to be made, beyond plumbing this field.


```proto
message HistoryEvent {
    int32 eventId = 1;
    google.protobuf.Timestamp timestamp = 2;
    oneof eventType {
        ExecutionStartedEvent executionStarted = 3;
        ExecutionCompletedEvent executionCompleted = 4;
        ...
    }
    string instanceId = 23;
    optional string appId = 24;
}
```

```proto
message OrchestrationInstance {
    string instanceId = 1;
    google.protobuf.StringValue executionId = 2;
    optional string appId = 3;
}
```

```proto
message CreateSubOrchestrationAction {
    string instanceId = 1;
    string name = 2;
    google.protobuf.StringValue version = 3;
    google.protobuf.StringValue input = 4;
    optional string appID = 5;
}
```

#### Dapr Runtime

The Dapr workflow engine will be updated to support the new `appID` field on durabletask messages.
Upon receiving a work item which contains a `appID` field, the Dapr runtime will route the message to the correct actor type for the target app ID.
Remember that the actor type of Orchestration and Activities contains the app ID, so the Dapr runtime can route the message to the correct actor instance.
There are no issues with cross app ID state access, as state is passed via the reminder, and the Orchestration actor handles database state transactions.

```
dapr.internal.<namespace>.<appID>.workflow
dapr.internal.<namespace>.<appID>.activity
```

There are no security considerations since all actor types in the same namespace can already communicate openly.
