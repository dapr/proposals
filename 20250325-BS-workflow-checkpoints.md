# Workflow Checkpointing
- Author: @whitwaldo

## Overview
This design proposal augments the implementation for Workflows to support the creation of sub-workflows from 
"checkpoints" within existing workflows.

## Background
During the maintainers call on 3/24/25, @joshvanl talked about a customer request to support a means for reviving a
workflow from a terminal state and resuming it from an earlier point with different inputs. While thinking about how
this might look from an SDK perspective and more generally, I thought of an alternative way to approach this that 
I raised on the call and wanted to detail more specifically here for consideration.

## Related Items

## Implementation Details
Where a Dapr Workflow will be created and run by an external scheduling, it will run until it hits one of several 
states (e.g. terminated, completed, cancelled). Today, the only mechanism available to run similar work again is to 
start the workflow from scratch. This isn't necessarily ideal - take the following:

A customer is executing a multi-step workflow in which a food order comes in. In a starting activity, they ensure they
are open for business and in another verify that they have sufficient staff to create the order. The customer's payment
mechanism is charged for the value of the transaction in another activity and in still another, an 
activity runs to validate whether there is sufficient inventory of the ingredients to prepare the selected order and,
here, the activity returns false and the workflow completes. In today's Dapr Workflows, there's no Saga support and no
mechanism to roll back the customer's transaction, so this has to be accommodated through alternative means. 

This proposal instead envisions a circumstance where the ingredients are sourced a few minutes later and the workflow 
could theoretically be resumed from the point before the failed activity ran, the activity could be re-run and produce 
different results and the workflow continued from where it left off retaining the same state it had up until this point.

While Workflows must be deterministic, this change honors this requirement in that it's more like a resume operation
from a cloned event source state under a new workflow ID so nothing about the original workflow up until that point 
is changed (including inputs and outputs to each of the already-completed activities). Rather that allow new inputs
to be provided, this instead maintains the current constraints around workflows and simply provides the opportunity for
the only non-deterministic aspects of workflows, the activities, to return a different value in a "try-again" state.

### Design
The idea spans two key changes to today's workflow with several ancillary adjustments:
- A mechanism to create a clone of an existing workflow instance's event source state identified by a string value 
from workflow context
- A mechanism similar to `RaiseEventAsync` that allows the scheduling of a new checkpointed workflow that instead 
resumes from the specified event source state in a standing workflow, but uses a new instance ID. 

Where today's event source is strictly bound to the instance ID it's created for, this should be modified to also
persist a checkpoint name string, e.g. `$instanceId` to `$instanceId||$checkpointName`. By default, `$checkpointName` 
should be an empty string emblematic of the workflow instance it's initially associated with and only named checkpoints
should have non-empty `$checkpointName` string values.

Every time the workflow context signals the need to create a checkpoint, the runtime should clone the event source
state for the current workflow instance ID and assign the given checkpoint name to it. It's very likely that not
all checkpoints will ever be resumed, so it'll be important to specify in the documentation that use of this feature
should be paired with TTL and generous use of `Purge` is encouraged when an instance ID is known to have completed 
fully and associated checkpoints aren't necessary any longer.

Otherwise, when the user, from the `DaprWorkflowClient` schedules a new workflow from a checkpoint, they specify the
workflow instance ID it's based on (as the inputs for any given workflow could vary) along with the name of the workflow
and an optional new workflow instance ID (else one will be created for them as happens normally). The indicated event 
source data in the checkpoint should be copied into the new workflow ID (if given, otherwise generated) and the workflow
should otherwise replace as though it were just a retry operation like any other up to the step following
the checkpoint operation (as that would be part of the current event source state) and pick up where it left off setting
an `OrchestrationRuntimeStatus` of `Running` accordingly. 

By having the newly created workflows make their own copies of the checkpointed event sources, it allows for multiple
workflows to be created from the same checkpoint, should subsequent runs not yield expected changed values from the
subsequent activities, for example.

### SDK
Dapr Workflow SDKs will be updated to include a new method for both creating named checkpoints and scheduling 
checkpointed workflows as new. 

The creation of new checkpoints would happen within the `WorkflowContext` and might look
like the following in .NET:

```c#
await context.CreateCheckppoint("MyCheckpoint");
```

When the developer wishes to create a workflow from a previous checkpoint, they would call something like the following
from the `DaprWorkflowClient`, again per .NET:

```c#
await daprWorkflowClient.ScheduleNewOrchestrationFromCheckpoint(nameof(DemoWorkflow), existingInstanceId, "MyCheckpoint", newInstanceId);
```

### Protos
The following protos will need to be added: 

```protobuf
// Starts a new instance of a checkpointed workflow
rpc StartCheckpointedWorkflowBeta1 (StartCheckpointedWorkflowRequest) returns (StartCheckpointedWorkflowResponse) {}

// Creates a new checkpoint of a given workflow
rpc CheckpointWorkflowBeta1 (StartCheckpointWorkflowRequest) returns (google.protobuf.Empty) {}

// StartCheckpointedWorkflowRequest is the request for StartCheckpointedWorkflowBeta1
message StartCheckpointedWorkflowRequest {
  // The ID to assign to the started workflow instance. If empty, a random ID is generated.
  string instance_id = 1 [json_name = "instanceID"];
  // Name of the workflow component.
  string workflow_component = 2 [json_name = "workflowComponent"];
  // Name of the workflow.
  string workflow_name = 3 [json_name = "workflowName"];
  // Name of the checkpoint
  string checkpoint_name = 4 [json_name = "checkpointName"];
  // Additional component-specific options for starting the workflow instance.
  map<string, string> options = 4;
}

// StartCheckpointedWorkflowResponse is the response for StartCheckpointedWorkflowBeta1
message StartCheckpointedWorkflowResponse {
  string instance_id = 1 [json_name = "instanceID"];
}

message StartCheckpointWorkflowRequest {
  // The ID of the workflow instance
  string instance_id = 1 [json_name = "instanceID"];
  // Name of the checkpoint
  string checkpoint_name = 2 [json_name = "checkpointName"];
}
```

This does not risk corruption of existing workflows as, like `ContinueAsNew` it creates an entirely new workflow but
from its own event source data sourced from the checkpoint event source data. Further, it doesn't violate the existing
requirements around the non-deterministic nature of workflows nor require any dramatic changes to the workflow 
concept as a whole and should be a fairly minor change outside of the event source cloning operation. SDK impact is
minimal as it merely requires the addition of two methods and a protos refresh.