# Feature-Maintainer role in DAPR

* Author(s): Radhika Bollineni (@rabollin)
* State: Ready for Implementation
* Updated: 2023-12-18

## Overview

This is a process proposal to introduce a new role 'Feature-Maintainer' in DAPR repos.

## Background

Maintaining an open-source software (OSS) project involves a range of responsibilities that go beyond just writing code. Open-source software is typically developed collaboratively, and maintainers play a crucial role in managing various aspects of the project's lifecycle. An "OSS maintainer" refers to an individual or group of individuals who are the technical authority for a project/subproject. They MUST have demonstrated SME( for example : deep understanding of technical domain of the project/subproject, have sustained contributions to the project through design/ implementation and reviews), good judgment and responsibility towards the health of that subproject.

Some of the DAPR repos ( for example:[Runtime](https://github.com/dapr/dapr) ) are challenging for someone in the community to develop the SME for the entire repo, take design/ technological decisions across the repo and become a maintainer. In addition, due to the bandwidth of Core Maintainers of these repos to handle the change volumes, the velocity of PR reviews merge is slowing.

### Proposal

To increase the velocity of PR reviews from maintainers, we introduce the role of a Feature Maintainer for specific larger repos in particular DAPR Runtime and Components Contrib.

### Goal

1. To enable a distributed decision structure and code ownership, as well as providing focused forums for getting work done, making decisions increase the no. of maintainers across the repos.
2. To improve the velocity of Proposal/ PR reviews, timely guidance to community members, facilitating collaborative/ inclusive environment and planning for project’s long-term sustainability.

### Classification of ownership

**DAPR Runtime**:

1. As Dapr is built with API building blocks and control plane services we can segregate Feature Maintainers ownership for Runtime Repo as follows:

* List of building blocks and control plane services:

 | **Subarea**   | **Subarea**                                  |
| ---------- | ---------------------------------------------|
| Control Plane Services – Injector    | Service Invocation    |
| Control Plane Services - Operator| Pub-sub |
 | Control Plane Service – Sentry (include mTLS)|  Bindings|
 | Middleware | Secrets
 | Distributed locks | State management |
 | Cryptography | Actors (includes Reminders & Placement Service) |
 | DAPR deployments | Workflow |
 | Pluggable Components | Configuration
 | Resiliency|  |

2. Though these building blocks are logically separated there are certain inter dependencies with other DAPR APIs and the sub-maintainer ownership includes the interdependent code paths too.

* For example - Workflow will have interdependency with Actors/ statestore/pub-sub; any change in the context of Workflow leading to the change in these dependent APIs falls under the Workflow Sub maintainer ownership. However, any changes/proposals/design that has directly reported with State/ pub-sub and has no relation with Workflow cannot be handled by this Workflow Sub maintainer.

3. The code/ features changes in other common areas don’t fall in any of the specific categories like Utils/ protos will fall under maintainer responsibility.

4. Feature-maintainers should be able to merge the PR of their designated API/Building blocks.

5. If the PR/ feature is spread across multiple building blocks, then the designated Feature maintainers of those building blocks can approve before merging the PR. Core Maintainers can always approve and merge the PRs any time.

6. CoreThe Repo maintainers will always have ultimate veto power to override any decisions as relevant.

7. The code/ features changes in other common areas like Observability/Utils/tools/protos will fall under the repo maintainer responsibility.

**Components-Contrib Repo:**

1. DAPR Components list is so vast spread across multiple domains, platforms and providers, we can segregate Sub maintainers ownership by Company/ Cloud Provider as follows:

| **Subarea**   | **Subarea**                                  |
| ---------- | ---------------------------------------------|
 General | Alibaba Cloud |
 Google Cloud Provider(GCP) | Cloudflare |
 Microsoft Azure | Apache|
 Amazon Web Services(AWS) | Zeebee(Camunda Cloud)
Oracle Cloud | Hashicorp

2. Since these components are clearly segregated, the sub maintainer can get the ownership against cloud/ company providers.  They are responsible for reviews/approve/merge the changes against their sub areas.

3. The Repo maintainers will always have ultimate veto power to override any decisions as relevant.

Maintainership is defined by: GitHub organization ownership, permissions, entry in sub-maintainer.md, a similar file like [MAINTAINERS.md](https://github.com/dapr/community/blob/master/MAINTAINERS.md).

**Note:**

1. We will create feature-maintainer.md indicating the area of ownership clearly. Reference example: [sigs.yaml](https://github.com/kubernetes/community/blob/master/sigs.yaml).  
2. Feature-Maintainer definition will be updated in the below page along with other Community roles explained already. <https://github.com/dapr/community/blob/master/community-membership.md#maintainer>

## Requirements

The following apply to the sub area(ex: building block API/ Control plane service/ Component provider) for which one would be a Feature-maintainer:

* Deep understanding of the technical goals and direction of the sub area.
* Deep understanding of the technical domain (specifically the language) of the sub area
* Sustained contributions to design and direction by doing all of:
  * Authoring and reviewing proposals
  * Initiating, contributing and resolving discussions  (For example Discord discussions,  GitHub issues, meetings)
  * Identifying subtle or complex issues in designs and implementation PRs
* Directly contributed to the subarea through implementation and / or review
* Must have been active for 3 months or more for the given sub-area
* Inactivity for more than 3 months leads to a removal vote by the repo maintainers and not an automatic removal

## Responsibilities and privileges

The following applies to the sub area(ex: building block API/ Control plane service/ Component provider) for which the feature-maintainer would be an owner:

* Make and approve technical design decisions for the sub area
* Set technical direction and priorities for the sub area.
* Define milestones and releases working along with the maintainers
* Decides on when PRs are merged to control the release scope
* Mentor and guide approvers and contributors of the sub area
* Escalate approver and maintainer workflow concerns.  For example responsiveness, availability, and general contributor community health to the STC
* Ensure continued health of sub area:
* Adequate test coverage to confidently release
* Tests are passing reliably (not flaky) and are fixed when they fail.
* Work with other feature-maintainers & Core-maintainers to maintain the project's overall health and success holistically.
* Membership tracked in feature-maintainer.md entry and scoped to a sub area.
* MUST maintain components, review, and approve proposals for enhancing areas falling under the user sub area.
* Actively participate in triaging issues and reviewing PRs
Set milestone priorities for the sub areas or delegate the responsibility to repo maintainers.
Ensure a healthy process for discussion and decision making is in place

## Acceptance

New Feature-Maintainers can be added to the project by a super-majority (two-thirds / 66.66%) vote. Only the maintainers of the repository in which the nomination is applied have a binding vote, while maintainers from other repositories are on an informed basis via a separate email thread.

A potential feature-maintainer may be nominated by an existing maintainer from the repository in which the nomination is applied to. A vote is conducted in private between the current maintainers over the course of a one week voting period. At the end of the week, votes are counted, and a pull request is made on the repo adding the new feature-maintainer to the feature-maintainers.md file.

Feature-maintainers MUST remain active. If they are unresponsive for >3 months, they will be automatically removed unless a super-majority of the other repository maintainers agrees to extend the period to be greater than 3 months.

A feature-maintainer may step down by submitting an issue stating their intent.

Example of Feature Maintainers in DAPR Runtime repo:
| **Subarea**   | **Name**                                  |
| ---------- | ---------------------------------------------|
| Workflow    | Chris Gilum                                 |
| Control Plane Service(mTLS)| Josh V |

### Mandatory follow-up tasks  

Once the proposal for Feature-maintainer as described above is approved then immediately, we should work on the following items:

1. Create the Feature-Maintainer.md file in Community repo which will be updated by Core Maintainers through PR once the  Feature maintainer for a sub-area is confirmed.
2. Ability to restrict the access permissions for  Feature Maintainers to their designation sub-area of the repo; reference: <https://docs.prow.k8s.io/docs/overview/>
3. Bot that can auto-merge the PRs once  it is approved and all necessary checks are passing.
