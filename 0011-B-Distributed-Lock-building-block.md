# Distributed Lock building block

* Author(s): Alessandro Segala (ItalyPaleAle), based on @seeflood's original design
* State: Partially implemented
* Updated: October 2023

## Overview

Dapr has been including a "distributed lock" building block since version 1.8, which was originally proposed in dapr/dapr#3549. However, the current implementation is incomplete, with the original design never fully implemented, and today we are missing some key capabilities that make these APIs useful for most cases.

This proposal is built on top of dapr/dapr#3549, with the goal of completing the design, rationalizing some APIs, and providing the foundation for completing the implementation.

After this proposal is accepted and implemented, it should be possible to move the distributed lock building block towards a more mature (beta and eventually stable) status.

## Background

This proposal is meant to complete the work started in dapr/dapr#3549. Please refer to that issue for a discussion on the busines problem that the building block is trying to solve, as well as the context.

Currently, as implemented the building block offers these 2 APIs:

- A non-blocking `TryLock` method, which attempts to acquire a lock on a resource if it's not already locked, or fails otherwise.
- An `Unlock` method to release a lock.

### Outstanding issues

As explained in [this comment](https://github.com/dapr/dapr/issues/3549#issuecomment-1484399578), the current implementation lacks some features that are required to make the building block usable for more general scenarios. In particular, we need to implement _at least_:

- A blocking `Lock` method. Unlike `TryLock`, if the resource is already locked, this method blocks until it becomes available (with a maximum timeout).
- Support for renewing locks, either explicitly (via a `RenewLock`), implicitly (as long as the app is healthy, the lock is renewed by the Dapr sidecar unless it's explicitly released), or tied to a long-lived request/stream.
- Better response messages and status codes. For example, the APIs today return an Internal Server Error in all error cases. It's not possible, for example, to understand if `TryLock` failed because the lock was already acquired, or because of an internal error in Dapr; likewise, `Unlock` doesn't differentiate the error response based on whether the lock was invalid or an error occurred.
- The current HTTP APIs are not RESTful, as they do not leverage the HTTP verbs to perform operations.

We also have the opportunity to rationalize the concepts of lock IDs, which are called "owners" in the current APIs. Currently, the caller is responsible for providing an owner, which means the caller (the apps) need to track them.

### More components

Although not directly part of this proposal, which focuses on the building block's APIs, for the building block to become beta/stable we also need to add more components. Currently, the building block only support Redis (in "cluster" mode); other possible backends include:

- [Kubernetes native leases](https://kubernetes.io/docs/concepts/architecture/leases/#custom-workload)
- [etcd leases](https://etcd.io/docs/v3.5/tutorials/how-to-create-lease/)
- Dapr Actors, which allow using any Dapr state store that can be used as actor state store (and locks are automatically cleaned up using reminders)

## Implementation Details

### Backwards compatibility

The proposed changes are not backwards-compatible by design. Given the current APIs alpha status, Dapr makes no promises of maintaining backwards compatibility.

However, since the current APIs have been present since Dapr 1.8 and haven't changed since they have been introduced, the proposal is to:

- The new APIs will have version "alpha2".
- Maintain the existing APIs as "alpha1" for one more Dapr release, showing a warning that they are deprecated and will be removed in the next Dapr version.
- Because the proposed changes impact both components and runtime, this may require keeping the existing code (especially for the existing component) in a "legacy" folder temporarily, for one release.

### Components interface

The interface for all lock components is expanded to the following:

```go
// Store is the interface for lock stores.
type Store interface {
 	metadata.ComponentWithMetadata

 	// Init the lock store.
 	// Common metadata properties include:
 	// - "ttl": default TTL for locks (as a Go duration)
 	Init(ctx context.Context, metadata Metadata) error

 	// Features returns the list of supported features.
 	Features() []Feature

 	// Lock acquires a lock.
 	// By default, if the lock is owned by someone else, this method blocks until the lock can be acquired or the context is canceled.
 	Lock(ctx context.Context, req LockRequest) (LockResponse, error)

 	// TryLock tries to acquire a lock.
 	// If the lock cannot be acquired, it returns immediately with the error ErrResourceLocked.
 	TryLock(ctx context.Context, req LockRequest) (LockResponse, error)

 	// RenewLock attempts to renew a lock if the lock is still valid.
 	// If the lock ID is invalid, or the resource is not locked, returns ErrLockNotFound.
 	RenewLock(ctx context.Context, req RenewLockRequest) error

 	// Unlock tries to release a lock if the lock is still valid.
 	// If the lock ID is invalid, or the resource is not locked, returns ErrLockNotFound.
 	Unlock(ctx context.Context, req UnlockRequest) error
}

var (
	ErrResourceLocked = errors.New("the resource is already locked")
 	ErrLockNotFound   = errors.New("the resource is locked with a different lock ID or the lock does not exist")
)

// LockRequest is the request to acquire locks, used by Lock and TryLock.
type LockRequest struct {
	// ID of the resource to lock.
 	// This is an arbitrary value supplied by the user.
 	ResourceID string `json:"resourceId"`
 	// Maximum duration for the lock (if not renewed).
 	// If <= 0, the lock uses the default TTL set in the component.
 	TTL time.Duration `json:"ttl"`
}

// LockResponse is the response used by Lock and TryLock when the operation is completed.
type LockResponse struct {
 	// ID of the lock.
 	LockID  string `json:"lockID"`
}

// RenewLockRequest is a lock renewal request.
type RenewLockRequest struct {
 	// ID of the resource whose lock needs to be renewed.
 	// This is an arbitrary value supplied by the user.
 	ResourceID string `json:"resourceId"`
 	// ID of the lock.
 	LockID string `json:"lockID"`
 	// Maximum duration for the lock (if not renewed).
 	// If <= 0, the lock uses the default TTL set in the component.
 	TTL time.Duration `json:"ttl"`
}

// UnlockRequest is a lock release request.
type UnlockRequest struct {
 	// ID of the resource that is locked.
 	ResourceID string `json:"resourceId"`
 	// ID of the lock.
 	LockID  string `json:"lockID"`
 	// If true, will force the lock to be released, even if the lockID does not match.
 	// This may not be supported by all lock stores.
 	// This is a potentially destructive operation that should be used with caution.
 	Force bool `json:"force"`
}
```

The main differences include:

- New blocking `Lock` and `RenewLock` methods.
- Removed the concept of "lock owners", supplied by the caller, in favor of lock IDs that are generated by the component.
- Standard errors are returned to indicate situations such as resource already locked or resources that fails to be unlocked (or whose lock fails to be renewed) due to an invalid lock ID.
- TTLs are pased as `time.Duration`, and components have a default TTL.
- Added a `force` option in `Unlock`.

### Runtime APIs

The new APIs aim at offering a simplified user experience where developers interact with at most 3 Dapr APIs, and Dapr takes care of all background activities including renewing locks. This allows Dapr to provide additional, differentiated value to users too.

We propose two possible ways to implement the new APIs, which are both proposed below, each with pros and cons. The difference lies in how the user removes the lock.

**In both cases**:

- Dapr renews locks automatically in background. Apps do not need to invoke a method to renew a lock explicitly. The TTL sent in requests is only used to control how frequently locks are to be renewed (lower TTLs cause locks to be renewed more frequently, but allow for quicker detection of failed hosts).
- If the app becomes unhealthy (when app healthchecks are enabled) or if the sidecar crashes, locks are not renewed and are left to expire when the TTL ends.

> About force unlocking: this is implemented primarily to recover from situations where the lock was owned by an app that is known to have crashed. This is a potentially dangerous activity, as the previous owner of the lock is not informed (at least, not immediately) that the lock has been taken away, at least not until it tries to renew or relinquish the lock.  
> Some components may not implement forced unlocking.

#### Option 1: explicit `Unlock`

In this case, there are three methods:

- `Lock` is used to acquire a lock (unless `immediate` is true, this blocks if the resource is already locked, until the request is canceled). The response includes a lock ID which is the identifier of the lock, and the app can use it to unlock the resource.
- `Unlock` is invoked with the lock ID to unlock a resource.
- `ForceUnlock` is invoked to force the removal of a lock.

For gRPC:

```proto
service Dapr {
  rpc Lock (LockRequest) returns (LockResponse);
  rpc Unlock (UnlockRequest) returns (UnlockResponse);
  rpc ForceUnlock (ForceUnlockRequest) returns (ForceUnlockResponse);
}

message LockRequest {
  // When set to true, if the resource is already locked, the method returns right away with an error, and does not wait for the resource to become available.
  bool immediate = 1;
  // Resource ID.
  string resource_id = 2;
  // If <= 0, uses the default TTL.
  Duration ttl = 3;
}

message LockResponse {
  // Lock ID. The app should treat this as an opaque value.
  string lock_id = 1;
}

message UnlockRequest {
  // Resource ID.
  string resource_id = 1;
  // Lock ID.
  string lock_id = 2;
}

message UnlockResponse {
  // Empty
}

message ForceUnlockRequest {
  // Resource ID.
  string resource_id = 2;
}

message ForceUnlockResponse {
  // Empty
}
```

For HTTP (versions are omitted):

```md
# Lock

**POST /lock/<resource-id>**

Body (JSON):

- `immediate`: bool
- `ttl`: string (Go duration)

Response (JSON):

- `lockID`: string

# Unlock and Force Unlock

**DELETE /lock/<resource-id>**

Body (JSON):

- `force`: bool
- `lockID`: string

No response
```

Pros:

- Probably simpler to implement in the SDKs.
- Has simple, 1:1 mapping to HTTP methods.

Cons:

- If an app forgets to relinquish a lock, it could be renewed forever by the Dapr sidecar.
- Apps are not notified if the lock is lost. Doing so would require adding a new method to the app callback channel.

> Note: The fact that apps are not notified if a lock is lost (for example, if renewing a lock fails) may be a significant limitation.  
> A possible solution could involve adding a new method to the app callback channel to notify the app when a lock is lost. However, this does increase the complexity significantly, as apps are now required to create a separate listener and/or method to handle this scenario. It also requires the app to have an app channel enabled.

#### Option 2: Tied to a long-lived stream

In this case, there are only 2 methods:

- `Lock` is a server-streamed RPC and it's used to acquire a lock:
  - Apps start by making a call to `Lock`
  - If `immediate` is false (the default), then the method blocks until a lock is acquired.
  - If `immediate` is true and the resource is already locked, then an error is returned right away and the stream is closed
  - After the lock is acquired (regardless of whether it was immediate or not), the caller receives a signal on the RPC.
  - The RPC remains alive after that.
  - If the client disconnects, Dapr assumes that the lock is to be relinquished.
  - If the lock is lost (e.g. after a failed renewal attempt), Dapr sends an error on the RPC and closes it.
- `ForceUnlock` is invoked to force the removal of a lock.

For gRPC:

```proto
service Dapr {
  rpc Lock (LockRequest) returns (stream LockServerStream);
  rpc ForceUnlock (ForceUnlockRequest) returns (ForceUnlockResponse);
}

message LockRequest {
  // When set to true, if the resource is already locked, the method returns right away with an error, and does not wait for the resource to become available.
  bool immediate = 1;
  // Resource ID.
  string resource_id = 2;
  // If <= 0, uses the default TTL.
  Duration ttl = 3;
}

message LockServerStream {
  oneof lock_server_stream {
    LockPing ping = 1;
    LockAcquired lock_acquired = 2;
  }
}

message LockAcquired {
  // This is an empty message.
  // Receiving this message means that the lock has been acquired.
}

message LockPing {
  // This is an empty message.
  // The server can send this message periodically to make sure the channel is still active.
  // The app does not need to respond.
}

message ForceUnlockRequest {
  // Resource ID.
  string resource_id = 2;
}

message ForceUnlockResponse {
  // Empty
}
```

For HTTP (versions are omitted):

```md
# Lock

**POST /lock/<resource-id>**

Body (JSON):

- `immediate`: bool
- `ttl`: string (Go duration)

Response is a long-lived stream. Server will send messages as ND-JSON (Newline-Delimited JSON) when they are available. Messages can be:

1. `{"type": "lock-acquired"}\n` to indicate a lock was acquired
2. `{"type": "error", "error": "<error-message>"}\n` to indicate an error (the stream is closed after this)
3. `{"type": "ping"}\n` to indicate a ping (used by the server to ensure the stream is still alive)

# Force Unlock

**DELETE /lock/<resource-id>**

No body

No response
```

Pros:

- The lifecycle of the lock is strongly tied to the `Lock` RPC. Failures are detected automatically.
- There's a built-in way for Dapr to notify apps that the locks have been lost, as Dapr can close the server stream with an error.
- Probably simpler approach for end users.

Cons:

- Probably harder to implement in SDKs.
- This works really well with protocols based on HTTP/2 (including gRPC and REST calls that use HTTP/2, as it's now supported by the Dapr's server). However, if the client uses HTTP/1, this will require a dedicated HTTP connection, and thus a dedicated TCP socket, for this method, and it's wasteful (this is not a problem with HTTP/2 due to multiplexing). In this case, Dapr should probably show a warning indicating that this method should be invoked using HTTP/2.

## Completion Checklist

- [ ] Code changes
  - [ ] New APIs implemented in the interface in components-contrib
  - [ ] Update existing component(s)
  - [ ] Add runtime support for the new APIs
- [ ] Tests added
  - [ ] Expand conformance tests in components-contrib
  - [ ] Expand E2E tests
  - [ ] ~~Integration tests~~ - Integration tests are out of scope unless we offer an implementation that uses an in-memory (or local) backend
- [ ] SDK changes
- [ ] Documentation

