# API Design Guidelines

* Authors: Mukundan Sundararajan (@mukundansundar), John Ewart (@johnewart)
* Updated: 10/25/2022

## Proposal requirements


For any API (new or updates), the following must be included in the proposal:

  * Relevant high level design
  * Proposed contract for the API
    * HTTP and gRPC APIs should be consistent in behavior and user experience.
  * Identifying what additions to existing components / creation of new components are required for this API (if any)
  * Scope for current and following releases (i.e what can be expected from this iteration and what is being pushed down the road)
  * Known limitations, where applicable
    * Performance issues
    * Compatibility issues
  * Code examples (pseudocode is acceptable)


## API Lifecycle expectations

APIs are expected to go through three stages in their lifetime: Alpha, Beta and Stable. For each of these phases it should be clear to a user what they can expect. In the case of an API, those expectations are:

* **Alpha**
   * API is not production ready yet and might contain bugs
   * Recommended for only non-business-critical uses because of potential for incompatible changes in subsequent releases
   * May not be backwards-compatible with an API it intends to replace
   * May not be highly performant or support all SDKs
  
* **Beta**
   * API is not production ready yet
   * Multiple components implement the API and API contract is mostly finalized
   * Recommended for non-business-critical use only due to potentially backwards-incompatible changes in subsequent releases
   * Should have support in (at least) the primary SDKs (i.e Python, Go, Java?)
   * Performance should be production-ready but may not be in all cases

* **Stable**
   * API will not undergo backwards-incompatible changes
   * API is considered ready for production usage
   * Performance numbers are published for the API and there are tests and safeguards in place to prevent regression


## Requirements for API changes

No matter if the change is a net-new API or an update to an existing API, the following is required:

* Changes to documentation must be identified and written
* Existing E2E and performance tests must pass
* If a new command/modifications to existing command is required to facilitate ease of use of the new API, related code must be added to the Dapr CLI 

### Creation of new APIs

* Both HTTP and gRPC protocols need to be supported for the new API
* Documentation must be provided for the API 
  * HTTP API must be added to Dapr documentatio
* Issues should be added in `dapr/quickstarts` to create examples for the new API to enable users to easily explore the new functionality provided by the API
* If the new API is considered an _optimization_ of an existing API (say, the addition of `BulkGetSecrets` alongside `GetSecret`) then:
  * The performance improvement gained due to this API should be documented
  * Guidance must be provided to the users in docs as to when to use this API vs using the older one
* Performance tests should (though preferably must) exist for this new API
* _Must_ include new E2E tests that exercise the API


### Updates to existing APIs

Depending on the phase of the existing API, the proposed changes may or may not be backwards-incompatible

_Backwards-**incompatible** changes_ 

* May _only_ be proposed to Alpha or Beta APIs
* Require updates to existing E2E tests to support these changes
* Breaking changes to existing Alpha or Beta APIs must be tracked and updated in docs/release notes

_Backwards-**compatible** changes_

* May be proposed to _any_ API
* Proposed changes to both the HTTP and gRPC API must be included


## Requirements for Building Block changes

Finally on addition of a new API, there may be addition of the capability to either an existing component or if it is a new building block, creation of a new set of components in the `dapr/components-contrib` repo.

### Creating new API as part of a new building block in `dapr/components-contrib`**

- Interfaces to be used by `dapr/dapr` code must be defined and agreed upon
- New building block package is defined in `components-contrib` repo, new code must only be added inside that building block package
- Conformance tests enable validating the components compliance with defined interface for the building block and creates a baseline for conformance testing any new components added. Conformance tests may be added for the new API with the understanding that it may evolve


**Creating new API for an existing building block in `dapr/components-contrib`**

- Interfaces changes for the new API must be defined and agreed upon
- Existing components that support the new API must be enhanced to be in compliance with the proposed interface as per the defined and agreed upon scope of the original proposal
- Conformance tests must be updated
  - Get sign off on a basic suite of conformance tests for the interface method(s)
  - Implement the suite of conformance tests as part of the existing suite of tests for the building block
- Ensure successful execution of existing conformance and certification tests for any modified components



## Progression of an API 

### Alpha to Beta 

In addition to the requirements that are required of any Alpha API, the following requirements must be met so that the API can graduate to Beta. For an API to be promoted to Beta, it must exist for at least one release cycle after its initial addition as Alpha. (i.e something added in 1.10 could become  Beta in 1.12, having been stabilized through 1.11)

For all APIs, the following criteria need to be met: 

* E2E test with extensive positive and negative scenarios must be defined
* Most (if not all) changes needed in the user facing structures must be considered to be complete (in order to reduce the number of breaking changes)
* All "core" SDKs must have support for this API _(Python, Go, .NET, Java?)_
* Documentation of the API must be completely up-to-date with any changes tha
* Quickstarts must be defined for the API allowing users to quickly explore the API
* Performance tests should be added (if not already available in Alpha stage) / updated where relevant


For **building blocks** to progress, the following criteria are required:

* Conformance test(s) must be added/updated. (this is for scenario for new building block not having conformance test in the Alpha stage)
* Conformance tests must test both positive and negative cases (i.e deliberately attempt to break them)
* Certification tests should be added to the different components and this API must be exercised in the certification tests
* Multiple implementations must be present for this building block

### Beta to Stable

In addition to the requirements for a Beta API, the following requirements must be met so that the API can graduate to Stable. Similar to the previous phase change, this API must have been in the Beta phase for at least one full release _without any breaking changes_. In addition, the following criteria apply:

* E2E scenarios must be well defined and comprehensive 
* Performance tests must be added/updated (this is for scenario where new building block does not have performance tests in the Alpha/Beta stage)
* Expected performance data must be added to documentation

For **building blocks** to progress, the following must also be true:

* E2E tests must exercise _at least two different implementations_ of the building block's API
* Conformance tests testing both positive and negative cases must be defined
* Certification tests for multiple components implementing this API must be defined
