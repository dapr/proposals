# Feature name

> If you need more information on what needs to be completed, look in the `guides` directory for relevant guidance.

# Links

Links to any relevant resources go here:

* Relevant proposal
* Existing issues
* Milestones

# Lifecycle Expectations

## Alpha / Beta / Stable

For each stage, identify the expectations of this feature at that stage. For example, 
are there any performance issues, configuration changes or feature deprecation that will happen?

* Anticipated performance / known limitations
* Compatability guarantees / requirements
* Deprecation / co-existence with existing functionality
* Feature flags required

# Acceptance criteria

> For each of the stages, add any specific tasks that need to be completed before this feature reaches that particular stage. If a particular item is *not needed* then a reason should be given.

## Alpha

- [ ] Minimum of one core SDK supports this feature (.NET / Python / Go / Java)
- [ ] Feature documentation added to `dapr/docs`
- [ ] Telemetry data (metrics) available for this feature
- [ ] Issue opened in `dapr/quickstarts` for quickstart examples to be created

Additionally, for **APIs**:

- [ ] Both HTTP and gRPC protocols implemented
- [ ] HTTP API documentation added to the `Reference` section of Dapr documentation

Additionally, for **building blocks**:

- [ ] Interfaces to be used by `dapr/dapr` code defined and agreed upon
- [ ] New building block package is defined in `components-contrib` repo
- [ ] Conformance tests validating the components compliance added
- [ ] Minimum of _one_ implementation (preferably something already in-use such as Redis if possible to reduce complexity)


## Beta

- [ ] E2E tests are up-to-date and comprehensive
- [ ] SDK spec is updated
- [ ] No major changes to the API have occurred in the last XXX time period (releases? months?)
- [ ] Support in core SDKs
   - [ ] Python
   - [ ] Go
   - [ ] Java
   - [ ] .NET
- [ ] Documentation up-to-date with any new changes since Alpha
- [ ] Quickstarts have been created
- [ ] Performance tests exist but do not block builds

Additionally, for **building blocks**:

- [ ] Conformance tests updated to match any API changes that have been made
- [ ] Conformance tests exercise both positive and negative cases 
- [ ] Minimum of N (three?) implementations of this building block 
- [ ] Certification tests for implementations 
- [ ] APIs that are used in the building block also meet Beta criteria


## Stable 


- [ ] Documentation is complete in `dapr/docs` with any changes since Beta
- [ ] E2E scenarios well defined and comprehensive
- [ ] Performance tests exist and regressions will prevent them from successfully passing
- [ ] Performance data added to documentation (https://docs.dapr.io/operations/performance-and-scalability/)

