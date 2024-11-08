# Ownership of Dapr + .NET Aspire Integration 

* Author(s): Phillip Hoff (@philliphoff)
* State: Ready for Implementation
* Updated: 2024-11-08

## Overview

This is a proposal for the ownership transfer of the Dapr + .NET Aspire integration to the Dapr .NET SDK repo and its maintainers (with approval of the .NET Aspire team). This provides a central location for tooling covering both development and test aspects of Dapr development in .NET.

## Background

Dapr [integration](https://github.com/dotnet/aspire/tree/main/src/Aspire.Hosting.Dapr) was added to .NET Aspire as part of its initial Public Preview in December 2023, namely the ability to automate the launch of Dapr sidecars for individual .NET projects. Additional features were added as .NET Aspire reached GA, namely the ability to associate and automated component configurations with Dapr sidecars.

As the popularity of .NET Aspire has grown, it has accreted a significant number of additional product integrations, most of which, like Dapr, live in the main [.NET Aspire GitHub repo](https://github.com/dotnet/aspire). This arrangement results in a number of issues:

 - The management (i.e. build, test, and publish) of these integrations has increased the workload of .NET Aspire maintainers
 - .NET Aspire maintainers may not have the expertise in each product area to judge the applicability and quality of proposed changes
 - .NET Aspire maintainers may not have the bandwidth to pursue improvements and fixes of integrations
 - Updates to integrations will tend to align with the (typically longer) schedule of .NET Aspire itself rather than the product's typical lifecycle

These issues all conspire to cause integrations, like Dapr, to languish, fall behind the latest product features, and result in customer dissatisfaction that extends to the product itself.

By transferring ownership of the Dapr + .NET Aspire integration to the Dapr .NET SDK repo and its maintainers:

 - It brings the integration closer to and under the purview of those in the best place to respond to issues and maintain quality
 - It allows .NET Aspire integration features to be prioritized on a Dapr-centered scale than a .NET Aspire scale
 - It enables management (i.e. build, test, and publish) on a schedule that matches the product lifecycle

## Expectations and alternatives

 - TBD: How does the licensing of the existing integration source change, if at all?
   - Dapr .NET SDK source is Apache v2.0 licensed
   - .NET Aspire source is MIT licensed
 - Both the Dapr .NET SDK maintainers and .NET Aspire maintainers will "co-own" the `Aspire.Hosting.Dapr` NuGet package and have publication rights
   - This is more a function of the NuGet organization and ownership model than anything else

## Implementation Details

### Design

 - The Dapr .NET Aspire integration [source](https://github.com/dotnet/aspire/tree/main/src/Aspire.Hosting.Dapr) in [dotnet/aspire](https://github.com/dotnet/aspire) will move to the Dapr .NET SDK [repo](https://github.com/dapr/dotnet-sdk) as a sibling to its existing [packages](https://github.com/dapr/dotnet-sdk/tree/master/src)
  - Likewise, for any existing Dapr-specific test projects
 - The `Aspire.Hosting.Dapr` NuGet package will be built, tested, and published as part of the Dapr .NET SDK CI pipeline
 - TBD: Should documentation of use of Dapr with .NET Aspire move from its existing [location](https://learn.microsoft.com/en-us/dotnet/aspire/frameworks/dapr?tabs=dotnet-cli) to the [Dapr documentation site](https://docs.dapr.io/)?

### Feature lifecycle outline

 - TBD: How will the `Aspire.Hosting.Dapr` NuGet package version?
   - The existing .NET Aspire versioning scheme (e.g. 8.x, 9.x) does not match the existing Dapr versioning scheme (e.g. 1.14.x, 1.15.x)
 - The `Aspire.Hosting.Dapr` NuGet package will maintain compatibility with the current releases of .NET Aspire 
   - Preview versions of .NET Aspire will not necessarily be supported, or only at a "best effort level

### Acceptance Criteria

 - Source has moved to Dapr repo
 - Source has been built/tested as part of a Dapr CI pipeline
 - `Aspire.Hosting.Dapr` NuGet package published by Dapr

## Roadmap

TBD

## Completion Checklist

 - [ ] Licensing questions resolved
 - [ ] Source moved to Dapr repo
 - [ ] Versioning questions resolved
 - [ ] Source integrated into Dapr CI pipeline
 - [ ] NuGet package published
 - [ ] Documentation moved and/or updated
