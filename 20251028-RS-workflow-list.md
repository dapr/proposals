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
    rpc ListInstanceIDs (ListInstanceIDsRequest) returns (ListInstanceIDsResponse);
    rpc GetInstanceHistory (GetInstanceHistoryRequest) returns (GetInstanceHistoryResponse);
}

// ListInstanceIDsRequest is used to list all orchestration instances.
message ListInstanceIDsRequest {
    // continuationToken is the continuation token to use for pagination. This
    // is the token which the next page should start from. If not given, the
    // first page will be returned.
    optional string continuationToken = 1;

    // pageSize is the maximum number of instances to return for this page. If
    // not given, all instances will be attempted to be returned.
    optional uint32 pageSize = 2;
}

// ListInstanceIDsResponse is the response to executing ListInstanceIDs.
message ListInstanceIDsResponse {
  // instanceIds is the list of instance IDs returned.
  repeated string instanceIds = 1;

  // continuationToken is the continuation token to use for pagination. If
  // there are no more pages, this will be null.
  optional string continuationToken = 2;
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

In order to implement the listing functionality, a new `KeysLike` state store API will be added to components-contrib.
This API allows listing keys which match a given SQL LIKE style wildcard pattern ("%" and "_").
This API will be implemented solely to enable the workflow instance listing functionality in Dapr, however could be exposed to the general state APIs in future.

```go
// KeysLiker is an optional interface to list state keys with an
// optional SQL style wildcard pattern.
type KeysLiker interface {
	KeysLike(ctx context.Context, req *KeysLikeRequest) (*KeysLikeResponse, error)
}
```

```go
type KeysLikeRequest struct {
	// Pattern is the SQL LIKE pattern to match keys against.
	Pattern string `json:"pattern"`

	// ContinueToken is an optional parameter to indicate the key from which to
	// start the search.
	ContinueToken *string `json:"startKey,omitempty"`

	// PageSize is an optional parameter to indicate the maximum number of keys
	// to return.
	PageSize *uint32 `json:"pageSize,omitempty"`
}
```

```go
// KeysLikeResponse is the response object for getting keys like a pattern.
type KeysLikeResponse struct {
	Keys []string `json:"keys"`

	// ContinueToken is an optional token which can be used to continue the
	// search of keys. Usually only present if a `PageSize` was set on the
	// request.
	ContinueToken *string
}
```

