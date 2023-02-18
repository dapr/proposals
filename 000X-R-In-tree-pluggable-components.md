# Title of proposal 

* Author(s): Alessandro Segala (@ItalyPaleAle)
* State: {Ready for Implementation, Implemented}
* Updated: [Date]

## Definitions

* **In-tree**: the code for the components lives in the official tree, i.e. in the `dapr/components-contrib` repository.

## Overview

This is a proposal for implementing the infrastructure to allow in-tree components to be shipped with Dapr but instantiated as pluggable components by the Dapr runtime.

This is **not** a proposal to remove components from Dapr's distribution or remove them from the Dapr repositories. Components that are instantiated as in-tree pluggable components should work for end-users in the same way that statically-compiled components do.

## Background: business problem

As of Dapr 1.10, we are shipping over 110 components across all of our building blocks, and the number is growing pretty much constantly with each release. This is great and it's the sign of a healthy OSS project with a growing community.

However, this also brings some challenges. Dapr is written in Go, which as a technology incentivizes binaries to be statically compiled and self-contained. As of writing, there's no "plug-in" system for Go that works reliably across all operating systems. This means that the more components Dapr gains, the larger our binary becomes, and the more code is loaded in memory when `daprd` is started.

The problems we've seen include, but are not limited to:

* Amount of data loaded in memory.  
  Dapr 1.10 has a virtual memory of almost 900MB. Most of that is code pages that are not really a burden on the system, but the amount is growing with each release as we ship more code. A lot of that code is indeed due to components.  
  More concerning, however, is when components instantiate memory when `daprd` is started, for example in `init` functions in Go packages. Often this memory is allocated on the heap, which also means that it causes more work for the garbage collector and has a negative impact on `daprd` overall.
* Concerns with running on environments with limited resources.  
  This is connected with the point above. Certain systems such as embedded ones or IoT devices have limited resources, starting from storage and memory, so the increasing footprint of `daprd` is a concern in these scenarios. For example, we are aware of at least one large Dapr users who is re-building Dapr with a very limited subset of components to be able to fit it on embedded devices.
* Effect of security vulnerabilities in third-party packages.  
  Most of our components depend on third-party packages such as SDKs, and we import dozens of them in Dapr. Suppose that component X depended on a library for which a vulnerability was found. Users who do not leverage component X are likely not impacted by the vulnerability, but they are still running a binary that depends on a vulnerable package, possibly causing compliance conerns.
* Ability to ship some components with outsized impact on performance.  
  Already in the last months we've had to abandon components that were accepted in `dapr/components-contrib`, after being reviewed, and then we had to remove them because we noticed they had an unacceptable impact on the resources used by `daprd`. Sadly, our processes and tools don't allow us to catch these things until the component is actually registered in `daprd`.  
  This has a negative impact on contributors, who invest time and energy to write the code of a new component, on maintainers who spend cycles reviewing the contributions, and on users who do not get to enjoy the new features.

## Related Items

### Related proposals

This proposal is related to the "pluggable component" techology that was first implemented with Dapr 1.9, which is still evolving.

We've considered the issues of the large number of components in the past, with proposals such as "cloud-specific builds", for example dapr/dapr#3168. However, those proposals came with a lot of limitations, both for Dapr developers and end-users.

## Expectations and alternatives

* **What is in scope for this proposal?**
  * This proposal is about the underlying technology to support instantiating in-tree components as pluggable.
* **What is deliberately *not* in scope?**
  * This is *not* a proposal about removing components from Dapr so they are not shipped together with Dapr and in the same Docker containers.
  * This proposal is also *not* concerned with *which* components should be shipped as statically compiled or in-tree pluggable. Maintainers are expected to make this decision collegially in separate conversations.
  * This is also *not* a proposal for a technology that enables customers to build images of Dapr with a custom set of components statically compiled or as in-tree pluggable, although this does enable that scenario in the future.
* **What alternatives have been considered, and why do they not solve the problem?**
  * The first alternative is the *status quo*. Dapr continues to grow with the addition of new components.
  * We can also decide to ship multiple "flavors" of Dapr each with a different set of components statically compiled inside, e.g. as per dapr/dapr#3168. This however is more limiting for end users and adds operational complexities for Dapr developers.
  * Removing components from Dapr entirely is an option as well, for example to ship them as external, pluggable components. These would be separate from `daprd` in all ways, including: they're shipped in separate Docker images, need to be explicitly added to applications (see [0005-R-pluggable-components-injector.md](./0005-R-pluggable-components-injector.md)), and can have separate release lifecycles and support.
  * Users can also re-compile Dapr with a custom set of components. Although we don't officially support this, we are aware of at least some users who do this as a practice. This is something only advanced users may consider doing due to the very high complexity and operational burden.
* **Are there any trade-offs being made?**  
  * Components that are instantiated as in-tree pluggable components will likely have *slightly* degraded performance, due to the overhead of inter-process communication and gRPC serialization/unserialization. Based on the research that was done for pluggable components, that should be negligible for most users.
  * The `daprd` Docker image will likely grow in size due to containing more binaries, each one with some overhead.

## Implementation Details

### No changes to components-contrib

No changes will be needed in `dapr/components-contrib` or in the processes follow in that repository. Components in `dapr/components-contrib` continue to live in folders in that repository, and tests (including certification and conformance) continue to be written and run as they are today.

### daprd as a process manager

The first step is to allow `daprd` to become a "process manager" that is capable of instantiating and supervising other processes as needed.

When a user defines a component (in a Component YAML) that uses an in-tree pluggable component, during the initialization step of the component `daprd` will spawn a process that hosts the component as a pluggable component, and connects to it via UNIX Domain Socket (UDS), like any other pluggable component.

Unlike "regular" pluggable components, `daprd` is the parent process that spawns the process hosting the component. `daprd` is also responsible for restarting the process if it crashes and can dispose of it if not needed anymore (e.g. in a future where components are more "dynamic").

### Binaries

Currently, all components are statically-compiled in `daprd` by creating a file in the [`cmd/daprd/components`](https://github.com/dapr/dapr/tree/release-1.10/cmd/daprd/components) package. Components that continues to be statically-compiled will not change.

We will create a new package, for example `pkg/intree_pluggable`, which acts as host for pluggable components. It uses the Dapr pluggable component SDK to create the gRPC server on a socket, and to communicate with the component's code.

Inside a folder such as `cmd/components/`, we will then create a folder for each component that becomes available as in-tree pluggable. In there, there's a `main` package that contains a `main()` function, which uses `pkg/intree_pluggable` and a registration code to create a binary that manages the component.

These packages are compiled independently and generate separate binaries, one per each in-tree pluggable component.

#### Example

As a fictious example, let's look at a state store component that uses the Ethereum blockchain to store state 🙃

The component's code is defined in `dapr/components-contrib` as any other component, for example in the `state/ethereum` package (in components-contrib).

We then create a file in `dapr/dapr` in `cmd/components/state_ethereum/main.go` with content such as:

```go
package main

import (
	"github.com/dapr/components-contrib/state/ethereum"
	pluggable "github.com/dapr/dapr/pkg/intree_pluggable"
)

func main() {
	// Create a new in-tree pluggable component for a state store
	component := pluggable.NewStateStore(logger)

	// Register the component - just like we do in cmd/daprd/components
	component.Register(ethereum.NewEthereumStore)
	// Start the component, this is a blocking call
	component.Run()
}
```

The component is then compiled like any other Go binary, creating a statically-compiled binary `intree_pluggable`.

### Packaging

The last step is packaging. After building all binaries, **they are included in the same container image as `daprd`**. That's important so end-users don't need to do anything different when deploying Dapr than they do today.

## Future expansion

Although not part of this proposal, the goal is to set the foundation for future projects such as being able to flexibly build Dapr images with components embedded statically, shipped as in-tree pluggable, or omitted entirely.

## Completion Checklist

What changes or actions are required to make this proposal complete? Some examples:

* ✅ Decisions around what components to ship as statically-compiled, and which ones as in-tree pluggable
* ✅ Code changes
* ✅ Tests added (E2E)
