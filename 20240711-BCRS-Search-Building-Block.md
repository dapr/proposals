# Search Building Block

* Author(s): Mike Nguyen (@mikeee) - hey(at)mike.ee
* State: RFC - Request for Comment / EARLY DRAFT
* Updated: 2024-07-11

## Overview

This is a proposal to implement a new Search Building Block to Dapr, providing a
standardised API for integrating search-based functionality.

This proposal covers changes and additions to the following areas:

* The definition of a new building block and components-contrib type.
* The runtime will have changes to the exposed client API and communication with
a new component.
* SDKs updates to implement the client APIs.

## Background

As state stores grow in size and complexity, the need for an efficient and
scalable search capability becomes crucial. Currently there is only a query-API
in Alpha and does there has not been work to stablise it.

This building block has the potential to solve longer-term issues such as
searching stored state data and integrates well with present and future building
blocks providing an improved and consistent search operation developer
experience.

### Use Cases / Examples

* Retrieval & relevance searching e.g. finding relevant results in a shop.
* Specialised searching e.g. consuming a firehose event

This section is intented to provide the community with the reasoning behind this proposal -- why is this proposal being made? What problem is it solving for users / developers / operators and how does it solve that for them?

## Related Items

### Related proposals

Links to proposals that are related to this (either due to dependency, or possibly because this will replace another proposal)

### Related issues

Please link to any issues that this proposal is related to, for example, are there existing bugs filed in various Dapr repositories that this will affect?


## Expectations and alternatives

### Scope

Core functionality of searching and requirements including these operations:

* Index definitions
* GET/INSERT/UPDATE/DELETE document operations

This proposal includes implementing two backends initially, I believe
Meilisearch and Elasticsearch are good candidates with Go SDKs available.

### Explicit items not in scope

Search analytics is also not in scope for the first iteration.

Geospatial search capabilities are not in scope for this initial proposal.

### Alternatives considered

The alternative to this proposal is to extend the existing state management
building block - this has been partially tested through the query-API.

## Implementation Details

### Design

TODO

#### APIs

##### gRPC

The protobufs will be extended:

```proto
service Dapr {
…
// Create new documents
rpc InsertSearchDocumentsAlpha1(InsertSearchDocumentsRequest) returns (InsertSearchDocumentsResponse) {}

// Retrieves documents
rpc GetSearchDocumentsAlpha1(GetSearchDocumentsRequest) returns (GetSearchDocumentsResponse) {}

// Deletes documents
rpc DeleteSearchDocuments(DeleteSearchDocumentsRequest) returns (DeleteSearchDocumentsResponse) {}

// Updates a document
rpc UpdateSearchDocument(UpdateSearchDocumentRequest) returns (UpdateSearchDocumentResponse) {}
...
}

message SearchDocument {
    string id = 1;
    map<string, string> fields = 2;
}

message SearchRequest {
    string query = 1;
    int32 limit = 2;
    int32 offset = 3;
    map<string, string> filters = 4;
}

message InsertSearchDocuments {
    repeated SearchDocument documents = 1;
}

message InsertSearchDocumentsResponse {}

// Specify either a number of documents to retrieve or a request
message GetSearchDocumentsRequest {
    repeated SearchDocument documents = 1;
    SearchRequest request = 2;
}

message GetSearchDocumentsResponse {}

// Delete by document or search query
message DeleteSearchDocumentsRequest{
    repeated SearchDocument documents = 1;
    SearchRequest request = 2;
}

message UpdateSearchDocumentRequest{
    SearchDocument document = 1;
    SearchDocument updated = 2;
}
```

##### HTTP

TODO
`GET/PUT/POST/DELETE`

### Feature lifecycle outline

This proposal will follow this lifecycle outline

#### Alpha

* A search and document management API is implemented but may change as testing
takes place.
* Performance is not guaranteed. Numbers will be published.
* The core SDK support for this building block will be the Go SDK, a Rust SDK
implementation would be a non-core implementation.
* The initial components supported would be Meilisearch and Elasticsearch.
Documentation for setting these up alongside the SDK documentation will be added
to `dapr/docs`.
* Telemetry data (metrics) will be minimally included.
* An issue will be opened in `dapr/quickstarts` for examples to be created.
* Both HTTP and gRPC protocols are implemented.
* A new building block package will be defined in the `dapr/components-contrib`
  repository.
* Conformance tests will be added to `dapr/components-contrib`.
* An initial E2E test will be implemented in the runtime or sdk.
* No data migration support is available. Details of breaking changes made may
be provided.

#### Beta

* E2E tests updated and comprehensive.
* SDK spec is updated
* No major changes to the API would have occurred in the last release.
* Support in the core SDKs as far as is practicable, with a focus on ensuring
the groundwork is complete and SDK feature additions are added where maintainers
have bandwidth. The lack of this building block from an SDK will not constitute
a blocker to graduation from Beta.
* Documentation updates ready for graduation to beta.
* Quickstarts will have been updated where necessary.
* Performance tests will exist.
* Certification tests will be added to `dapr/components-contrib`
* No data migration support is available. Details of breaking changes may be
provided.

#### Stable

* TODO

### Acceptance Criteria

How will success be measured?

* Proof of concept with loading a test dataset and ensuring consistency with
returned results across two search backends.
* Ensuring operations are completed in an acceptable amount of time.
* Defining and reporting limitations of the building block.

* Performance targets
* Compabitility requirements
* Metrics

## Completion Checklist - Alpha / Beta

* [ ] Code Changes
  * [ ] Runtime
  * [ ] Components-Contrib
    * [ ] Meilisearch
    * [ ] Apache Solr [Beta]
    * [ ] Elasticsearch
* [ ] Tests
  * [ ] E2E
  * [ ] Unit [Beta]
  * [ ] Integration [Beta]
  * [ ] Components-Contrib
    * [ ] Certification [Beta]
    * [ ] Conformance
* [ ] SDK Implementations
* [ ] Documentation
  * [ ] Runtime
  * [ ] Components-Contrib
  * [ ] SDK
    * [ ] Go-SDK
    * [ ] Rust-SDK [Non-blocking]

The completion checklist will be split into two categories - Alpha and Beta.
