# Caching: State Stores as Cache

* Author(s): Bernd Verst
* State: Ready for Implementation
* Updated: 2022-11-18

## Overview

Support caching in Dapr by allowing each existing state store with TTL feature support to act as a cache.

Existing state store components can be reused as cache. When a request to the state store component does not exist in the cache (defined as another referenced state store component) or the TTL has expired Dapr will read the key from the underlying state store source and write this value with the appropriate TTL to the cache (state store).

## Background

Today every Dapr State Store `Get` requests causes a request to the server. This is especially problematic in cases where data does not change frequently, but is accessed a lot. Several Dapr community users have assumed that the Dapr sidecar automatically caches data (and potentially monitors the underlying data source) which is not the case.

## Related Items

### Related proposals (to be completed)

Links to proposals that are related to this (either due to dependency, or possibly because this will replace another proposal)

### Related issues (to be completed)

Please link to any issues that this proposal is related to, for example, are there existing bugs filed in various Dapr repositories that this will affect?


## Expectations and alternatives (to be completed)

* What is in scope for this proposal?
* What is deliberately *not* in scope?
* What alternatives have been considered, and why do they not solve the problem?
* Are there any trade-offs being made? (space for time, for example)
* What advantages / disadvantages does this proposal have? 

## Implementation Details

### Design

New state store metadata format which introduces the `cache` property.

```
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: <NAME>
  namespace: <NAMESPACE>
spec:
  type: state.<TYPE>
  version: v1
  cache:
    - state: <COMPONENT-NAME-REF>
    - ttlInSeconds: 15s
  metadata:
  - name:<KEY>
    value:<VALUE>
  - name: <KEY>
    value: <VALUE>
```

Only components implementing the TTL feature (verified via TTL feature key in state store capabilities) can act as state store cache.

If a cache property is configured:
- State Store Get Request:
  - Perform Get Request against Cache, return value is present
  - If value is not present:
    1. Perform Get request against original state store
    2. Perform Set Request against Cache for the key and apply default TTL for expiry (or apply TTL read from original state store item)
    3. Return the value.
- State Store Set Request:
  1. Perform Set Request against original State Store.
  2. Perform Set Request against Cache, setting default TTL (or TTL from metadata)

If a Set Request cannot be applied to the Cache, the cache should be invalidated for this key, but the request should succeed.
If any underlying Get or Set Request performed against the underlying state store fails, the whole operation should fail.

### Feature lifecycle outline

The additionally component metadata parameter acts as an explicit opt-in for this feature. Whether the cache feature is used or not the Dapr user experience should not change.

### Acceptance Criteria

How will success be measured? 

* Existing conformance tests pass even with cache layer enabled

## Completion Checklist

What changes or actions are required to make this proposal complete? Some examples:

* Code changes
* Tests added (e2e, unit)
* Documentation
* No SDK changes

