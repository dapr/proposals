# Modify Release Process
- Author: Whit Waldo (@whitwaldo)
- Status: Proposed
- Introduced: 9/23/2025

## Overview
Dapr's [release milestone](https://github.com/dapr/community/blob/master/release-process.md#release-milestone) today 
calls for three phases in a 13-week (3-month) period:
- Feature definition (~2 weeks)
- Feature implementation and bug fixing (~9 weeks)
  - Code freeze happens ~7 weeks into this implementation phase
- Stabilization & release (~2 weeks)

Today, the prototypes for any given feature are released only at the end of the feature implementation cycle. If the goal
is to then release the new version of Dapr after a 2-week stabilization period, the runtime team then has two weeks to::
- Write the runtime documentation describing the feature
- Stabilize the functionality and prepare for release.

But suddenly things get really busy for the SDK maintainers (and I do mean the maintainers: there is not enough time
for the average contributor to help) to:
- Update the SDKs to accommodate changes to existing prototypes (even without adding new features)
- Design and add new functionality (e.g., what should the SDK API look like to effectively implement the new feature?)
- Write SDK feature documentation
- Develop examples of using the new feature with the SDK
- Write and update quickstarts that feature using the new SDK functionality

So where the runtime team has 13 weeks to iterate on new functionality, this becomes a real time-crunch for the SDK
maintainers to accommodate in only 2 to 4 weeks. It's even worse when you consider that there are often multiple features
in each Dapr release and there is often only one maintainer (and precious few active contributors) for each of the SDKs.
All of these release features often come out at once and put a significant strain on the SDK maintainers to 
accommodate to the high bar of quality and stability they would otherwise strive for.

## Proposal
I propose a shift in the feature design process to be more aligned with common software development practices. Following
feature definition, the feature authors should produce a design document to GitHub that describes the feature and the 
prototypes that comprise its API (changes to existing prototypes and new definitions). Following any discussion by other
runtime and SDK maintainers to clarify the approach, it should be generally approved by relevant parties and locked
for the remainder of the release (e.g., a "protos freeze").

At this point, work can proceed asynchronously across many different parties:
- Each SDK has plenty of time to design, implement and test the feature, and add examples and documentation
- Global documentation can be developed and iterated on that describe the feature, its purpose, what it solves and how 
it works alongside other capabilities

And finally, at the end of the release cycle, while we're all working on stabilization and producing release candidate
builds, the SDKs can also release RCs, the quickstarts can be updated to target each RC and we can all have a far more
relaxing end to the release.

I propose that this design document be either a part of the original proposal in `dapr/proposals` or should be in a 
separate directory within `dapr/proposals` so all the original thinking around feature design can be readily retained 
in one place.

## Final Thoughts
It's critical that when the prototypes are frozen, they're truly locked. The rest of the Dapr project will be relying 
on the shape of the prototypes not to change any more in this release. Should additional changes be needed (as they 
regularly will), they should be done in a separate and future release.

I think this would contribute significantly to a smoother and more timely release cycle for all maintainers and
contributors of Dapr as a whole. 