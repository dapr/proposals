# Workflow Patching and Versioning
- Author: Whit Waldo (@whitwaldo)
- Status: Proposed
- Introduced 10/28/2025

## Overview
This timely proposal provides a rigorous look at how to implement a robust workflow versioning scheme in Dapr Workflows.
The breadth of the implementation details will be left to the individual SDK maintainers, but I propose an overarching 
concept spanning a "Versioning" and a "Patching" system applied combining great ideas from both 
[this proposal](https://github.com/dapr/proposals/pull/82) documenting named workflow versions and 
[this proposal](https://github.com/dapr/proposals/pull/92) documenting patched-based versioning.

Following quite a lot of discussion on either thread, I would like to propose a union of the two versioning concepts as 
part of a richer Dapr workflow versioning system. The introductory paragraph of the documentation describing this functionality
in the Dapr documentation might read as follows:

> Dapr Workflow versioning provides a robust versioning system for your workflows. Fully compatible with multi-app run,
> resiliency, replay, ContinueAsNew, and re-run capabilities, versioning provides the means to create new versions of 
> your workflow without manually migrating logic from one to the next. Via a straightforward process of creating new 
> versions of your workflow types, Dapr Workflows will seamlessly transition between published versions between 
> successful invocations. For those limited scenarios where more granular control is warranted (e.g., fixing a bug or 
> applying a minor tweak), our versioning strategy also accommodates in-file patching ensuring your workflows remain 
> deterministic and replayable.

I write this proposal to provide a clear and unified view of the two proposals so they might be introduced at the same
time in a refined, complementary, cohesive and well-designed way that showcases Dapr's unique advantages and workflow 
process.

## Background
While discussing either of the previous versioning proposals, it became clear to me that we were solving the same problem
but in two different ways. I was approaching the problem from a place of ensuring a clear break from the deterministic
logic in previous versions, but the counter-proposal sought to provide a mechanism to deterministically apply logical
changes that might be applied to a replaying workflow to make small changes to an existing workflow without necessarily
undertaking the effort of moving to an entirely new type version.

Both are common approaches to the problem - Temporal implements both of them, but I'd also assert that the union of these
two ideas provides a clearer mechanism for addressing workflow versioning than their solution because we're able to 
take advantage of Dapr's unique architectural differences.

The problem we're looking to solve here is complex and unique to the nature of how Azure Durable Functions Workflows and
Dapr Workflows are built. The entire workflow operation is bounded by moments of external invocations passing an input
and receiving an output from some external system (e.g., a child workflow, an activity, a timer or an external interaction).
It's mandatory that Dapr remain deterministic at the workflow level so that these inputs are consistent - this allows
us to perform a replay operation that lets us skip over successful historical boundary invocations and retrieve the outputs
for each, assign those to the workflow variables and proceed accordingly without re-running the actual activities giving
us a richer system than one might get through a checkpointing process. 

This does mean that we face challenges when designing a comprehensive versioning scheme. First and foremost, all versioning
must accommodate the fact that we're looking to change the external boundary invocations in some way in future runs, but
in a way that ensures a still-deterministic process when re-run. In proposal #82, this is achieved by placing a "clean break"
between workflow executions. When a request comes into the app running the Workflow SDK that is part of a new invocation, 
the configured versioning convention is applied to the available workflow types and the "latest" version selected. When 
it runs and reports successful execution back to the Dapr runtime's workflow orchestrator, it includes the name of the
versioned type so that subsequent invocations replaying that type are run against a consistent version. But because new
invocations of the workflow aren't marked with such versioning metadata, it can always run on whatever the "latest" type
is without registering this in any way with the runtime ahead of time.

That's fantastic for changes both large and small, but it means that there's some regular clerical work involved to
address even the smallest of changes (e.g., flip an inadvertent negative sign somewhere in the code).

Neither solution can address a bug that's been introduced to a workflow and causing it to crash (see the Caveat
section towards the end of this document). However, we can certainly accommodate the desire to apply minimalistic changes
to a workflow while opting out of a more-involved process.

In my version of the patching proposal, it's providing a named "Band-Aid" for code, wrapping the "old" and "new" code in 
a simple `if/then` statement with a named identifier. Patches can be applied throughout the whole of a workflow
subject to the following constraints:
- Patches cannot overlap one another
- While patches are applied within a version, they are not themselves versioned
- Patches must still be deterministically replayable - in other words, even if they introduce a new path, if it was not
evaluated during the original run (prior to replays), it will not be evaluated during replays as this would violate
our deterministic constraint.

Generally, the standing guidance would be that developers simply create new types to version their code. As a
(perfectly valid) exception, one or more patches should be applied. They should be understood to fix minor things in the
workflow and are not intended to be a replacement for the whole-type versioning, but rather a complementary and necessary
mechanism to supplement whole-type versioning in limited conditions.

## Assumptions
- All workflow base types and activities are registered in the Workflow SDK at startup as is necessary today
- Versioning should only be performed on workflows that are not in a re-run state - otherwise, the routing mechanism in the
SDK will be responsible for ensuring that the same versioned workflow type is used as was last re-run, absent patch changes.
- This proposal addresses workflow versioning only and not activity versioning. Activities are not required to be
deterministic and can be swapped out, although this generally isn't recommended. Clear documentation should generally
advise against sharing activities across workflows due to the potential for unexpected ramifications.
- All examples in this workflow are written using C# and are subject to differences across SDK implementation as makes 
sense in each example given language norms, conventions and available tooling.
- This proposal assumes a monotonically increasing version number applied to the suffix of the workflow type as one 
of many potentially configurable versioning schemes. Despite opting to use this for consistency and clarity throughout
the proposal, I'm certainly not trying to give the impression that it's the only scheme any Workflow SDK could implement.
- There's no need to ever delete older workflow type versions. Rather, they should be persisted so long as there's any 
workflow utilizing them (e.g., runtime context notes any outstanding workflows in which that type version is used).
- Deleting older workflow types is certainly worth thinking about, but not until after the proposed 
[1.17 release](https://github.com/dapr/proposals/pull/93) of the workflow list and the get history RPCs and their 
availability in the SDKs to query. The biggest blocker to this would be long-running workflows that have never run and
migrated to a newer typed version (and thus are not recorded in the runtime's workflow list). Besides, this history
would only be available at runtime and not at compilation time, limiting the timeliness of the information at this time.

## Design
This proposal suggests using an in-SDK router for versioning because:
1. It minimizes the amount of state required on the runtime in each workflow log to maintain live versioning information
2. It maximizes the flexibility of SDKs to provide workflow conventions that best highlight each SDK's ecosystem and
tooling capabilities.

## What Does Versioning Seek to Solve?
Imagine having the following infinitely looping workflow as implemented in C#. The workflow receives some object identifier
and runs it through an activity to check its status and, if that returns a `true` result, it performs another action to
notify the customer. It then sleeps for 15 minutes and repeats using a `ContinueAsNew` method providing it with the 
original object once again.

```csharp
internal sealed SampleWorkflow : Workflow<string, object?>
{
    public override async Task<object?> RunAsync(WorkflowContext context, string id)
    {
        var status = await context.CallActivityAsync<bool>(nameof(CheckStatus), id);
        if (status) 
        {
            await context.CallActivityAsync(nameof(NotifyCustomer), id);
        }
        
        await context.CreateTimer(TimeSpan.FromMinutes(15));
        context.ContinueAsNew(id, false);
        
        return null;
    }
}
```

Because of the deterministic nature of Dapr Workflows, we have several considerations that must be taken into account
when considering a home-made versioning system here:
- **One cannot simply swap out this workflow type in a new deployment:** Swapping out the workflow type might fail
catastrophically if inlight-workflows were unable to complete or if a replay were performed because of a previously-
encountered failure. This would lead to developer confusion and they might assume that a new deployment would supersede
previous deployment versions.
- **Code cannot simply be added to the workflow**: Following the reasoning of the previous point, adding new code 
violates deterministic integrity as it changes the logic that provides and uses the inputs and outputs at each of the
invocation boundaries (again, there's a singular exception to this noted in the Caveat section below).

If the workflow were to have stopped when it was finished, it would give the developer an opportunity to relaunch it 
using a different type enabling a blue/green-like deployment model - that is, deploy a new "green" type until all the 
existing "blue" types are known to have stopped running and are instead running as "green" types", then delete the "blue"
types from the next deployment altogether. However, in this very likely scenario, the developer never has an opportunity
to swap out Workflow types, especially in those long-running types that have been idle for some time even between new
blue/green deployments.

Now, had the developer thought about this in a previous version, they had options to implement this themselves. They 
could have:
- Introduced an activity that performed some reflective analysis of its own service, or
- Used some sort of Workflow registry to determine that there was a newer type available to swap to.

It could them have scheduled a one-time Dapr JOb with the updated Workflow type name as the payload and ended the current
workflow to have the job invoke the new Workflow type. However, this introduces a lot of complexity and still doesn't
address how bugs in live workflows can be fixed between migrations to new types. Since we're trying to simplify distributed
application development with Dapr, I propose that there's a richer and simpler approach here. I introduce a
powerful mechanism to swap between Dapr Workflow types and introduce patches to outstanding workflows with minimal 
effort by existing Dapr developers.

## Implementation Details
I propose that the internal Dapr Workflow API be slightly modified to include a string value whenever a workflow is
executed. This value contains the name of the versioned type that ran for any given workflow invocation as well as a 
list of each named patch applied and run in that execution.

Different Workflow SDKs should introduce a variety of versioning strategies that best accommodate their unique
capabilities and allow the user to opt-into versioning and select a strategy at startup. Documentation should be written
that strongly urges developers to pick one strategy and stick with it, though future iterations of this in the SDKs
could certainly introduce some mechanism to migrate from one strategy to another.

*Specifics about how this can be implemented in different Dapr SDKs is covered in more detail below and is briefly 
provided here to give a high-level overview of the approach.*

Workflows and activities will be registered with the Dapr Workflow SDK as they are today.

Whenever a request come into the application from the Dapr runtime to run work on a specific workflow following a 
`ScheduleNewWorkflowAsync` call, during the same following a `ContinueAsNew` call or even following a 
`CallChildWorkflowAsync`, a request should come into the SDKs to run that named workflow.

### Version isn't specified on orchestration request
If there is no version specified on the event, the SDK should map the request based on the named base type to the
latest version available in the application and run the workflow there. Every time a patch is evaluated, the "new code" 
route should be taken, and the patch identifier persisted in a list of encountered patches.

At each boundary invocation except workflow orchestration completion, the SDK should return the workflow status with
the (potentially updated) version information defined below containing the name of the workflow type executed and the list 
of named patches followed.

### Version is specified on orchestration request
If the version is supplied, the workflow type should be parsed out and the SDK should ignore "latest" evaluations and use
that type to run the workflow execution. The list of patches should be extracted from the version prototype. At each
evaluation of the patch, the path should be taken that corresponds to whether the patch exists in the list:

- If the patch exists in the version list, the "new code" route should be taken
- If the patch does not exist in the version list, the "old code" route should be taken

#### Version Specification
The version object needs to reflect the name (not the canonical name) of the workflow that was that actually invoked
in the previous execution so the SDKs can override the routing and point directly to this type. It also needs to include
the list of patch identifiers that evaluated as true when this was run so they might replay in the same manner.

I propose that a new prototype be created to reflect this data in a cleanly structured manner:

```protos
// Reflects the workflow version and patches successfully evaluated for workflow replay purposes
message TargetWorkflowVersion {
  // The name of the specific workflow type executed as part of a previous workflow execution
  string workflowTypeName = 1;
  // The list of patches that evaluated as true during the execution
  repeated string patchNames = 2;
}
```

If this is part of a replay, this proto should be sent by the runtime to the SDK as part of a request to start an 
orchestration via the `OrchestratorRequest` message, modified as follows:

```protos
message OrchestratorRequest {
    string instanceId = 1;
    google.protobuf.StringValue executionId = 2;
    repeated HistoryEvent pastEvents = 3;
    repeated HistoryEvent newEvents = 4;
    OrchestratorEntityParameters entityParameters = 5;
    bool requiresHistoryStreaming = 6;
    optional TaskRouter router = 7;
    // This should be populated with the version information from a previous `OrchestratorResponse` if this is a replay
    optional TargetWorkflowVersion version = 8;
}
```

In the response from the SDK to the runtime regarding the orchestration operation, the `OrchestratorResponse` message
should be modified as follows:
```protos
message OrchestratorResponse {
    string instanceId = 1;
    repeated OrchestratorAction actions = 2;
    google.protobuf.StringValue customStatus = 3;
    string completionToken = 4;

    // The number of work item events that were processed by the orchestrator.
    // This field is optional. If not set, the service should assume that the orchestrator processed all events.
    google.protobuf.Int32Value numEventsProcessed = 5;
    // While marked as optional for backwards compatibility, this should always be populated by versioned workflows
    optional TargetWorkflowVersion version = 6;
}
```

If the operation was a replay, the `version` field should be populated with the original information provided by the
`OrchestratorRequest` message. If not, the `version` field should be populated with the name of the actual workflow
type that executed the operation in the SDK and a list of each patch identifier that evaluated as true during execution.

## Practical Example
This proposal diverges in a few refined ways from the proposal at #82. Assuming a C# application, registration
does not have to differ in any meaningful way from how it exists today with the exception of the versioning opt-in and 
configuration:

```csharp
var builder = Host.CreateDefaultBuilder(args).ConfigureServices(services => {
    services.AddDaprWorkflow(options => {
        options.WithVersioning(opt => opt.Strategy = DaprWorkflowVersioningStrategy.NumerialSuffix); //Adding this enables versioning on this application
    
        // Standard workflow registration (no need to record any versioned types except the base type absent suffix)
        options.RegisterWorkflow<MyWorkflow>();
        
        // Standard activity registration
        options.RegisterActivity<MyActivity>();
        options.RegisterActivity<MyOtherActivity>();         
    });
});
```

### Name-Typed Workflows
To create a new named-type workflow, the developer would create a new type with the same name as their existing workflow type
and modify the name to reflect the next version (e.g., increase the number on the name suffix by one or apply today's
date). Documentation would suggest that the older type be kept in a separate directory, e.g., "Archive" that contains all
the deprecated workflow types no longer in active use to avoid polluting the directory. The developer is then free to make
as many changes as they would like to the new workflow without consideration for backwards compatibility with the former
version as migration will be performed as a clean transition between successful executions.

### Patches
Patches are applied to workflows to apply simple code changes that just don't necessarily need a whole type version
to accommodate. They are intentionally narrow in scope and capability as they're intended to provide a "Band-Aid" style
change to an existing workflow that introduces another path in future invocations that differs from the "old code" path.

In the C# SDK, a method would be introduced to the `WorkflowContext` instance (typically provided to the workflow in a 
variable called `context`) called `IsPatched`, e.g., `context.IsPatched(string name)`. The name parameter is used to 
differentiate patches throughout a versioned type and is distinct to that version of the workflow type. In other words,
a patch name can be re-used in other workflows and workflow versions without any chance of conflict. In that sense, it 
provides mild documentation capabilities.

The return type of a call to `context.IsPatched` is a non-nullable boolean value meaning that it's intended to be used 
only to provide an `if/else` statement to the workflow code.

A call to `IsPatched`:
- Returns `true` if the workflow is not presently replaying
- Returns `true` if the workflow is replaying and the patch name is present in the version information
- Returns `false` if the workflow is replaying and the patch name is **not** present in the version information

Patches themselves are not versioned. They exist within the scope of the versioned workflow type, but they themselves
do not exist beyond two states.

##### Example
The following demonstrates what a workflow supplemented with a patch might look like in a C# application:

```csharp
internal sealed SampleWorkflow : Workflow<string, object?>
{
    public override async Task<object?> RunAsync(WorkflowContext context, string id)
    {
        var status = await context.CallActivityAsync<bool>(nameof(CheckStatus), id);
        if (status) 
        {
            await context.CallActivityAsync(nameof(NotifyCustomer), id);
        }
        
        if (context.IsPatched("shorten-window")) {
            // New code
            await context.CreateTimer(TimeSpan.FromMinutes(5));
        } else {
            // Old code
            await context.CreateTimer(TimeSpan.FromMinutes(15));
        }
                
        context.ContinueAsNew(id, false);
        
        return null;
    }
}
```

## Caveats
Determinism is still mandatory. Neither #82, #92 nor this proposal proffer a fix for a scenario in which there's a bug
in the workflow that's currently wreaking havoc on the system and needs to be fixed. In the sole case that there's an
exception consistently throwing on a workflow such that history has not been preserved beyond that point (meaning there's
no recorded behavior to violate non-deterministically), the recommended approach should be to simply apply an in-place
update to the workflow type with the fix. The change should **not** be versioned per any of these proposals as it does
not violate any deterministic guarantees in this case.

## Protos Changes
As described in more detail above regarding expected behavior, a new message type should be added for clarity and 
two existing messages should be modified to reflect an optional field for this new type:

```protos
// Reflects the workflow version and patches successfully evaluated for workflow replay purposes
message TargetWorkflowVersion {
  // The name of the specific workflow type executed as part of a previous workflow execution
  string workflowTypeName = 1;
  // The list of patches that evaluated as true during the execution
  repeated string patchNames = 2;
}

message OrchestratorRequest {
    string instanceId = 1;
    google.protobuf.StringValue executionId = 2;
    repeated HistoryEvent pastEvents = 3;
    repeated HistoryEvent newEvents = 4;
    OrchestratorEntityParameters entityParameters = 5;
    bool requiresHistoryStreaming = 6;
    optional TaskRouter router = 7;
    // This should be populated with the version information from a previous `OrchestratorResponse` if this is a replay
    optional TargetWorkflowVersion version = 8;
}

message OrchestratorResponse {
    string instanceId = 1;
    repeated OrchestratorAction actions = 2;
    google.protobuf.StringValue customStatus = 3;
    string completionToken = 4;

    // The number of work item events that were processed by the orchestrator.
    // This field is optional. If not set, the service should assume that the orchestrator processed all events.
    google.protobuf.Int32Value numEventsProcessed = 5;
    // While marked as optional for backwards compatibility, this should always be populated by versioned workflows
    optional TargetWorkflowVersion version = 6;
}
```

## SDK Changes
I share the following suggestions on how this should be implemented in several of the SDKs while emphasizing that it's 
up to the maintainers of each SDK to ultimately decide the best approach to take, so how this is actually done may
differ dramatically. As the maintainer of the .NET SDK and JavaScript SDK, these suggestions represent the best design
I've come up with so far, but again, subject to change upon final release.

Typically, I wouldn't include the low-level details of SDK implementation details in this proposal, but I do so here to make
it clear that the concept _can_ be implmeneted in other languages as imagined in this proposal. 

I'm currently of the mind that a versioning convention should be applied at the global level, but that it can be o
overridden at the workflow level as desired. Different teams may have strictly different ideas about how to version
their code and each SDK should accommodate this while having a default fall-back.

### Proposed .NET Changes
The implementation for .NET can be quite straightforward because of the availability of source generators. I tentatively
propose the following changes across the Dapr .NET repositories and projects.

#### Dapr .NET SDK - `Dapr.Workflows`
The `WorkflowContext` will need to be updated to reflect the `IsPatched` method. This method will accept a string value
that reflects the name of the patch to evaluate. It will return a boolean value according to the following logic applied
in order (copied from above):

A call to `IsPatched`:
- Returns `true` if the workflow is not presently replaying
- Returns `true` if the workflow is replaying and the patch name is present in the version information
- Returns `false` if the workflow is replaying and the patch name is **not** present in the version information

The string associated with each request to `IsPatched` that returns true should be recorded in a list available
on the `WorkflowContext` so it might be read and included in the `OrchestratorResponse` message communicated back to
the Dapr runtime when the orchestration has completed.

#### Dapr .NET SDK - `Dapr.Workflows.Versioning`
This functionality will be built into a new `Dapr.Workflows.Versioning` NuGet package because it requires the use
of a source generator so the code can be generated at compile time instead of using reflection. This ensures that
developers are explicitly opting into the use of this feature and that the SDK is not doing anything that might
cause unexpected behavior. Further, it'll give us a sense of how many people are using the feature based on NuGet package
downloads.

During dependency injection registration at startup, when the developer opts into using Dapr Workflows, add an opt-in
method that allows the user to enable versioning at all. This might look like the following:

```cs
builder.Services.AddDaprWorkflow(opt => {
  // This is the new method that opts the user into versioning and declares the convention to use
  opt.WithVersioning(opt => opt.Strategy = DaprWorkflowVersioningStrategy.NumericalSuffix);
  
  // New alternate workflow registration (if the base type isn't available - this allows the canonical name to be defined for a given type)
  options.RegisterVersionedWorkflow(nameof(MyWorkflow2), "MyWorkflow"); 

  // Existing registration functionality
  opt.RegisterWorkflow<MyWorkflow>();
  opt.RegisterActivity<MyActivity>();
  opt.RegisterActivity<MyOtherActivity>();
});
```

An overload should exist on `WithVersioning` that accepts a collection of types that opt-out of versioning despite the
otherwise global opt-in. This would allow those teams that for whatever reason do not want to use versioning for a 
workflow to opt all other workflow in and not those that are specified.

The SDK should also introduce an attribute called `VersionedWorkflowAttribute`, though per C# convention, this would
only show up as `VersionedWorkflow`. It should accept a minimum of a string bearing the canonical name of the decorated
workflow and optionally a `DaprWorkflowVersioningStrategy` allowing the default convention to be overridden for this 
and other workflow types bearing the same canonical name. Once globally specified in the DI registration, workflow
versioning is performed on an opt-out basis. 

When not applying any overriding configurations, use of the attribute is not required as .NET has the capability via
source generators to find this type based on the existing DI workflow registrations (to infer the canonical name) 
and infer the subsequent types based on the indicated convention. However, best practices would be to proactively
mark each workflow with this attribute nonetheless for consistency:

```cs
[VersionedWorkflow("MyWorkflow")]
public sealed class MyWorkflow2 : Workflow<string, object?>
{
}
```

When looking to override the versioning convention, the developer could pass an additional argument into the attribute:

```cs
[VersionedWorkflow("MyWorkflow", DaprWorkflowVersioningStrategy.DateSuffix)]
public sealed class MyWorkflow20250501 : Workflow<string, object?>
{
}
```

In a later version of the SDK implementation, there might be some sort of migration scheme whereby the developer can
change versioning strategies on a per-workflow basis over the course of implementing new versions. However, this is 
considered out of scope for this initial release. Rather, it will need to be documented that once a versioning
strategy is selected, it should remain consistently applied, and in .NET, (time willing), we can enforce this with 
analyzers.

#### Dapr Durable Task .NET SDK
The following details how type versioning will work on the internal SDK router.

The workflow SDK will add a new interface called `IWorkflowRegistry` which will provide the properties expected by a 
generated `WorkflowRegistry` class that will serve as a lookup table for which workflow types represent the most recent 
versions of any given type (if any).

```cs
public interface IWorkflowRegistry
{
    public Dictionary<string, List<string>> RegisteredWorkflows { get; }
}
```

The dictionary should be keyed by the canonical name of the workflow type and the value should be a list of the
version names that are available for that type, ordered by most recent first per the configuration convention applied
to each.

A `WorkflowRegistry` partial class will already exist in the SDK so it's can be easily referenced, but the logic to
"set" the `RegisteredWorkflows` property on the class will need to be implemented by the source generator itself. It
will look at the configured versioning strategy in the project's DI registration, recursively locate each workflow
type in the solution, compare each of the suffixes to their configured versioning strategies (either globally or on 
each `VersionedWorkflow` attribute) and then populate the dictionary accordingly.

At runtime, the GRPC Workflow Processor class will receive an `OrchestratorRequest` which may contain an optional 
`versionData` property. If it does, the SDK will parse out the workflow type and realized patches from this string and
this represents a "replay" operation.

If this represents a replay operation After building the runtime state from the event history, the 
`GrpcDurableTaskWorker.Processor` will find the value containing the indicated workflow type (to validate it exists) 
and if not, it will throw an exception to the Dapr runtime indicating an execution failure. If it does find the type, 
the shim factory will be modified to point the inbound request to a new instance of the indicated type rather than the 
canonical type registered in DI. Execution will proceed as normal.

If this does not represent a replay operation, a lookup will be done on the dictionary against the canonical keys to 
retrieve the latest (first element in the list value) workflow type and the shim factory will be modified to point the
inbound request to a new instance of this type rather than the canonical type registered in DI. Execution will proceed
as normal, except when the run concludes, the `OrchestratorResponse` will be modified to include a serialized 
`versionData` value reflecting both the canonical workflow type and the list of patch names for which the workflow
context evaluated as `true` during execution.

#### Dapr .NET Workflow CLI
A new CLI tool should be created that can be installed as part of the application following the approach used by 
Entity Framework Core. This tool would allow the developer to perform administrative operations with the workflows to
simplify the level of effort needed to properly create new typed versions. This tool would be installed as a global
.NET tool on the developer's system and would expect to be focused on the working directory containing the workflow
project.

In its initial release, it might be good to support the following commands with examples of calling them, accepted 
parameters, and expected operational result:

##### `dotnet dapr workflow version add`
Adds a new typed version to the workflow project for the specified canonical workflow type.

Arguments:

| Argument     | Description                                                            |
|--------------|------------------------------------------------------------------------|
| &lt;NAME&gt; | The canonical name of the workflow type to add a new typed version to. |

Options:

| Option                    | Short | Description                                                                                                                      |
|---------------------------|-------|----------------------------------------------------------------------------------------------------------------------------------|
| --output-dir &lt;PATH&gt; | -o    | The directory to use to store archived type versions. Paths are relative to the target project directory. Defaults to "Archive". |
| --clean                   | -c    | Statically attempt to remove all "old code" patches from the previous version as part of the migration.                          | 

##### `dotnet dapr workflow version list`
Lists each of the workflow types available in the project as well as the latest version for each.

Options:

| Options | Short | Description                                                            |
|---------|-------|------------------------------------------------------------------------|
| --all   | -a    | List all versions of each workflow type, including the latest version. |

##### `dotnet dapr workflow version prune`

*This would not be available with the 1.17 release, but perhaps in a later version after coordinating how this might
work with the Dapr CLI, perhaps...*

The tool could query the live Dapr instance to understand which versions are no longer
in active use so it might delete them from the local project as part of a developer-driven cleaning operation.

Options:

| Options                 | Short | Description                                                                                  |
|-------------------------|-------|----------------------------------------------------------------------------------------------|
| --workflow &lt;NAME&gt; | -w    | The name of the workflow type to prune. If not specified, all workflow types will be pruned. |

### Documentation
The SDK and building block documentation would need a few paragraphs and several code examples added to explain and 
demonstrate the new versioning with as much low-level detail as possible. It's very likely that the actual implementation
experience will differ by language based on their tools available (e.g., this will be implemented heavily with source 
generators in .NET and similar capability isn't available in other languages).


## Benefits
In short, I believe that pairing both of these proposals in a more cohesive way will improve the developer experience,
provide a useful utility to migrate and version workflows and improve developer perception of the design of Dapr
because we'll have two versioning approaches that go hand-in-hand instead of potentially introducing two complex 
concepts to maintain into our shared ecosystem.

Nothing about this approach precludes rendering older versions obsolete after the 
[improved workflow visibility RPCs](https://github.com/dapr/proposals/pull/93) are introduced in 1.17. It doesn't limit
what versioning strategies can be implemented by each SDK nor require strictly numerical distinctions and leaves that 
to the discretion of the SDK maintainers.

But fundamentally and critically, this appropriate maintains the deterministic nature of the workflows themselves without
introducing substantial complexity that might limit developer adoption. It doesn't change anything about how the workflows
themselves execute thereby limiting the need for developers to re-educate themselves to a new process. Instead,
it leaves us the broadest door possible to facilitate compatibility with existing Dapr Workflow capabilities and to
accommodate new concepts in the future.

As always, I appreciate your time and consideration.


## FAQ
I wanted to centralize some of the back-and-forth experienced in the other proposal threads and propose a response in 
context of this proposal as all the feedback has been instrumental in crafting this approach.

- Does this have an analogy to the versioning performed in Temporal?

Yes. I would describe the versioning side as a hybrid of Temporal's 
[Worker Versioning](https://docs.temporal.io/production-deployment/worker-deployments/worker-versioning) and their 
name-based versioning.

There's a great write-up from a few years ago [here](https://community.temporal.io/t/workflow-versioning-strategies/6911)
contrasting the various approaches and I want to take a moment to distinguish how this proposal differs from Temporal here.

Adapting some of the bullets from that thread, the type-based versioning in this proposal is:
- Simple: A new type is a new version and CLI tooling can help with this
- Robust: Changes are isolated from one another that limits mistakes
- Flexible: There's no need to contemplate backwards compatibility between workflow types

Further, the patch-based versioning:
- Simple: There's no versioning to patches - it's a simple ask "is this patched" or not and handled as an `if/else` statement
- Distinct: Because names are not used outside of that workflow type, there's no chance of inadvertent name collisions,
especially between multiple teams working on the same thing.
- Limited: Because patches are designed to implement small changes, they are useful in the limited capacity of applying
a small change here or there in a workflow. When that gets messy or unwieldy, it creates a clear inflection point at whic
a workflow type version can be useful to refactor the workflow and clear the slate.

- Are there any other key ways this approach diverges from Temporal's?

Yes, I sort of touched on it above, but the four key downsides to type-name versioning in Temporal are:
1) One must register new workflow types.
2) One must update all callers of the workflow to update the running type.
3) New type versions introduce duplicate code.
4) Named types doesn't actually provide any version migration.

In this proposal, differing slightly from #82, numbers 1 and 2 are not relevant. The base type is used throughout the 
proposed C# implementation as routing is automatically handled by the SDK based on the presence of the version prototype 
(if at all).

Further, to make small changes to the workflow, I introduce a simplified version of patching from #92. Regardless,
the notion of having many types to handle migrations is nothing new to anyone that's worked with SQL migrations before
and I suggest building tooling to handle it in the same way - take the old types and stash them in an archive directory.
They're still discoverable by the program, but out of view of the developer's file explorer tree.

And of course, the whole point of this proposal is to provide version migration, so that addresses number 4.

This approach comprehensively addresses all the downsides of named types in Temporal while providing, what I think,
is a better developer experience across the board.

- Wouldn't I have to use patches if I wanted to keep in-flight workflows running while updating my code definitions?

Nope - this is one of the key changes that separates my proposal in #82 from this one with regards to how the SDK finds 
the versioned types. Again as covered in the last section, in the proposed C# SDK implementation, one needn't
register new workflow types nor re-invoke them to get the benefits of mapping.