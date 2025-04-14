# Improving the Dapr Feature Planning

* Author(s): Whit Waldo (@whitwaldo)
* State: Proposed
* Date: 4/12/2025

## Overview
This proposal aims to formalize a new approach to planning releases, enhancing feature eligibility criteria, and
bringing transparency, clarity and definitiveness to the current process.

### Current Process
After releasing a new Dapr version, maintainers and contributors meet weekly to discuss high-level goals for the
next release. We review what happened in the previous release interval, identify improvement opportunities (process,
tooling, new capabilities) and draft proposals reflecting these discussions over the following weeks.

These proposals are debated and iterated upon until consensus is reached. Developers and assigned tasks,
POCs are built, tests are added, documentation is produced, SDKs are updated, and features may be highlighted in a
community call before being released.

### Shortcomings
The current process has several shortcomings:
- **Pending Proposals:** Many proposals in dapr/proposals have been iterated on for months or years by numerous 
contributors but remain inactive due to lack of advocacy. Despite broad community interest across a long period, these
proposals are often overlooked.
- **New Proposals:** In contrast, new proposals benefit from only a week of feedback from a limited number
of contributors. This rushed process often results in features that may not align with larger project goals or
provide lasting positive impact.
- **Artificial Urgency:** Setting tentative release dates upfront and selecting priorities before designing them creates
unnecessary urgency. This rush to consensus risks half-baked ideas being implemented and potentially impact
credibility around release timelines as issues that might have been caught during the design process are identified
and iterated on on-the-fly in a rush to get the feature completed.

The current release cycle structure is ineffective as it forces us to race potentially incomplete ideas out the door.

## Proposal
I would like to propose several changes to our release process starting immediately.

### Develop Proposals Between Releases
We launched v1.15 on February 26th. Each of the current goals of this release span proposals introduced largely around
and since then:

| Proposal | Introduced At                                     | Proposed By |
| -- |---------------------------------------------------| -- |
|Actors: Bi-directional streaming | February 17                                       | Josh van Leeuwen |
| Actors: PubSub | February 26, alternative proposal raised March 25 | Josh van Leeuwen, alternative by Whit Waldo |
| Multi-App Workflows | March 24                                          | Josh van Leeuwen |
| Event Streaming | March 24                                          | Cassie Coyle |
| Workflow Re-runs | March 25, alternative proposal raised April 6     | Whit Waldo, alternative by Josh van Leeuwen |

Despite being raised no more than a couple of weeks ago, many of these proposals have already gained significant traction. 
However, implementing well-designed flagship features based on a three-week ideation and design timeline is unrealistic
and isn't the process taken with larger software groups. It shouldn't be done here either. Furthermost, most of these
issues have minimal community presence outside of the most-active contributors themselves.

#### Proposed Changes
Now is the time for brainstorming big-picture ideas when contributors take breaks from coding or casual discussions
spark new inspiration.

Discord frequently features requests for smaller features that could collectively form larger improvements both in terms
of quality of life and feature additions. Even if these authors don't write original proposals themselves, we should document
these ideas (in a place the stalebot won't hide them) and solicit community feedback to gauge traction. Regular community
and meteor calls should highlight these new ideas with balanced commentary from hosts and direct viewers to relevant
issues so they can chime in themselves.

Designing large-scale features without time pressure soliciting feedback from less-invested contributors can help ensure
that new capabilities are well-thought-out and either fit into our larger vision or require further refinement.

### Release Prioritization
Currently, we ideate high-level goals on maintainer and design calls following a release and wait for proposals to be
submitted, and then largely just move forward with them.

#### Proposed Changes
I propose a substantial change to our process. Barring hotfixes for intermediate patch releases, only ideas backed by an
existing proposal (at least two months old) should be considered for a new release.

We aim for four major releases a year, though we're more on track for three (one release every four months). By
requiring proposals to exist for at least two months, they will have been considered for at least half of the previous 
release cycle, especially during the final stages of the release when unexpected issues tend to crop up while trying
to get the release completed.

This longer proposal period means release prioritization meetings can focus on highlighting eligible proposals and 
providing a call-to-action for final comments and votes on the PRs. If there are too many accepted proposals, 
prioritization can narrowed to those proposals where contributors wish to be assigned and developers can 
immediately move forward with the implementation instead of only then thinking about the design.


### Proposal Acceptance
Currently, features are decided using the 
[following approach](https://github.com/dapr/proposals/blob/main/README.md#proposal-acceptance):
> To accept a proposal, the maintainers of the relevant repository must vote using comments on the respective PR. 
> A proposal is accepted by a majority vote supporting the proposal. When this condition is true, a maintainer from 
> the relevant repository may approve and merge the proposal PR. While everyone is encouraged to participate and 
> drive feedback, only the maintainers of the relevant repository have binding votes. Maintainers of other 
> repositories and community contributors can cast non-binding votes to show support. The majority vote needed is 
> a simple majority (more than 50% of total votes).

[This](https://github.com/orgs/dapr/teams/maintainers-dapr) is the list of people who currently have binding votes 
on runtime features:

| Name | Organization |
| -- | -- |
| Josh van Leeuwen | Diagrid |
| Yaron Schneider | Diagrid |
| Cassie Coyle | Diagrid |
| Loong Dai | Intel |

Dapr is a core runtime surrounded by a collection of language-specific implementations of it. There's no real proposal 
process for the language SDKs as that's left to the maintainers (who are most likely the most active contributors as
well), meaning that most of this proposal language is relevant only to the core runtime. 

More to the point, there will almost _never_ be a circumstance where some proposal is made to one of the SDKs that
would ever propagate into the runtime because it wouldn't work without runtime support. All new features must be built
out in the runtime first, making the "relevant repository" language unnecessarily restrictive.

By allowing only binding votes by the few maintainers of the runtime, this minimizes impact of contributions of the 
larger collection of [other maintainers](https://github.com/dapr/community/blob/master/MAINTAINERS.md) and their 
relevant prioritization interests:

| Name                | Organization
|---------------------| -- |
| Artur Souza         | Amazon? |
| Bernd Verst         | Microsoft |
| Cassie Coyle        | Diagrid |
| Cecil Phillip       | Stripe |
| Dana Arsovska       | Ahold Delhaize |
| Deepanshu Agarwal   | Microsoft |
| Elena Kolevska      | Redis |
| Hannah Hunter       | Microsoft |
| Hal Spang           | Microsoft |
| Josh van Leeuwen    | Diagrid |
| Long Dai            | Intel |
| Marc Duiker         | Diagrid |
| Mark Fussel         | Diagrid |
| Maurucio Salatino   | Diagrid |
| Mukundan Sundarajan | Microsoft |
| Nick Greenfield      | Microsoft |
| Paul Yuknewicz | Microsoft |
| Phillip Hoff | Microsoft |
| Rob Landers | Automattic |
| Whit Waldo | Innovian |
| Yaron Schneider | Diagrid |

Members of the steering group committee have equally weighted vote regardless their personal or organizational use
of Dapr. Non-core maintainers can have an equal passion for Dapr's success and relevance, so they should have a
single binding bote on runtime features as well.

By requiring a better-than-half consensus across the broader group of maintainers, this will lessen the unbalanced 
nature of the existing binding voting pool. Consider the most recent proposals again:

| Proposal | Introduced At                                     | Proposed By |
| -- |---------------------------------------------------| -- |
|Actors: Bi-directional streaming | February 17                                       | Josh van Leeuwen |
| Actors: PubSub | February 26, alternative proposal raised March 25 | Josh van Leeuwen, alternative by Whit Waldo |
| Multi-App Workflows | March 24                                          | Josh van Leeuwen |
| Event Streaming | March 24                                          | Cassie Coyle |
| Workflow Re-runs | March 25, alternative proposal raised April 6     | Whit Waldo, alternative by Josh van Leeuwen |


Most features proposed for 1.16 are introduced by members with binding votes atn an organization comprising
more than half of the votes needed to approve anything. By diluting this voting pool to all maintainers listed above
(even if some may need to be pruned based on activity), new Dapr features must be designed to be acceptable and
relevant to a broader pool than just the core team.

While expanding the core runtime maintainers list itself might be another way to address this, it requires time, 
expertise and a willingness from unknown other parties. Instead, we can change the process itself on a shorter timeline,
and I propose we do so.

## Conclusion
The proposed changes to the Dapr feature planning process aim to address the current shortcomings by fostering a more 
inclusive, transparent, and well-considered approach. By extending the proposal development period, we ensure that 
ideas are thoroughly vetted and refined, reducing the risk of rushed, incomplete implementations that may be effectively
rendered effectively dead on arrival. Additionally, broadening the voting pool to include all maintainers will 
democratize decision-making, ensuring that features not only relevant to a few, but to a broader group of maintainers based
on their personal experiences and the diverse parts of the community they represent. 

These changes will not only enhance the quality and coherence of new features while strengthening the collaborative
spirit of the Dapr project. By embracing a more structured and inclusive process, we can better align our development 
efforts with the varied needs and aspirations of our contributors and users, ultimately driving the project forward 
in a more balanced and sustainable manner.

Let's seize this opportunity to refine our approach and build a more robust, community-driven Dapr that continues 
to innovate and excel.