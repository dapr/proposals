# Unify the `DAPR_API_TOKEN` env variable across all SDKs

* Author(s): Elena Kolevska (@elena-kolevska)
* State: Ready for Implementation
* Updated: 2023-09-18

## Overview

This is a design proposal to unify usage of the `DAPR_API_TOKEN` variable across all SDKs.

## Background

Currently, the `DAPR_API_TOKEN` env variable is used in some SDKs, but not in others. For example, the Python SDK uses it, but the Javascript SDK does not. This can be confusing for users, and makes it difficult to write documentation that is consistent across all SDKs. It is also an unnecessary complication for maintainers who switch between SDKs. 

And finally, with a unified environment variable across all SDKs system administrators don't need to have different configurations per application based on programming language, because the same environment variables will work with every SDK.

## Related Items

This has already been discussed in different repos, notably in [this issue](https://github.com/dapr/java-sdk/issues/303) in the java-sdk.

We are moving to unify other environment variable across all SDKs, as discussed in [this proposal](https://github.com/dapr/proposals/blob/main/0008-S-sidecar-endpoint-tls.md).


## Expectations and alternatives

**What is in scope for this proposal?**  
- All supported SDKs need to consistently support the `DAPR_API_TOKEN` environment variable for authentication to Dapr

**SDKs that already use the `DAPR_API_TOKEN` env variable:**  
- java-sdk
- go-sdk
- dotnet-sdk
- python-sdk
- php-sdk

**SDKs that currently don't use the `DAPR_API_TOKEN` env variable:**  
- js-sdk
- rust-sdk
- cpp-sdk


## Implementation Details

### Design

- The `DAPR_API_TOKEN` environment variable will be used in all SDKs to authenticate to Dapr. The token will be passed to the Dapr sidecar via the `dapr-api-token` header.


### Feature lifecycle outline

- Some SDKs currently accept the a Dapr API token as an argument through the constructor. In this case, when a token is specified through the constructor of the client, it will take precedence over the environment variable.
- If there is an existing environment variable for the Dapr api token by a different name, the new `DAPR_API_TOKEN` variable will take precedence.

### Acceptance Criteria

How will success be measured? 

* Performance targets: N/A
* Compabitility requirements: Same environment variables work with any SDK
* Metrics: N/A

## Completion Checklist

What changes or actions are required to make this proposal complete?

* SDK changes (if needed)
    - Add support for the `DAPR_API_TOKEN` environment variable to all supported SDKs
    - Add integration testing on each SDK when possible
* Documentation
    - Update documentation to reflect the new environment variable

