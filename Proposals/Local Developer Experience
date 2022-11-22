# Dapr User Day 1 Experience [EPIC]

* Author(s): Mukundan Sundararajan (@mukundansundar), Radhika Bollineni(rabollin)
* State: Feature 1[<https://github.com/dapr/proposals/pulls/6>] is Ready for Implementation
* Updated: 11/22/2022

## Introduction

As a Dapr user, I need to be able to quickly setup, run and experiment using Dapr. As a new user looking at Dapr, this proposal aims to improve the experience for the user to quickly onboard onto Dapr.

## Scope

In local development environments it should be easy to run, experiment, use Dapr. Currently the scope only targets local development using self hosted mode of Dapr. In future, we can also look into improving our k8s and local-cloud experiences.

## Current challenges

Aligning to the developer life cycle in building the microservices with DAPR, there have been multiple challenges that needs improvements to get better developer experience. We can classify the highlevel problems as below:

* Increased development time complexity due to dapr side cars

* There is a need to run multiple services with multiple `dapr run` commands to get a quick microservice setup up and running

* Apps need to be started with a lengthy “dapr run…” command

* Debugging in DAPR is not that straight forward

Doubling down on the areas, we get into multiple items that needs improvement or better approach :

* Ease of local development:
  * There is a need to keep track of multiple ports, app-ids, etc
  * There is a need to run multiple services with multiple `dapr run` commands to get a quick microservice setup up and running
  * There is a need to define multiple component files, with different metadata per component increases the learning time of DAPR
  * Cannot access building blocks like statestore, secrets, configuration, distributed lock etc. using `dapr` CLI commands
  * Cannot access metadata API via CLI commands
* Using different components:
  * There are no tools to easily discover the supported components in Dapr
  * There is no proactive/easy way to validate component YAMLs for correctness
  * There is no easy way to view all potential metadata fields that are valid in a component definition
* Use of specific features in Dapr:
  * There are no quickstarts that show the use of features in Dapr like scopes, resiliency, subscription CRD etc.
  * There are no examples/quickstarts for a some of the building blocks in `dapr`

## Potential solutions for the problems identified above

Proposed solutions for some of the problems identified above:

* Component discovery and definition (Related to <https://github.com/dapr/components-contrib/issues/2014> - component metadata schema)
  * Add command to [lint](https://github.com/dapr/cli/issues/1074) component files and validate them
    * This is for allowing users to quickly validate the correctness of the YAML configuration before running it. Also this is expected to help users by providing informational error messages so that users can quickly identify issues in their YAML configuration
  * Add command to [discover and generate](https://github.com/dapr/cli/issues/992) YAML for components with sane defaults
    * This is adding the ability in the CLI to be able to discover the different components supported by Dapr and also to automatically generate the YAML configuration file with sane defaults
* Expose all building blocks/features in Dapr through `dapr` CLI, dashboard
  * Enhancement needed in `dapr` dashboard also to support all different building blocks
    * Support for showing metadata for an application
    * Support for showing all new building blocks (configuration API, distributed lock API)
    * Support for showing subscription CRDs, resiliency CRDs
  * [Default secret store](https://github.com/dapr/cli/issues/1066) on init
    * This enhancement provides a default secret store on init which can be used locally for development and quickstarts examples for secrets building block
  * Have quickstarts defined for all building blocks
  * Improve existing command support pubsub building block
    * Add support for a subscribe command with limited console subscriber [functionality](https://github.com/dapr/cli/issues/1127)
  * Add additional command support for bindings, secrets, configuration, state management etc. to improve quick use of the API without writing curl commands or writing an application
    * Add support in `dapr` CLI to directly invoke output bindings
    * Add support for getting and listing of secrets
    * Add support for getting and listing configurations
    * Add support for CRUD operations for state store
  * [Feature] Add support for [resiliency defaults](https://github.com/dapr/cli/issues/1027)
    * Provide default resiliency policies that can be used by users for their local developments
  * [Feature] Add command support for all [custom CRDs](https://github.com/dapr/cli/issues/950) in Dapr
    * Similar to `dapr components`, `dapr configuration`, add ability to list and view `resiliency and subscription` CRDs
    * Add support for users to view CRDs in local dev environment apart from Kubernetes support
* Improve `dapr init` experience
* Have a CLI [config file](https://github.com/dapr/cli/issues/1058) with sane defaults and additional knobs to improve `init` experience
* Run multiple services with Dapr sidecar, and keep track of ports
  * Proposed solution is [dapr compose](https://github.com/dapr/cli/issues/1123)

## Targets for 1.10

The following enhancements are targeted for 1.10 release:

* [] Add command to [lint](https://github.com/dapr/cli/issues/1074) component files and validate them
* [] [Feature] Add support for [resiliency defaults](https://github.com/dapr/cli/issues/1027)
* [] [Feature] Add command support for all [custom CRDs](https://github.com/dapr/cli/issues/950) in Dapr
* [] [Default secret store](https://github.com/dapr/cli/issues/1066) on init
* [] Support for showing metadata for an application
* [] Have quickstarts defined for all building blocks
* [] Alpha version of `dapr compose` command. Linux support at start.

**Please see <https://github.com/dapr/proposals/pulls/6> For the proposal PR for Dapr Compose**
