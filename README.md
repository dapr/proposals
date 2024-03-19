# Dapr Proposals

## Introduction

This repository stores proposals and designs for new features in Dapr (i.e not bug fixes or minor changes) with the intention of improving visibility, historical record-keeping and maintaining a consistent process.

### What types of changes warrant a proposal here?

As mentioned above, any significant change that needs design and a conversation around that design should go here. As a guideline, anything that would warrant a change in the Dapr SDKs would probably require a proposal. Some specific examples would include:

* New Dapr building blocks
* New APIs or breaking API changes (especially to a non-alpha component)

## How do I create a proposal?

1. Create a fork of this repository.
2. Copy the proposal template [templates/proposal.md](templates/proposal.md) following the format outlined below.
3. Edit the template, filling it in with the proposal (for guides, see information in `guides/`)
5. Submit a PR to `dapr/proposals` for community review.

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

Features in Dapr have a lifecycle (e.g [Components](https://docs.dapr.io/operations/components/certification-lifecycle/)) and, as such, should have a defined set of milestones / requirements for progression between the lifecycle phases. For example, can a user expect from a feature when it is Alpha quality? Once that is released, what is the plan to progress from Alpha to Beta, and the subsequent expectations? What is the expectation when this feature becomes Stable? It is important to identify what functionality or perfomance guarantees we are making to users of Dapr when adding something new.

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

