# Release process (automated)

* Author(s): Mike Nguyen (@mikeee) - hey(at)mike.ee
* State: Draft
* Updated: 23-SEP-2024

## Overview

This is a proposal to define a automated release process for Dapr.

The automation will leverage existing release workflows for each repository
where implemented, with updates to add commit statuses to the context where
tests have passed.

The new release process will be triggered by an issue being created with a
template defining the targets, once the required tests have passed a pull
request with the release notes will be created automatically and subject to
editing will kick off the release process when merged.

All the Dapr repositories are in scope during the release process.

## Background

The release process is currently manual, this includes ensuring tests have
passed as well as kicking off a release. This removes the onus on maintainers on
to dedicate time during the endgame and the release team can perform releases
confidently with a defined workflow by maintainers.

## Expectations

* The release process will be triggered by a standardised issue template in the
  main dapr/dapr repository.
* SDKs will be free to continue with releases where not part of a release run.
* Existing release workflows will be utilised where possible for testing and
releases.
* A daily trigger should be run to ensure that commit statuses are correct and
up to date.
* An automated pull request to the main dapr/dapr repository will be created
once tests are passing. This PR will contain release notes.
* The tagging and release process will be triggered by the merge of the release
  notes PR.

## Implementation Details

### Design

Repositories will have their workflows updated to include add commit statuses on
each commit reference where applicable e.g. release and main branches. The commit
reference is used to validate that it has passed the required tests and suitable
for release. This also reduces the burden of running extraneous jobs at
release time.

#### Commit statuses

Release branches going forwards for each repository should only be defined as
`release-X.YY`

Runs will be triggered daily against all repositories using the latest commit
references for dapr & CLI for the latest release branch and the prior (YY-1).
The main branches will also be tested as not all repositories utilise release
branches.

The context should be set to `release/tests` and have a state of: `error`,
`failure`, `pending` or `success`. The description should contain contextual
information for the run that will be displayed.

#### Trigger - Release Issue Creation

The process will be triggered by the creation of an issue from a template in
`dapr/dapr`.

The release will use an unordered list defining inputs:

```md
# Dapr Release Trigger

* RC: No
* dapr/components-contrib: VERSION
* dapr/dapr: VERSION
* dapr/cli: VERSION
* dapr/dashboard: VERSION
* SDKs:
  * go: VERSION
  * rust: VERSION
  * python: VERSION
  * dotnet: VERSION
  * java: VERSION

```

The version should be specified as MAJOR.MINOR e.g. 1.15
A full semantic version can be provided as an override.

The app will confirm whether there is an existing release, if so will provide
a patch release. If it is a RC release, it will increment the RC version unless
there is no existing RC in which case it will increment the PATCH version
number and append rc.0.

Specifying no/nil version will result in it being excluded from the release
process.

There will be a trigger response to display parsed the parsed markdown contents.

#### GitHub Workflow

The workflow will be split into 3 workflows:

##### 1. Issue trigger

This will follow an issue being opened, it will parse and respond to the issue
with a comment providing the parsed values/validity and provide a table of test
results.

If the issue cannot be parsed, it will be closed.

##### 2. Tests passing (repeated)

A workflow will be run every 5 minutes for 5 hours (the maximum run time is
6 hours) until tests are passing or the deadline is reached.

The issue comment will be maintained and updated after every job run.

Once defined tests are passing, a PR will be raised with the release notes for
editing.

If the deadline is reached, the issue will be closed and locked.

##### 3. PR Merge

The release team will propose changes to the PR for release notes if necessary
and subsequently approve and merge.

This will trigger the tag and release process in this order:

1. dapr/components-contrib
2. dapr/dapr
3. dapr/dashboard
4. dapr/cli
5. dapr/installer-bundle
6. SDKs
7. Operators / Shared / Taps etc.
8. dapr/quickstarts
9. dapr/samples
10. dapr/test-infra
11. dapr/docs
12. dapr/blog

The status of the release will be reported on the issue and PR.

## Completion Checklist

* [ ] Workflows implemented
  * [ ] Issue trigger
  * [ ] Tests passing
  * [ ] PR Merge
* [ ] Repository updates
  * [ ] Release workflows confirmed/created
  * [ ] Commit status jobs/steps added to test workflows
* [ ] Tests (Release process)
  * [ ] Unit
  * [ ] E2E
* [ ] Documentation
