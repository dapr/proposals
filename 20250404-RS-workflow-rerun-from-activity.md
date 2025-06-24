# Workflow: Rerun from Activity

* Author(s): @joshvanl, @whitwaldo

## Overview

This proposal details the ability to rerun a workflow from a previous point in its history.
A workflow in a terminal state can be rerun from a failed activity, before the failed activity, or at any activity in the history of a successful or failed workflow.

## Background

It is often the case that it's desirable to re-run business logic implemented inside a workflow.
This could be because an activity in the workflow failed due to a transient error, an external dependency changed or a resource is now available to an activity, or it's just desirable for some subset of the last set of Activities to be rerun.
Dapr should provide the functionality to rerun a workflow from any point in its history.

## Related Items

https://github.com/dapr/proposals/pull/79

## Design

The following proto RPC and messages will be added to durabletask, exposed via each SDK.
This API implements rerunning a workflow from a specific Activity in the history of the workflow.
The Activity to be rerun is chosen via its associated event ID.

All `Activties` are assigned an event ID in the durabletask history.
While `Timers` and `RaiseEvents` tasks are also assigned an event ID in the durabletask history, the workflow cannot be rerun from these events using this API.
Not only would supporting rerunning the workflow from these two event types require a significant code refactor, in practice, users are only interested in rerunning workflow from a specific _activity_, not a "control event".

It must be the case that the workflow is in a _terminal_ state before the rerun can be executed.
This would be because the workflow has completed successfully, failed at some activity, or force terminated.
Rerunning a workflow which is currently in progress does not make any practical or academic sense.
Attempting to do so will return an error to the client.

When rerunning a workflow, the workflow will be started from the event ID of an Activity in the history.
The client must give a _new_ input to the Activity to which the workflow will be rerun from.
The workflow history up until the event ID of the Activity will be cloned.
If defined, the activity will be started with the new input data.
If no input is given, the activity will be started with the same input as the original workflow.

The client can optionally give a new instance ID to use when rerunning the workflow.
This is useful for when the client wishes to preserve the history of the source workflow that is being rerun.
`RerunWorkflowFromActivity` must have a new instance ID to clone the workflow up until the event ID from.

If the targeted `eventID` does not exist, or is not an Activity event, the API will return an error to the client.

```proto
service TaskHubSidecarService {
    // Rerun a Workflow from a specific event ID from an activity.
    rpc RerunWorkflowFromActivity(RerunWorkflowFromActivityRequest) returns (RerunWorkflowFromActivityResponse);
}

// RerunWorkflowFromActivityRequest is used to rerun a workflow instance from a
// specific event ID.
message RerunWorkflowFromActivityRequest {
  // instanceID is the orchestration instance ID to rerun.
  string instanceID = 1;

  // the event id to start the new workflow instance from.
  int32 eventID = 2;

  // newInstanceID is the new instance ID to use for the new workflow
  // instance.
  string newInstanceID = 3;

  // input can optionally given to give the new instance a different input to
  // the next Activity event.
  google.protobuf.StringValue input = 4;
}

// RerunWorkflowFromActivityResponse is the response to executing
// RerunWorkflowFromActivity.
message RerunWorkflowFromActivityResponse {
    string instanceId = 1;
}
```

The Orchestration protos will be updated to include a new `uint64 attempt` field which signals the attempt number which the current workflow is on.
Starts from zero.
Each rerun will increment the attempt number by one.
The Orchestration protos will also include an optional `optional string rerunFrom` which will be set to the instance ID for which the workflow was rerun from.
If the workflow is not created from a rerun, this field will be nil.

### Getting Instance History

As a compliment to the `RerunWorkflowFromActivity` API, a new API is added to get the history of run activities for a workflow instance.
Note that the API returns _all_ history events for the workflow instance, including control events which do _not_ contain an event ID.
This API is intended to be used for discovering the event ID of the activity to rerun from.
The actor backend will get the instance history from the state store and return it to the client, using a new workflow Actor invoke method.

```proto
service TaskHubSidecarService {
    // GetInstanceHistory retrieves the history of a workflow instance.
    rpc GetInstanceHistory(GetInstanceHistoryRequest) returns (GetInstanceHistoryResponse);
}

// RerunWorkflowFromActivityResponse is the response to executing
// RerunWorkflowFromActivity.
message RerunWorkflowFromActivityResponse {
    string instanceId = 1;
}

// GetInstanceHistoryRequest is used to get the history of a workflow instance.
message GetInstanceHistoryRequest {
  // instanceID is the orchestration instance ID to get the history for.
  string instanceID = 1;
}

// GetInstanceHistoryResponse is the response to executing
// GetInstanceHistoryRequest.
message GetInstanceHistoryResponse {
  repeated HistoryEvent events = 1;
}
```

### Concurrent Activities

It is often the case that workflow activities are run concurrently, i.e. in fan-out patterns.
This means the resulting workflow history order of execution can be non-deterministic.
The durabletask history is currently a linear sequence of events.
This then means that rerunning a workflow from a specific Activity which is a member of a fan-out pattern will result in possible rerunning of peer fan-out activities, depending on the order of termination of Activities in the fan-out group.
This may or may not be desirable to the user, but is otherwise a limitation of the API.

Users who wish to use this API in a regular fashion and in expected places should be advised to make use of "checkpoint" activities.
These "checkpoint" activities should be a no-op- returning an output that is the same as the input.
These checkpoint activities are useful as well-known activity event ID markers where the user knows it will be desirable to rerun the workflow regularly.

```go
func checkpoint(ctx task.ActivityContext) (any, error) {
	var input int
	return input,ctx.GetInput(&input)
}
```

### SDK Changes

This API will be exposed on all durabletask SDKs.
The semantics are generally dependant on the flavour of each SDK language, however-
- The `instanceID` is a required string to target for the rerun.
- The `eventID` is a required int32 to target the Activity for the rerun.
- The `newInstanceID` is a required string, and reruns the workflow from the event ID of the Activity to the new instance ID.
- An optional `input` which, if defined, will be used as the input to the targeted Activity when rerunning the workflow.

## Completion Checklist

* Implement proto & API changes to durabletask.
* Update dapr workflow runtime to support the new APIs.
* Update SDKs to support the new APIs.
