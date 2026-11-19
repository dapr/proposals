# Dapr Proposals

[Dapr](https://github.com/dapr/dapr) is a portable, event-driven runtime that gives developers APIs for building resilient, distributed applications; see the [Dapr documentation](https://docs.dapr.io) to learn more.

## Introduction

This repository stores proposals and designs for new features in Dapr (i.e not bug fixes or minor changes) with the intention of improving visibility, historical record-keeping and maintaining a consistent process.

### What types of changes warrant a proposal here?

As mentioned above, any significant change that needs design and a conversation around that design should go here. As a guideline, anything that would warrant a change in the Dapr SDKs would probably require a proposal. Some specific examples would include:

* New Dapr building blocks
* New APIs or breaking API changes (especially to a non-alpha component)

## Proposal index

Proposals in the root of this repository, grouped by area. Dates and flags come from each file name (see [Proposal name format](#proposal-name-format)).

### Workflow, actors and scheduler

| Date | Title | Flags | Link |
|------|-------|-------|------|
| Legacy (0012) | Distributed Scheduler Building Block and Service | BIRS | [0012-BIRS-distributed-scheduler.md](0012-BIRS-distributed-scheduler.md) |
| 2024-09-19 | Scheduler Job Failure Policy | None | [20240919-scheduler-failure-policy.md](20240919-scheduler-failure-policy.md) |
| 2025-03-20 | Multi-App Workflows | RS | [20250320-RS-multi-app-workflows.md](20250320-RS-multi-app-workflows.md) |
| 2025-04-04 | Workflow: Rerun from Activity | RS | [20250404-RS-workflow-rerun-from-activity.md](20250404-RS-workflow-rerun-from-activity.md) |
| 2025-04-29 | Scheduler Broadcast Job Type | RS | [20250429-RS-scheduler-broadcast-job-type.md](20250429-RS-scheduler-broadcast-job-type.md) |
| 2025-10-22 | Consolidate Placement into Scheduler, Improve Actors and Workflows Reliability and Performance | S | [20251022-S-placementV2.md](20251022-S-placementV2.md) |
| 2025-10-28 | Workflow Patching and Versioning | BRS | [20251028-BRS-workflow-versioning.md](20251028-BRS-workflow-versioning.md) |
| 2026-03-18 | Workflow External Event Timer Tracking | RS | [20260318-RS-workflow-external-event-timer-tracking.md](20260318-RS-workflow-external-event-timer-tracking.md) |

### Building blocks and APIs

| Date | Title | Flags | Link |
|------|-------|-------|------|
| Legacy (0013) | PubSub Subscription Streaming | RS | [0013-RS-pubsub-subscription-streaming.md](0013-RS-pubsub-subscription-streaming.md) |
| 2023-03-27 | Crypto(graphy) building block | RCBS | [20230327-RCBS-Crypto-building-block.md](20230327-RCBS-Crypto-building-block.md) |
| 2023-04-06 | External Service Invocation | B | [20230406-B-external-service-invocation.md](20230406-B-external-service-invocation.md) |
| 2023-05-11 | Dapr Error Handling/Codes | BCIRS | [20230511-BCIRS-error-handling-codes.md](20230511-BCIRS-error-handling-codes.md) |
| 2024-06-18 | Conversation building block | RCBS | [20240618-RCBS-Conversation-building-block.md](20240618-RCBS-Conversation-building-block.md) |
| 2026-03-04 | MCPServer: A First-Class Dapr Resource for Durable MCP Server Integration | CR | [20260304-CR-mcpserver.md](20260304-CR-mcpserver.md) |

### Runtime, security and resiliency

| Date | Title | Flags | Link |
|------|-------|-------|------|
| 2022-11-21 | Pluggable components injector | R | [20221121-R-pluggable-components-injector.md](20221121-R-pluggable-components-injector.md) |
| 2023-10-24 | Dapr Trust Distribution | CIR | [20231024-CIR-trust-distribution.md](20231024-CIR-trust-distribution.md) |
| 2024-05-17 | HTTP Metrics Path Matching | R | [20240517-R-http-metrics-path-matching.md](20240517-R-http-metrics-path-matching.md) |
| 2024-09-17 | Resiliency Policy Error Code Retries | BR | [20240917-BR-resiliency-error-code-retries.md](20240917-BR-resiliency-error-code-retries.md) |

### SDKs and CLI

| Date | Title | Flags | Link |
|------|-------|-------|------|
| 2022-11-30 | Run multiple applications with Dapr sidecars | I | [20221130-I-enhance-dapr-run-multiple-apps.md](20221130-I-enhance-dapr-run-multiple-apps.md) |
| 2023-07-14 | Resiliency in Dapr SDKs | S | [20230714-S-sdk-resiliency.md](20230714-S-sdk-resiliency.md) |
| 2023-09-18 | Unify the `DAPR_API_TOKEN` env variable across all SDKs | S | [20230918-S-unified-api-token-env-variable.md](20230918-S-unified-api-token-env-variable.md) |
| 2024-05-08 | Dapr endpoint env and TLS support in SDKs | S | [20240508-S-sidecar-endpoint-tls.md](20240508-S-sidecar-endpoint-tls.md) |

### Proposal process

| Date | Title | Flags | Link |
|------|-------|-------|------|
| 2022-10-25 | Improving the Dapr proposal process | P | [20221025-P-proposal-process.md](20221025-P-proposal-process.md) |
| 2023-06-27 | Cross SDK changes approval | P | [20230627-P-proposal-sdk-approval.md](20230627-P-proposal-sdk-approval.md) |

## How do I create a proposal?

1. Create a fork of this repository.
2. Copy the proposal template [templates/proposal.md](templates/proposal.md) following the format outlined below.
3. Edit the template, filling it in with the proposal (for guides, see information in `guides/`)
4. Submit a PR to `dapr/proposals` for community review.

## Proposal name format

Proposal file are named in the following format:

> `YYYYMMDD-FLAGS-description.md`

Where *YYYY* is a 4-digit year, MM for 2-digit month and DD for 2-digit day of when the proposal was last updated (like `20240309`, for example), and *FLAGS* is one (or possibly more)  of:

* B - Building block change / creation
* C - Components change / creation
* I - Affects Dapr CLI
* P - The proposal Process itself
* R - Runtime
* S - Affects SDKs

So, for example, a proposal to create a new building block, such as the workflow building block, might be something like `20240102-BRS-workflow-building-block.md`, whereas a change to the actor system, which does not require any changes to the SDKs themselves, would be something like `20240103-R-actor-reminder-system.md`

A few older proposals predate this format: `0012-BIRS-distributed-scheduler.md` and `0013-RS-pubsub-subscription-streaming.md` use legacy sequential numbering instead of a date, and `20240919-scheduler-failure-policy.md` has no flags. New proposals should use the format above.

## Proposal Process

* The proposal will be opened as a PR against this repository
* Proposal will be reviewed by the community and the author(s) of the proposal
* The author(s) address questions/comments from the community in the proposal and adjust the proposal based on feedback
* Once the feedback phase is complete, and a proposal has been accepted, the proposal will be merged into this repository
* An issue needs to be created in dapr/dapr created from the template in [templates/lifecycle.md](templates/lifecycle.md) to track the work that needs to be done to implement this proposal
* Release of the feature will be slated for a specific release version of Dapr

### Proposal acceptance

To accept a proposal, the maintainers of the relevant repository must vote using comments on the respective PR. A proposal is accepted by a majority vote supporting the proposal. When this condition is true, a maintainer from the relevant repository may approve and merge the proposal PR. While everyone is encouraged to participate and drive feedback, only the maintainers of the relevant repository have binding votes. Maintainers of other repositories and community contributors can cast non-binding votes to show support. The majority vote needed is a simple majority (more than 50% of total votes).

## Feature lifecycle outline

Features in Dapr have a lifecycle (e.g [Components](https://docs.dapr.io/operations/components/certification-lifecycle/)) and, as such, should have a defined set of milestones / requirements for progression between the lifecycle phases. For example, can a user expect from a feature when it is Alpha quality? Once that is released, what is the plan to progress from Alpha to Beta, and the subsequent expectations? What is the expectation when this feature becomes Stable? It is important to identify what functionality or performance guarantees we are making to users of Dapr when adding something new.

For example, the lifecycle expectations of a "Foobar API" that is going to replace an existing API might look something like:

Alpha:
 * Initial contract for the Foobar API is complete
 * Performance is expected to be >10TPS
 * Will not support serialization via XML
 * Data stored will not be compatible with old API, existing data will be unavailable through this API (will need to use old API to access old data)
 * Only available when feature flag `foobar-api` is enabled
 * No migration of existing data from the old API available

Beta:
 * Performance meets or exceeds 1,000TPS
 * Enabled by default, users can opt-out via feature flag / configuration
 * Existing APIs marked as deprecated
 * Migration from previous data source / format can be done manually
 * XML will be supported
 * Backwards-incompatible changes may be made


Stable:
 * Enabled by default, existing APIs have been removed fully
 * Documentation has been changed to remove previous API definitions
 * Migration from previous data source / format will be done automatically (lazily)
 * API is stable and changes will not be backwards-incompatible



## Proposal Language

This information can be included either in the template or in a README -- and is designed to provide a common language for proposals so that the expectations are clear.


### Terminology

_(This is an incomplete list and should / will be expanded over time)_

| Term	| Meaning                                                                                                                                                     |
|------|-------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Building block	| Capabilities that solve common developmental challenges in building distributed applications                                                                |
| API	| Application Programming Interface - functionality exposed to end-users that can be used to interact with Dapr's building blocks in the application they are building  |
| Feature |	New or enhanced functionality that is being added to Dapr |

### Keywords

The keywords “MUST”, “MUST NOT”, "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY",
and "OPTIONAL" in this document are to be interpreted as described in RFC 2119.

