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

Workflows and activities will be registered with the Dapr Workflow SDK as they are today.

Whenever a request come into the application from the Dapr runtime to run work on a specific workflow following a 
`ScheduleNewWorkflowAsync` call, during the same following a `ContinueAsNew` call or even following a 
`CallChildWorkflowAsync`, a request should come into the SDKs to run that named workflow.

### Version isn't specified on orchestration start event
If there is no version specified on the event, the SDK should map the request based on the named base type to the
latest version available in the application and run the workflow there. Every time a patch is evaluated, the "new code" 
route should be taken, and the patch identifier persisted in a list of encountered patches.

At each boundary invocation except workflow orchestration completion, the SDK should return the workflow status with
the (potentially updated) version string defined below containing the name of the workflow type executed and the list 
of named patches followed.

### Version is specified on orchestration start event
If the version is supplied, the workflow type should be parsed out and the SDK should ignore "latest" evaluations and use
that type to run the workflow execution. The list of patches should be extracted from the version string. At each
evaluation of the patch, the path should be taken that corresponds to whether the patch exists in the list:

- If the patch exists in the version list, the "new code" route should be taken
- If the patch does not exist in the version list, the "old code" route should be taken

#### Version string format
The version string should use double pipe characters to separate the different parts of the value. It should always start
with the name of the workflow type, followed by a mandatory '||' and then a list of each case-sensitive named patch applied 
and used in the execution surrounded with square brackets and themselves separated by double pipe characters. The brackets 
should always be present whether there are any patches named within them or not. An example without any patches might 
look like:

```text
MyWorkflowType4||[]
```

An example with a single patch might instead read like:
```text
MyWorkflowType4||[patch-1]
```

And an example with multiple patches might read like:
```text
MyWorkflowTypes4||[patch-1||sample||bugfix2]
```

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
        
        // Alternate workflow registration (if the base type isn't available - this allows the name prefix to be defined for the type)
        options.RegisterVersionedWorkflow(nameof(MyWorkflow2), "MyWorkflow"); 
        
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


#### CLI Tooling
I propose the implementation of CLI tooling to simplify the management of new type versions. For C# developers, I'd propose
this be similar to EF Core's migration tooling:

```bash
dotnet dapr workflow version add
```

There are certainly variations of this that might be used such as specifying the relative archive directory:
```bash
dotnet dapr workflow version add --archive-dir "./Workflows/MyWorkflow/Archive/"
```

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
- Returns `true` if the workflow is replaying and the patch name is present in the version string
- Returns `false` if the workflow is replaying and the patch name is **not** present in the version string

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

#### Patch Naming Constraints
There are a few key constraints that should be applied to patch names:
- Patch names must not include pipe or square bracket characters
- Patch names will be evaluated in a case-sensitive manner

## Caveats
Determinism is still mandatory. Neither #82, #92 nor this proposal proffer a fix for a scenario in which there's a bug
in the workflow that's currently wreaking havoc on the system and needs to be fixed. In the sole case that there's an
exception consistently throwing on a workflow such that history has not been preserved beyond that point (meaning there's
no recorded behavior to violate non-deterministically), the recommended approach should be to simply apply an in-place
update to the workflow type with the fix. The change should **not** be versioned per any of these proposals as it does
not violate any deterministic guarantees in this case.

## What needs to change?
### Protos Changes
Only two changes are needed from the runtime to support Workflow Versioning:
1) An optional string value named `version` containing the version string defined above should be added to the workflow
   history
2) This string value should be returned as a `version` property on any inbound request to the Workflow SDK, if the version
   is present in the history. If it is, the last-seen version of the `version` property should be used.

### Durable Task SDKs (per language)
The runtime needn't know anything about whether versioning has been enabled on a given app as it simply needs to know
the name of the workflow type to invoke. Everything downstream of the runtime calling the SDK to invoke this workflow
is an implementation detail of the SDK. The SDK will be responsible for parsing the version string whether it's present 
or not and routing to the appropriate workflow type, and making the patch names available for the addition of the 
`IsPatched` method on the `WorkflowContext`.

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
it leaves us the broadest door possible to faciliate compatibility with existing Dapr Workflow capabilities and to
accommodate new concepts in the future.

I appreciate your time and consideration.