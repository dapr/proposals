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
   * Recommended for non-business-critical use only because of potential for incompatible changes in subsequent releases
   * May not be backwards-compatible with an API it intends to replace
   * May not be highly performant or support all SDKs

* **Beta**
   * API is not production ready yet
   * If an API moves into Beta, the intention is that it will continue on to become stable and not be removed
   * Multiple components implement the API and API contract is mostly finalized
   * Recommended for non-business-critical use only due to potentially backwards-incompatible changes in subsequent releases
   * Should have support in (at least) the _"core"_ SDKs _(i.e. Python, Go, Java)_
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

All new APIs that are defined start at the Alpha stage.

* Both HTTP and gRPC protocols should be supported for the new API
* Documentation must be provided for the API
  * HTTP API must be added to the `Reference` section in the Dapr documentatio
* Issues should be added in `dapr/quickstarts` to create examples for the new API to enable users to easily explore the new functionality provided by the API
* If the new API is considered an _optimization_ of an existing API (say, the addition of `BulkGetSecrets` alongside `GetSecret`) then:
  * The performance improvement gained due to this API should be documented
  * Guidance must be provided to the users in docs as to when to use this API vs using the older one
* Performance tests should (though preferably must) exist for this new API
* _Should_ include new E2E tests that exercise the API


### Updates to existing APIs

Depending on the phase of the existing API, the proposed changes may or may not be backwards-incompatible

_Backwards-**incompatible** changes_

* May _only_ be proposed to Alpha or Beta APIs
* Require updates to existing E2E tests to support these changes
* Breaking changes to existing Alpha or Beta APIs must be tracked and updated in docs/release notes

_Backwards-**compatible** changes_

* May be proposed to _any_ API
* Proposed changes to both the HTTP and gRPC API must be included

### Progression of API lifecycle to Stable

Currently for HTTP API, the alpha API has the path `v1.0-alpha1` and the beta API must have the path `v1.0-beta1` and the stable APIs must have the form `v1.0`. Similarly for gRPC, the alpha and beta APIs must have the suffix `Alpha1` or `Beta1` added to the gRPC methods.
SDKs implementing the API before the stable release use the above mentioned paths/methods for HTTP and gRPC respectively. Once the API is promoted to stable, the SDKs must be updated to use the stable paths/methods as soon as possible.

#### Compatibility

For `alpha` to `beta` or `alpha` to `stable` or `beta` to `stable` progressions, the following rules must be followed:
- Once an API has progressed to "higher" stage be it `beta` or `stable` from `alpha`, the prior API endpoints (HTTP/gRPC) must be still supported together with the newer stage (`beta` or `stable`) API endpoints in Dapr runtime for atleast 1 release to give SDKs enough time to update to the new API endpoints. Functionality wise, there may be breaking changes between a API stage transitions, but both the prior API endpoint and the newer endpoint must point to the same same functionality once it has graduated to a particular stage.

> Note: The components themselves might have breaking changes, that will not affect the API-SDK compatibility.

Example Scenario:
Consider v3.0 of JS SDK supporting the Config API in Alpha stage and Dapr runtime v1.10. The Config API is promoted to Stable in v1.11 of the runtime. In this case both path `/v1.0-alpha1`, method`Alpha1` and path `/v1.0`, ,method without suffix must be supported by the Dapr runtime.

Then the SDK can be updated independent of the runtime i.e. v3.0 version of JS SDK will still continue to work with the v1.11 runtime Config API.

> Note: This guidance is specifically for Dapr runtime API-SDK compatibility. SDKs may have their own way of exposing/differentiating between Alpha and Stable APIs.

## Requirements for Building Block changes

Finally on addition of a new API, there may be addition of the capability to either an existing component or if it is a new building block, creation of a new set of components in the `dapr/components-contrib` repo.

### Creating new API as part of a new building block in `dapr/components-contrib`**

- Interfaces to be used by `dapr/dapr` code must be defined and agreed upon
- New building block package is defined in `components-contrib` repo, new code must only be added inside that building block package
- Conformance tests enable validating the components compliance with defined interface for the building block and creates a baseline for conformance testing any new components added. Conformance tests may be added for the new API with the understanding that it may evolve


### Creating new API for an existing building block in `dapr/components-contrib`

- Interfaces changes for the new API must be defined and agreed upon
- Existing components that support the new API must be enhanced to be in compliance with the proposed interface as per the defined and agreed upon scope of the original proposal
- Conformance tests must be updated
  - Get sign off on a basic suite of conformance tests for the interface method(s)
  - Implement the suite of conformance tests as part of the existing suite of tests for the building block
- Ensure successful execution of existing conformance and certification tests for any modified components

## Progression of an API/Building block
### Alpha to Beta

In addition to the requirements that are required of any Alpha API, the following requirements must be met so that the API can graduate to Beta. For an API to be promoted to Beta, it must exist for at least one release cycle after its initial addition as Alpha. (i.e something added in 1.10 could become  Beta in 1.12, having been stabilized through 1.11)

For all APIs, the following criteria need to be met:

* E2E test with extensive positive and negative scenarios must be defined
* Most (if not all) changes needed in the user facing structures must be considered to be complete (in order to reduce the number of breaking changes)
* All _"core"_ SDKs must have support for this API _(i.e. Python, Go, .NET, Java)_
* Documentation of the API must be completely up-to-date
* Quickstarts must be defined for the API allowing users to quickly explore the API
* Performance tests should be added (if not already available in Alpha stage) / updated where relevant


For **building blocks** to progress, the following criteria are required:

* Conformance test(s) must be added(in case a new building block does not have conformance tests in the Alpha stage)/updated
* Conformance tests must test both positive and negative cases (i.e deliberately attempt to break them)
* Certification tests should be added to the different components and this API must be exercised in the certification tests
* Multiple implementations must be present for this building block

### Beta to Stable

In addition to the requirements for a Beta API, the following requirements must be met so that the API can graduate to Stable. Similar to the previous phase change, this API must have been in the Beta phase for at least one full release _without any breaking changes_. In addition, the following criteria apply:

* Change API version based on the [API version progression guidelines](#progression-of-api-lifecycle-to-stable)
* E2E scenarios must be well defined and comprehensive
* Performance tests must be added(in case a new building block does not have performance tests in the Alpha/Beta stage)/updated
* Expected performance data must be added to documentation

For **building blocks** to progress, the following must also be true:

* E2E tests must exercise _at least two different implementations_ of the building block's API
* Conformance tests testing both positive and negative cases must be defined
* Certification tests for multiple components implementing this API must be defined
