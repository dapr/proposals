# Centralized Cache Building Block
- Author: Whit Waldo (@whitwaldo)
- Updated: 2025-01-05

## Overview
This is the first of [many proposals](https://github.com/dapr/dapr/issues/7339) to overhaul the state building block. Rather than continue to iterate on the existing state building block and potentially disrupt existing users of those capabilities, I propose a clean break towards purpose-specific second-generation State Management building blocks in that the current generation can continue to be maintained for the foreseeable future, but that new work is focused on these newer blocks that seek less to adopt a kitchen-sink approach and instead are narrowly tailored for their use cases and purposes.

This proposal focuses on the implementation of a centralized cache store. While both this proposal and the existing state API are both designed around key/value stores, the proposed shape of this API different from the existing state store in subtle, but helpful ways and makes for some low-hanging fruit to get the ball rolling on these replacement opportunities. As it's a clean break from past decisions, it has the benefit of being a fresh start that favors a concise, simple API that addresses cache-specific functionality and nothing more. Should a developer want something more from the API such as an ability to list existing keys or query the values with some constraint, they're encouraged to turn to other specialized stores for that functionality such as my proposed rethink of a [Key/Value Store](https://github.com/dapr/dapr/issues/7338) or the propsoed [Document Store](https://github.com/dapr/dapr/issues/5146), respectively.

## What's the purpose of a cache?
As this is a purpose-built implementation, I wanted to take a moment to specifically call out the problem this block seeks to provide a solution for.

A cache should temporarily store some data (typically something not too large) that's associated with a key for some specified amount of time. While there are variations of the concept that distribute the cache across multiple points of failure (introducing the CAP theorem trade-off of limiting consistency in favor of availability) or that implement a local in-memory cache accessible to perhaps just a single class, this proposal seeks to augment and utilize Dapr's rich existing infrastructure to offer these caching capabilities for a single centralized provider. In other words, this proposal is specifically a "Centralized Cache" proposal and not one trying to provide a distributed cache via the sidecar runtime (though this could just as easily be the focus of another specialized state proposal).

## Why not use the existing state store?

Again, this is the first of many more targeted proposals, and as such bears some of the greatest resemblance to what we already have, but it does differ in subtle ways at the API level (addressing issues raised in the past at the SDK level) and targets a narrower set of valid providers. It makes sense to me that the existing state store be rebranded as a general purpose store that's still more than valid for broad generalized state management support, but that this block be positioned as the first of many specialized and more specifically built solutions for common scenarios. 

Here, we're seeking to target an ever-so-slightly diferent piece of the key/value store pie in that we're targeting those developer who expect to have small chunks of data expire over time. They know the keys they're intending to engage with they have specific ideas around the expiration strategies they want to exploint, meaning that this block will not support the [list operations](https://github.com/dapr/proposals/pull/61) offered in a separate proposal.

Caches are often designed for rapid retrieval and as such are often hosted in-memory, meaning that developers aren't expecting to store large values alongside their keys. As such, this proposal doesn't support streaming operations but will instead set and return whole values and the documentation should call out recommendations as to the maximum size of values being stored.

Similarly, there's no need for ETag support or other consistency operations. While that might make sense in the context of another key/value store scenario or a document store (e.g. scenarios where the values are potentially quite large), it simply doesn't fit with the narrow API envisioned for this block. Transactions will not be supported as they aren't common to cache implementations as a whole and as such, this will not be a suitable foundation for actors or workflows to build atop of.

So what's left? Again, a narrowly defined state store that is compatible with the Dapr resiliency capabilities, but which offers the ability to get and set values for known keys supporting both sliding and absolute expiration, and precious little more.

## Component YAML
This component is expected to have similar attributes to existing state stores with some variation:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: cache
spec:
  type: centralizedcache.<providerName>
  version: v1
  metadata:
    # Various properties necessary for component registration
```

## Implementation Overview
There are a few guiding principles I've stuck to while thinking through the shape of this proposal (iterated on in [this issue](https://github.com/dapr/dapr/issues/7886) to date):
- All SDK operations should be implemented asynchronously in a way that minimize operations with the sidecar.
- The key should be represented as a `string`, but the value should be persisted as a `byte[]` as it's not going to be separately queried and would be retrieved in full. This also leaves as an exercise to the developer or SDK precisely how serialization is handled (e.g. serialization strategy, formatting, encryption, compression, encoding, etc.). Ideally, ths various SDK maintainers can develop a uniform approach to this (a POC exists in the .NET SDK [here](https://github.com/dapr/dotnet-sdk/pull/1378)) so that data encoded in C# is easily decoded from any other language SDKs.
- No additional metadata should be stored alongside the value (e.g. minimize multiple operations), but should instead be bundled with the payload if necessary. This store is simply an abstraction for storing keys with values and facilitating storage and retrieval - nothing more.. from the SDK side.

That said, from an implementation perspective, I leave it as an open question in this proposal whether we want to also persist the sliding timespan in an adjacent key (e.g. `<key>-sliding`) so accommodate sliding expirations where it might not be implemented in the underlying provider. Further, while my overarching goal is to minimize having functionality implemented by Dapr and instead heavily rely on the native providers capabilities alone, the distributed scheduler could provide this sliding and absolute TTL support where underlying providers don't have such support, broadening the number of providers that could be used with our cache abstraction, so there's some minimal wiggle room on that. That said, such variations should be minimal and undertaken only where the support coincides with the goals of this block and not just to add more functionality just because, as that doesn't align with the purpose-driven philosophy.

### Expiration Strategy
The expiry should be expressed in [ISO 8601 duration format](https://en.wikipedia.org/wiki/ISO_8601#Durations) and will reflect both/either a sliding and/or an absolute expiration as desired. A sliding expiration puts a bounds on the inactivity of any given state before it's removed, but resets this expiration value upon access (e.g. a value is set at noon with a sliding expiration of 2 hours meaning it expires at 2:00 PM. At 1:00 PM, the value is accessed, meaning that the sliding expiration is reset once again to 2 hours later, expiring now at 3:00 PM). An absolute expiration indicates the optional maximum period a value should be cached and overrides any sliding expiration value, when set. For example here, if the absolute expiration is specified at an hour, the value will be removed and not returned even if the sliding expiration would not have yet elapsed.

### gRPC APIs
In the Dapr gRPC APIs, we'd extend the `runtime.v1.Dapr` service to add new methods:

| Note: APIs will have Alpha1 suffixed to the type names while in preview

| Note: Any authentication behaviors are maintained in the component YAML configuration

```proto
//Existing Dapr service
service Dapr {
    // Sets or updates a cache entry
    rpc SetCentralizedCacheAlpha1(SetCentralizedCacheRequest) returns (SetCentralizedCacheResponse) {}

    // Retrieves a previously set and unexpired cache entry
    rpc GetCentralizedCacheAlpha1(GetCentralizedCacheRequest) returns (GetCentralizedCacheResponse) {}

    // Refreshes the sliding expiration for an existing cache entry
    rpc RefreshCentralizedCacheAlpha1(RefreshCentralizedCacheRequest) returns (RefreshCentralizedCacheResponse) {}

    // Removes an existing cache entry
    rpc RemoveCentralizedCacheAlpha1(RemoveCentralizedCacheRequest) returns (RemoveCentralizedCacheResponse) {}
}

// Sets a cache entry in the store
mesage SetCentralizedCacheRequest {
    // The value to persist
    google.protobuf.Any value = 1;
    // Presented in ISO 8601 duration format
    string slidingExpiration = 2;
    // Presented in ISO 8601 duration format
    optional string absoluteExpiration = 3;
    // Optional: Indicates whether an existing key and expiry information should be overwritten (true) or not (false) with the new value and expiry information. Defaults to true.
    optional boolean overwrite = 4;
}

// The expected response from an attempt to set a cache entry in the store
message SetCentralizedCacheResponse {
    // Indicates whether a key already exists - if `overwrite` was false on the request, this tells the caller that the new value was not persisted. If `overwrite` was true, this just tells the caller an overwrite occured.
    boolean exists;
}

// The payload used to retrieve a stored and unexpired cahe entry
message GetCentralizedCacheRequest {
    // The key being accessed
    string key;
}

// Represents the returned value and expiration properties for a given cache entry
message CentralizedCacheValue {
    // Contains the value of the specified key.
    byte[] value = 1;
    // ISO 8601 duration format, updated to reflect the latest value post-touch
    string slidingExpiration = 2;
    // ISO 8601 duration format
    optional string absoluteExpiration = 3;
}

// Response from an attempt to retrieve a stored and unexpired value
message GetCentralizedCacheResponse {
    // Nullable: If null, the value does not exist or is expired. Otherwise, contains the latest entry value and updated expiration values.
    CentralizedCacheValue data
}

// Used to refresh the sliding expiration value for a cache entry
message RefreshCentralizedCacheRequest {
    // The key to refresh the sliding expiration for
    string key
}

// Represents the returned expiration properties for a given cache entry, absent the entry value
message RefreshCentralizedCacheValue {
    // ISO 8601 duration format, updated to reflect the latest value post-touch
    string slidingExpiration = 2;
    // ISO 8601 duration format
    optional string absoluteExpiration = 3;
}

// Reflects whether a given key existed to be refreshed or not.
message RefreshCentralizedCacheResponse {
    // Nullable: If null, the value doesn't exist or is already expired. Otherwise, contains the updated expiration values for the cache entry.
    RefreshCentralizedCacheValue data;
}

// Used to remove the value for a specified key
message RemoveCentralizedCacheRequest {
    // The key to remove
    string key
}
```

### HTTP APIs
The HTTP APIs align with the shapes of the gRPC APIs.

| Note: The following URLs assume that `state` is actually used as a root path for the URL so this and other specialized stores continue to be regarded under the state umbrella.

#### Add a value to the specified key
POST `http://localhost:{daprPort}/v1.0-alpha1/state/centralizedcache/{key}
Payload:
```json
{
    'overwrite': boolean,
    'value': byte[],
    'slidingExpiration': string,
    'absoluteExpiration': string //Optional
}
```
Response:
```json
{
    'exists': boolean // True if there was already an existing entry before the operation, false if there is no existing entry
}
```

#### Try to retrieve the value associated with the given key
GET `http://localhost:{daprPort}/v1.0-alpha1/state/centralizedcache/{key}
Response:
```json
{
    'data': {
        'value': byte[],
        'absoluteExpiration': string,
        'slidingExpiration': string
    } | nullable
}
```

#### Refresh the cache entry
HEAD `http://localhost:{daprPort}/v1.0-alpha1/state/centralizedcache/{key}
```json
{
    'data': {
        'absoluteExpiration': string,
        'slidingExpiration': string
    } | nullable
}
```

#### Remove cache entry
DELETE `http://localhost:{daprPort}/v1.0-alpha1/state/centralizedcache/{key}`

| Note URLs will reflect the `/v1.0-alpha1/` version while in preview.

## SDK Example - .NET
The following reflects what the API would look like in the .NET SDK.

| Method Signature | Description |
| -- | -- |
| Task<bool> TryAddAsync(string key, ReadOnlyMemory<byte> value, CentralizedCacheOptions options, CancellationToken cancellationToken) | Adds the specified key and value to the store along with a sliding expiration represented as a `TimeSpan` (relative to now) and an optional absolute expiration also represented as a `TimeSpan` (relative to now) or a `DateTimeOffset` (absolute and converted by the SDK to a relative duration). |
| Task AddAsync(string key, ReadOnlyMemory<byte> value, CentralizedCacheOptions options, CancellationToken cancellationToken) | Adds the specified key and value to the store along with the expiration options provided in `TryAddAsync` and overwrites any existing information. |
| Task<bool> TryGetAsync(string key, out CacheEntry value, CancellationToken cancellationToken) | Attempts to retrieve the value for the specified key. If the key is available, returns true and populates the `out` value with the value and expiration properties. If the key is not available, returns false. |
| Task<bool> TryRefreshAsync(string key, out CacheProperties valuecancellationToken, CancellationToken cancellationToken) | Returns true if an unexpired key is found and false if the key is not found. If found, returns the expiration values as the `out` value. |
| Task TryRemoveAsync(string key, CancellationToken cancellationToken) | Attempts to remove the entry with the specified key from the store. |

Each of these methods supports a cancellation token because while it's not expected that the methods would take any substantial amount of time to complete (as that would be antithetical to the purpose of these component), it facilitates future-proofing in case the Dapr runtime supports cancellation tokens in the future, it makes the methods consistent with the expected shape of other C# async methods, it facilitates testing to enable users to simulate cancellation scenarios themselves and it allows for the operation to be canceled by the client in case of an unexpected timeout with call to the Dapr runtime.

## Implications
- Outside of augmenting documentation accordingly, no immediate implications of implementing this updated state store mechanism. In the longer run, as this and other state store blocks are added and mature, it might be worth revsiting completely deprecating the existing general-purpose state store, but that's well beyond the current horizon.

## Completion Checklist
[] Centralized cache API code
[] Tests added (e2e, unit)
[] SDK changes
[] Documentation