# Workflow: History State Retention

* Author(s): @joshvanl

## Overview

This proposal details new functionality to the workflow runtime to give users the ability to delete old workflow state from the actor state store after some configured time.
An app scoped configuration policy can be given to set a TTL for all workflows in that app.
All workflow instances may be configured with a unique TTL at workflow scheduling  time.
The default remains that workflow state will _not_ be deleted from the actor state store, and will remain there indefinitely.

## Background

It is currently the case that in order for users to delete old workflow state from the actor state store database, they either need to use the Purge Workflow API, or delete state from the database directly, either via out of Dapr database operations, or via using some kind of first class TTL feature of that database.
Users typically want to delete old workflow state after some period of time from when the workflow has reached a terminal state.

https://github.com/dapr/dapr/issues/9020

## Design

A configuration spec will be added to the runtime workflow spec to set a retention policy for all workflow created from that app.
When scheduling a workflow, users will be able to configure some duration which upon elapsing after the workflow has reached a terminal state, the workflow will be purged from the actor state store.
The duration will only start once the workflow has reached either a TERMINATED, COMPLETED, or FAILED state.

If a retention policy is set at both the app level and the workflow level, the workflow level setting will take precedence.

Any duration may be given, i.e. days, weeks, or years.
A duration of `0` may also be given, if the workflow actor state is wished to be deleted immediately after reaching a terminal state.

### Usage

#### Configuration

```go
type WorkflowSpec struct {
	// StateRetentionPolicy defines the retention configuration for workflow
	// state once a workflow reaches a terminal state. If not set, workflow
	// instances will not be automatically purged.
	StateRetentionPolicy *WorkflowStateRetentionPolicy `json:"stateRetentionPolicy,omitempty" yaml:"stateRetentionPolicy,omitempty"`
}

// WorkflowStateRetentionPolicy defines the retention policy of workflow state
// for workflow instances once they reaches a specific or any terminal state.
// If not set, workflow instances will not be automatically purged. If a
// specific and any terminal state are both set, the specific terminal state
// takes precedence. Accepts duration strings, e.g. "72h" or "30m", including
// immediate values "0s".
type WorkflowStateRetentionPolicy struct {
	// AnyTerminal is the TTL for purging workflow instances that reach any
	// terminal state.
	AnyTerminal *time.Duration `json:"anyTerminal,omitempty" yaml:"anyTerminal,omitempty"`

	// Completed is the TTL for purging workflow instances that reach the
	// Completed terminal state.
	Completed *time.Duration `json:"completed,omitempty" yaml:"completed,omitempty"`

	// Failed is the TTL for purging workflow instances that reach the Failed
	// terminal state.
	Failed *time.Duration `json:"failed,omitempty" yaml:"failed,omitempty"`

	// Terminated is the TTL for purging workflow instances that reach the
	// Terminated terminal state.
	Terminated *time.Duration `json:"terminated,omitempty" yaml:"terminated,omitempty"`
}
```

```yaml
kind: Configuration
metadata:
  name: wfpolicy
spec:
  workflow:
    stateRetentionPolicy:
      anyTerminal: "5s"
      completed: "0s"
      failed: "999h"
      terminated: "999h"
```

#### CLI

Users can give a Go style duration string when running a workflow from the CLI.

```bash
$ dapr run my-workflow --state-retention=120h
```

```bash
$ dapr run my-workflow --state-retention=0s
```

The new retention reminders will be displayed like:

```bash
$ dapr scheduler list
NAME                            BEGIN  COUNT  LAST TRIGGER
workflow-retention/my-workflow  96h    0
```

#### Go

```go
wf.ScheduleWorkflow(ctx, "my-workflow", workflow.WithStateRetention(time.Hour*24*5))
wf.ScheduleWorkflow(ctx, "my-workflow", workflow.WithStateRetention(0))
wf.ScheduleWorkflow(ctx, "my-workflow", workflow.WithStateRetentionCompleted(0))
wf.ScheduleWorkflow(ctx, "my-workflow", workflow.WithStateRetentionFailed(time.Hour*24*5))
```

#### Python

```python
wfClient.schedule_new_workflow(workflow=my_workflow, state_retention=timedelta(days=5))
wfClient.schedule_new_workflow(workflow=my_workflow, state_retention=timedelta(seconds=0))
wfClient.schedule_new_workflow(workflow=my_workflow, state_retention_completed=timedelta(seconds=0))
wfClient.schedule_new_workflow(workflow=my_workflow, state_retention_failed=timedelta(days=5))
```

#### Javascript

```js
workflowClient.scheduleNewWorkflow({workflow: MyWorkflow, state_retention: Temporal.Duration.from({days: 5})})
workflowClient.scheduleNewWorkflow({workflow: MyWorkflow, state_retention: Temporal.Duration.from({})})
workflowClient.scheduleNewWorkflow({workflow: MyWorkflow, state_retention_completed: Temporal.Duration.from({})})
workflowClient.scheduleNewWorkflow({workflow: MyWorkflow, state_retention_failed: Temporal.Duration.from({days: 5})})
```

#### .NET

```dotnet
workflowClient.ScheduleNewWorkflowAsync(
  name: nameof(MyWorkflow),
  stateRetention: TimeSpan.FromDays(5);
);
workflowClient.ScheduleNewWorkflowAsync(
  name: nameof(MyWorkflow),
  stateRetention: TimeSpan.FromSeconds(0)
  stateRetentionCompleted: TimeSpan.FromSeconds(0)
  stateRetentionFailed: TimeSpan.FromDays(5)
);
```

#### Java

```java
opts.setStateRetention(Duration.ofDays(5));
opts.setStateRetentionCompleted(Duration.ofSeconds(0));
opts.setStateRetentionFailed(Duration.ofDays(5));
workflowClient.scheduleNewWorkflow(OrderProcessingWorkflow.class, opts);
```

### Runtime

#### protos

The following protos will be updated with the new retention policy message so it is piped from workflow creation to execution.

The new option will be added to `CreateInstanceRequest`, populated by the client.

```proto
message CreateInstanceRequest {
    string instanceId = 1;
    string name = 2;
    // ...
    optional InstanceStateRetentionPolicy retentionPolicy = 10; // NEW
}

message InstanceStateRetentionPolicy {
  optional google.protobuf.Duration allTerminal = 1;
  optional google.protobuf.Duration completed = 2;
  optional google.protobuf.Duration failed = 3;
  optional google.protobuf.Duration terminated = 4;
}
```

`ExecutionStartedEvent` will contain the retention policy which signals the duration after which the workflow has completed should be purged.
This field will be persistent in the history log.
This field will be populated by the durabletask backend executor, piping the field from `CreateInstanceRequest`.

```proto
message ExecutionStartedEvent {
    string name = 1;
    // EXISTING
    optional InstanceStateRetentionPolicy = 10; // NEW
}
```

#### Actors

Upon workflow reaching a terminal state, after the orchestration actor has written the result to the actor state store, it will then create an actor reminder if the state retention policy field is present in the execution started event or as the app ID workflow configuration.

This reminder will target a new actor workflow type, with the actor ID being the instance ID of the workflow.

The new actor type will follow convention and have the following form:

```
dapr.internal.<namespace>.<app-id>.retentioner
```

Upon activation of the reminder, the new retentioner actor will be activated, call the purge API on the workflow orchestrator actor for the given instance ID, and then deactivate itself.
Along with the other workflow actor types, this type will be registered on workflow client connection, and unregistered on workflow worker client disconnection.

By using a new actor type, this feature is fully backwards compatible as older clients will not register for this new purge workflow type.

```
WORKFLOW COMPLETE -> orestrator -> create retentioner reminder -...> execute retentioner reminder -> execute retentioner actor -> execute purge on orchestrator
```

# Alternatives

Another option is to use the actor TTL state store functionality to delete store keys based on individual key TTls.
This is not appropriate as it _must_ be the case that workflow data be only delete from the state store once the workflow has reached a terminal state.
Not doing so would corrupt the workflow processing.
It is therefore necessary that the Purge API is used to delete the stored data, which itself processes the request inside the same workflow state machine.
