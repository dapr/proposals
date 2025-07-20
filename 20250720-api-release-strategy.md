# Evolving Dapr's API and Release Strategy
Author: @whitwaldo

## Overview
This proposal introduces a new approach to how Dapr defines, evolves, and releases its APIs and runtime features. It 
recommends moving away from a rigid, 
[calendar-based release cycle](https://github.com/dapr/community/blob/master/release-process.md#release-cycle-and-cadence)
in favor of a feature-driven release model and adopting semantic versioning for all APIs. These changes aim to improve 
the quality and predictability of releases, reduce pressure on contributors, and enable better coordination across the 
runtime, SDKs, components, and documentation.

By decoupling release timing from the calendar and aligning SDKs with explicitly versioned APIs, this model fosters
a more sustainable development process. It empowers contributors to focus on delivering well-designed, well-reviewed 
features — released when they’re truly ready — while improving the developer experience and trust in the platform.

## Background
Dapr currently follows a quarterly release cadence, with each release bundling new features, bug fixes, and updates
across the runtime, SDKs, and components. APIs are labeled as `alpha`, `beta`, or `stable`, with the assumption that 
`stable` APIs are locked in shape but may receive additive changes.

In practice, this model has led to several challenges:
- **Misaligned expectations**: The term "stable" is often misunderstood as a signal of feature maturity or reliability,
rather than API immutability.
- **Late-stage changes**: Features are frequently added late in the release cycle, bypassing the proposal process and
leaving SDKs will little time to adapt.
- **SDK lag and inconsistency**: The SDKs can struggle to keep up with the runtime changes, especially when APIs are
still developing late in the release cycle.
- **Release drift**: The project has consistently missed its self-imposed 3-month release targets, sometimes by multiple
months, creating stress and undermining confidence in the process.

| Version | Release Date | Drift from Last Release |
|---------|--------------|-------------------------|
| 1.10    | 2/16/23      | -                       |
| 1.11    | 6/12/23      | +1 month                |
| 1.12    | 10/11/23     | +1 month                |
| 1.13 | 3/5/24 | +2 months               |
| 1.14 | 8/14/24 | +2 months |
| 1.15 | 2/27/25 | +3 months |
| 1.16 | Jul/Aug 2025 | +2-3 months |

These issues have made it harder for contributors to collaborate effectively and for developers to trust that new features
are ready for production use. This proposal addresses those concerns by introducing a more deliberate, versioned, and
decoupled release model - one that supports both innovation and stability.

## Proposed Changes
### 1. Adopt Feature-Driven Releases
- Replace the current calendar-based release cycle with a feature-driven release model.
- Under this new model, a new minor version of the runtime and components is published only when a feature is complete - 
meaning it has gone through the full lifecycle of proposal authorship, design, implementation, testing, and documentation.
- Bug fixes and security patches will continue to be released as patch versions from the most recent minor release.

#### Why?
- This will reduce pressure on contributors to rush features into a release window, which can lead to incomplete 
designs, insufficient testing, or skipped documentation.
- It improves quality by allowing each feature to take exactly as much time as it needs - no more, no less.
- It encourages thoughtful planning and prioritization, since features are released when ready, not when the calendar
demands it.
- This aligns with real-world development: In practice, many features already miss the intended release window and are
added late in the cycle, often without a comprehensive review. This change formalizes a more sustainable and realistic
approach.

### 2. Use Semantic Versioning for APIs
- Eliminate the current `alpha`, `beta`, and `stable` labels for APIs and instead adopt semantic versioning for each
building block's API.
- Each API will have its own version number, independent of the runtime version:
  - **Major version**: Introduces breaking changes to existing endpoints
  - **Minor version**: Adds new, backward-compatible functionality
  - **Patch version**: Not applicable to APIs (reserved for runtime/component bug fixes or SDK improvements)

#### Why?
- Clarifies expectations because developers can immediately understand the scope of the changes by looking at the 
version number.
- Improves communication as it resolves ambiguity around what "stable" means and replaces it with a well-understood
industry standard.
- It supports incremental evolution because APIs can grow and improve without needing to wait for a major runtime release.
- Versioned APIs make it easier to generate accurate SDK, documentation, and compatibility matrices enabling better tooling
and documentation.
- It prevents stagnation and avoids the trap of APIs becoming "frozen in time" once marked stable. Semantic versioning
allows for continued evolution while maintaining backward compatibility guarantees, encouraging innovation without fear
of breaking existing users.

### 3. Independently Version HTTP and gRPC APIs
- Allow the HTTP and gRPC interfaces for each building block to be versioned independently.
- While we still have a goal of functional parity between teh two, this change acknowledges that certain features may
be more naturally suited to one protocol over the other.

#### Why?
- **Supports protocol-specific innovation**: Some capabilities may be easier or more efficient to implement in gRPC than
HTTP (or vice versa), and this model allows for that flexibility.
- **Avoids unnecessary coupling**:  Changes to one protocol's API don't need to block or delay progress on the other.
- ** Minimal disruption**: Since most SDKs already prefer to target the gRPC APIs - using HTTP only as a fallback - this
change does not require significant rework. It simply formalizes a pattern that is already widely followed.

### 4. Runtime Support for API Versioning
- The Dapr runtime should support multiple versions of each building block's API simultaneously - ideally, the latest
version and at least the two prior minor versions (i.e., **N**, **N-1**, **N-2**).
- This versioning should apply independently to each building block and protocol (gRPC and HTTP), allowing them
to evolve at their own pace without forcing synchronized upgrades across the entire system.
- When a new API version is introduced, the runtime should:
  - Continue to serve older versions for backward compatibility
  - Route requests based on the version specified in the request (e.g. via header or query string or metadata for gRPC).
  - Clearly document which versions are supported and when they are scheduled for deprecation or removal.

#### Why?
- **Predictable Compatibility**: Developers can confidently build against a specific API version, knowing it will be supported
for a defined period
- **Decoupled Evolution**: Building blocks can evolve independently, reducing the need for large, coordinate changes 
across the runtime and SDKs
- **Improved Developer Experience**: Developers can adopt new features at their own pace, without being forced to upgrade
to the latest runtime or SDK version immediately
- **Foudation for Modular Runtime**: This model lays the groundwork for a future where the runtime itself could adopt a
more modular architecture - potentially versioning and releasing building blocks independently, just as some SDKs do.

This versioning strategy ensures that the runtime becomes a stable, flexible platform for innovation - one that supports
both rapid iteration and long-term reliability.

### 5. SDKs Target Specific API Versions
- SDKs should explicitly target and implement only released versions of each building block's API.
- Each SDK should document which versions of each API it supports, and ideally stay within one minor version of the latest
available API version.
- SDKs can be versioned independently of the Dapr runtime, reflecting their own release cadence and the specific APIs
they support.

#### Why?
- **Improves reliability**: Developers using SDKs can trust that the functionality exposed is stable, supported, and 
well-documented.
- **Decouples SDK and runtime timelines**: There is no longer any pressure to ship updates in lockstep with runtime 
releases, especially when APIs are still evolving.
- **Supports modular packaging**: SDKs can adopt building-block-specific packages (e.g. how .NET has Dapr.Workflows,
Dapr.Actors, Dapr.Jobs, etc.) that align with the versioning of the APIs they wrap. This allows for more granular
updates, and better dependency management.
- **Enables preview isolation**: SDKs may optionally offer preview packages for unreleased APIs (e.g. version 0.x APIs),
clearly signaling their experimental nature and avoiding confusing with stable releases.
- **Aligns with real-world usage**: Many SDKs already operate semi-independently of the runtime version (e.g., the 
- JavaScript SDK uses a different major versioning scheme). This proposal formalizes and supports this flexibility.

## Impact on Runtime
- **Asynchronous Development**: The runtime team is no longer constrained by SDK timelines or calendar deadlines. Features
can be developed, reviewed, and released when they are ready - not when the calendar says they must be.
- **Proposal-First Culture**: With versioned APIs and feature-driven releases, every change - large or small - must go 
through the full proposal lifecycle. This ensures that all features receive appropriate design scrutiny, documentation,
and testing before release.
- **Simplified Compatibility Model**: Supporting multiple API versions (e.g N, N-1, N-2) allows the runtime to evolve
without breaking existing integrations. This makes it easier to introduce new capabilities while maintaining trust with 
users.
- **Foundation for Modularity**: Just as SDKs can benefit from modular packaging and independent versioning, this model
lays the groundwork for the runtime to eventually do the same - enabling more targeted updates and clearer ownership
 boundaries across building blocks.
- **Improved Developer Experience**: Developers interacting with the runtime will have a clearer understanding of what's
supported, what's changing, and how to adopt new features - all without surprises or regressions.
  - Moving away from calendar-based releases also builds developer trust and confidence that a feature is truly ready
  when it ships, rather than being rushed to meet a deadline.
  - It avoids the negative optics and morale impact of repeatedly missing scheduled release windows, which can erode
  confidence in the release process and the stability of new features.
- **Enhanced Communication Opportunities**: With more frequent, feature-driven releases, the project gains flexibility
in how it communicates progress:
  - Major features can be highlighted in dedicated blog posts or announcements, given them the spotlight they deserve
  - The community team could maintain a quarterly recap of all changes across the runtime, components, SDKs, and 
  documentation - regardless of release timing.
  - Either approach should continue to recognize and celebrate contributors to each feature, which may require new tooling
  to better track contributions across modular releases - but offers a valuable opportunity to strengthen community 
  engagement and visibility.

## Impact on SDK Maintainers
- **Clear Implementation Boundaries**: SDKs will only implement features from officially released API versions, eliminating
the ambiguity and churn caused by evolving or unstable APIs.
- **Independent Release Cadence**: SDKs can release updates on their own schedules, aligned with the APIs they support - not
the runtime's minor version. This is especially beneficial for SDKs that already use independent versioning schemes.
- **Support for Modular Packaging**: SDKs can adopt building-block-specific packages (e.g. Dapr.Workflows,
Dapr.Actors, Dapr.Cryptography), each versioned according to the API it supports. This enables more focused development,
testing, and release workflows.
- **Improved Documentation and Transparency**: By clearly documenting with API versions are supported, SDKs can provide
a more predictable and trustworthy experience for developers.
- **Optional Preview Support*: SDKs may choose to support unreleased APIs via preview packages (e.g. for v0.x APIs),but
this is opt-in and clearly separated from stable releases - reducing risk and confusion.

## Next Steps
To ensure a smooth transition and avoid immediate versioning confusion across the ecosystem, the following steps are 
recommended upon acceptance of this proposal:

1. Baseline all existing APIs at v2.0
All currently released APIs should be versioned as **v2.0** to establish a clean starting point for semantic versioning.
This ensures that SDKs - particularly those that currently align their versioning with the Dapr runtime - are not 
immediately out of sync and can adopt the new model without disruption.
2. Update SDKs to reflect API versioning
SDK maintainers should begin updating their packages to reflect the specific API versions they support. This may include:
- Updating documentation to indicate supported API versions
- Refactoring SDKs into modular packages (where applicable) to align with building block versioning
- Planning for preview/experimental packages for unreleased APIs (e.g. v0.x)
- Updating build and release pipelines to support independent versioning and publishing of modular SDK packages, enabling
more granular and maintainable release workflows.
3. Update Runtime to Support API Version Routing
The runtime should begin implementing support for versioned API routing and compatibility (e.g. N, N-1, N-2 support and 
identifying how the version should be communicated, e.g. query string or header for HTTP, metadata for gRPC).
4. Move quickly ahead of 1.17 planning
This proposal should ideally be accepted prior to the 1.17 release planning cycle, so that the new model can be piloted
immediately rather than continuing under the current calendar-based approach. Once accepted:
- The mechanics of API version signaling (e.g. whether the version if passed via query string or header for HTTP, 
confirmation that use of metadata in gRPC) should be finalized and documented.
- Each existing API should be formally released and documented as version 2.0, establishing a clear baseline for 
semantic versioning going forward.
- During this transition period -  and indefinitely going forward until v2.0 is deprecated under the project's backwards-
compatibility policy (e.g. N, N-1, N-2) - if a client does not explicitly specify an API version, the runtime should 
assume the request targets v2.0. This ensures a smooth migration path for existing users and provides a stable default
for new integrations.

## Conclusion
This proposal is shared with the goal of empowering Dapr maintainers and contributors to collaborate more effectively,
reduce unnecessary stress, and deliver higher-quality features. By moving away from rigid, calendar-based release cycles
and embracing a feature-driven, semantically versioned model, we create space for thoughtful design, thorough testing,
and meaningful documentation - all without the pressure of arbitrary deadlines.

These changes are not about slowing down progress, but about ensuring that progress is sustainable, deliberate, and inclusive
of the entire ecosystem. When features are released only when they're truly ready - and when ever change receives the
attention it deserves - we build trust with our users, reduce technical debt, and make it easier for all contributors
to work in sync.

Ultimately, this proposal is about enabling the Dapr project to scale with clarity and confidence, while continuing to
deliver powerful, reliable capabilities to the developers who depend on it.