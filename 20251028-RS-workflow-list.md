# Workflow List and History RPCs for Durable Task

* Author(s): @joshvanl

## Overview

This proposal adds two new RPCs to the durabletask framework which are used to support observability and discoverability of running and completed workflows in Dapr.
Specifically, adding a `ListInstances` and `GetInstanceHistory` RPC to the durabletask gRPC service.

## Background

Today, there is no ways of discovering the list of workflow instances that are currently running or have completed in the past without using external storage queries.
The Dapr CLI [introduced list and workflow history commands](https://github.com/dapr/cli/pull/1560) to get information about running and completed workflows, however these commands rely on direct queries to the underlying storage provider.
By introducing this functionality into the durabletask framework itself, these commands need only talk to Daprd, removing the requirement for direct access to the storage provider as well as authentication.
Daprd can make these queries itself, and use the Actor State Store component to access the underlying storage.

## Related Items

## Implementation Details

### Design

Two new RPCs will be added to the durabletask gRPC service, which will be available to both the application client, as well as the dapr CLI.
The list RPC will be used to discover the workflow instance IDs, which their metadatda can then be fetched.
The workflow history RPC will be used to fetch the full history of a given workflow instance.

```proto
service TaskHubSidecarService {
    rpc ListInstances (ListInstancesRequest) returns (ListInstancesResponse);
    rpc GetInstanceHistory (GetInstanceHistoryRequest) returns (GetInstanceHistoryResponse);
}

// ListInstancesRequest is used to list all orchestration instances.
message ListInstancesRequest {
    // continuationToken is the continuation token to use for pagination. This
    // is the index which the next page should start from. If not given, the first
    // page will be returned.
    optional uint32 continuationToken = 1;

    // pageSize is the maximum number of instances to return for this page. If
    // not given, all instances will be attempted to be returned.
    optional uint32 pageSize = 2;
}

// ListInstancesResponse is the response to executing ListInstances.
message ListInstancesResponse {
  // instanceIds is the list of instance IDs returned.
  repeated string instanceIds = 1;
}

// GetInstanceHistoryRequest is used to get the full history of an
// orchestration instance.
message GetInstanceHistoryRequest {
    string instanceId = 1;
}

// GetInstanceHistoryResponse is the response to executing GetInstanceHistory.
message GetInstanceHistoryResponse {
    repeated HistoryEvent events = 1;
}
```
