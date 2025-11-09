# Workflow: History Purge TTL on Completion

* Author(s): @joshvanl

## Overview

This proposal details new functionality to the workflow runtime to give users the ability to delete completed workflow state from the actor state store after some configured time.
All workflow instances may be configured with a unique TTL at workflow execution time.
The default remains that workflow state will _not_ be deleted form the actor state store, and will remain there indefinitely.

## Background

It is currently the case that in order for users to delete old workflow state from the actor state store database, they either need to use the Purge Workflow API, or delete state from the database directy, either via out of Dapr database operations, or via using some kind of first class TTL feature of that database.
Users typically want to delete old workflow state after some period of time from when the workflow has reached a terminal state.

https://github.com/dapr/dapr/issues/9020

## Design

When scheduling a workflow, users will be able to configure some duration which upon elapsing after the workflow has reached a terminal state, the workflow will be purged from the actor state store.
The duration will only start once the workflow has reached either a TERMINATED, COMPLETED, or FAILED state.

Any duration may be given, i.e. days, weeks, or years.
A duration of `0` may also be given, if the workflow actor state is wished to be deleted immediately after reaching a terminal state.

### Usage

#### CLI

Users can give a Go style duration string when running a workflow from the CLI.

```bash
$ dapr run my-workflow --purge-ttl=5d
```

```bash
$ dapr run my-workflow --purge-ttl=0s
```

The new purge reminders will be displayed by:

```bash
$ dapr scheduler list
NAME                        BEGIN  COUNT  LAST TRIGGER
purge-workflow/my-workflow  96h    0
```

#### Go

```go
wf.ScheduleWorkflow(ctx, "my-workflow", workflow.WithPurgeTTL(time.Hour*24*5))
```

```go
wf.ScheduleWorkflow(ctx, "my-workflow", workflow.WithPurgeTTL(0))
```

#### Python

```python
wfClient.schedule_new_workflow(workflow=my_workflow, putge_ttl=timedelta(days=5))
```

```python
wfClient.schedule_new_workflow(workflow=my_workflow, putge_ttl=timedelta(seconds=0))
```

#### Javascript

```js
workflowClient.scheduleNewWorkflow({workflow: MyWorkflow, putge_ttl: Temporal.Duration.from({days: 5})})
```

```js
workflowClient.scheduleNewWorkflow({workflow: MyWorkflow, putge_ttl: Temporal.Duration.from({})})
```

#### .NET

```dotnet
workflowClient.ScheduleNewWorkflowAsync(
  name: nameof(MyWorkflow),
  stateTTL: TimeSpan.FromDays(5);
);
```

```dotnet
workflowClient.ScheduleNewWorkflowAsync(
  name: nameof(MyWorkflow),
  stateTTL: TimeSpan.FromSeconds(0)
);
```

#### Java

```java
workflowClient.scheduleNewWorkflow(OrderProcessingWorkflow.class, TODO: @joshvanl);
```

```java
workflowClient.scheduleNewWorkflow(OrderProcessingWorkflow.class, TODO: @joshvanl);
```

### Runtime

#### protos

The following protos will be updated with the new state TTL duration field so it is piped from workflow creation to execution.

The new option will be added to `CreateInstanceRequest`, populated by the client.

```proto
message CreateInstanceRequest {
    string instanceId = 1;
    string name = 2;
    // OTHERS
    google.protobuf.Duration purgeTTL = 10; // NEW
}
```

`ExecutionStartedEvent` will contain the TTL duration which signals the duration after which the workflow has completed should be purged.
This field will be persistent in the history log.
This field will be populated by the durabletask backend executor, piping the field from `CreateInstanceRequest`.

```proto
message ExecutionStartedEvent {
    string name = 1;
    // OTHERS
    google.protobuf.Duration purgeTTL = 10; // NEW
}
```

#### Actors

Upon workflow reaching a terminal state, after the orchestraion actor has written the result to the actor state store, it will also create an actor reminder if the `purgeTTL` field is present in the execution started event.

This reminder will target a new actor workflow type, with the reminder name being the instance ID of the workflow.

The new actor type will follow convention and have the following form:

```
dapr.internal.<namespace>.<app-id>.purge-workflow
```

Upon activation of the reminder, the new purge actor will be activate, call the purge API on the workflow orchestrator actor for the given instance ID, and then deactivate itself.
Along with the other workflow actor types, this type will be registered on workflow client connection, and unregistration on workflow client disconnection.

By using a new actor type, this feature is fully backwards compatible since older clients will not register for this new purge workflow type.


```
WORKFLOW COMPLETE -> orestrator -> create purge reminder -...> execute purge reminder -> execute purge actor -> execute purge on orchestrator
```

# Alternatives

Another option is to use the actor TTL state store functionality to delete store keys based on individual key TTls.
This is not appropriate as it _must_ be the case that workflow data be only delete from the state store once the workflow has reached a terminal state.
Not doing so would corrupt the workflow processing.
It is therefore necessary that the Purge API is used to delete the stored data, which itself processes the request inside the same workflow state machine.
