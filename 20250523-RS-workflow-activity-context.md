# Enrich workflow activity context

* Author(s): Javier Aliaga (@javier-aliaga)
* State: Draft
* Updated: 2025-05-26

## Overview

This is a proposal to enrich workflow activity context by introducing new fields. This will make it easier for users to control the workflow's state within activities. They are particularly useful for:

1. **Idempotency**: Ensuring activities are not executed multiple times for the same task
2. **State Management**: Tracking the state of activity execution

## Background

Workflow users have raised concerns about controlling and preventing an activity from being invoked more than once or tracking the state of the execution. Input parameters can be used to control the workflow's state, but certain scenarios do not have enough data to do it. Some examples are:

- A notification activity where the message content alone isn't enough to determine uniqueness
- An external service call where idempotency can't be guaranteed by the input parameters

The current implementations of the activity context do not expose any valuable field for this purpose.
- [GO-SDK](https://github.com/dapr/go-sdk/blob/main/workflow/activity_context.go)
- [JAVA-SDK](https://github.com/dapr/java-sdk/blob/master/sdk-workflows/src/main/java/io/dapr/workflows/WorkflowActivityContext.java)
- etc...

This limitation creates the need to extend the activity context with additional fields that can track execution uniqueness and state. By introducing these new fields to the activity context, users will have a better mechanism to ensure activities are not invoked multiple times, even in cases where input parameters are not enough for this purpose.


## Related Items

### Related proposals 

N/A

### Related issues

An attempt to solve this problem has been tried in the JAVA-SDK. However, the solution is not consistent with scenarios where the same task is executed multiple times as the task execution key is built using the workflow instance id and the task name.

- https://github.com/dapr/durabletask-java/pull/18
- https://github.com/dapr/java-sdk/pull/1352

## Expectations and alternatives

* What is in scope for this proposal?
  * Workflow runtime
  * SDKs
* What advantages / disadvantages does this proposal have?
  * Uniform across all SDKs

## Implementation Details

### Design

The proposed fields introduced in this document are:

- WorkflowInstanceId: This field will provide a unique identifier for the workflow instance. It is already available in the orchestration context but will now be propagated to the activity context.

- TaskInstanceId: This field will provide a unique identifier for the same activity among retries. This new field will be part of the [Activity Request](https://github.com/dapr/durabletask-protobuf/blob/main/protos/orchestrator_service.proto) and needs to be populated in the runtime.

- RetryAttempt: This field will contain the current retry count for the activity execution.  


### Feature lifecycle outline

* Expectations
* Compatability guarantees
* Deprecation / co-existence with existing functionality
* Feature flags

### Acceptance Criteria

How will success be measured? 

* Integration and unit tests will be added to verify the new functionality.


## Completion Checklist

What changes or actions are required to make this proposal complete? Some examples:

* Code changes
* Tests added (e2e, unit)
* SDK changes (if needed)
* Documentation

