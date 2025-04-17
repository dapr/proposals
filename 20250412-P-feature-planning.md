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

## Conclusion
The proposed changes to the Dapr feature planning process aim to spend more time considering and reasoning about new 
features in advance of committing to new releases instead of during. This will also maintainers and contributors to
focus on building out concepts that have already spent being vetted and refined, reducing the risk of rushed, incomplete
implementations that may deviate substantially from the original concept at release.