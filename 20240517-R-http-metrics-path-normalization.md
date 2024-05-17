# HTTP Metrics Path Normalization

* Author(s): @nelson-parente @jmprusi 
* State: Ready for Implementation
* Updated: 2024-05-17

## Overview

This is a design proposal to implement a new opt-in API for path normalization within Dapr HTTP metrics. By enabling path normalization users can define paths that will be normalized without being at risk of unbounded path cardinality and other issues that motivate the introduction of the low cardinality mode in Dapr. This will enable users to have more meaningful and manageable metrics in a controlled way. 

## Background

In [#6723](https://github.com/dapr/dapr/issues/6723), Dapr reduced the cardinality of its HTTP metrics in order to address memory issues users reported and restrain unbounded path cardinality which posed as a security threat. This change introduced two cardinality modes (high/low) controlled by the `increasedCardinality` flag.

The caveat with low cardinality is that it dropped paths since they were one of the sources for the high cardinality. While this is a reasonable approach it leads in the loss of important data needed for monitoring, performance analysis, and troubleshooting. To address this we opened [#7719](https://github.com/dapr/dapr/issues/7719).

This proposal introduces an opt-in API that allows users to define the paths that matter the most, effectivelly normalizing metrics without reliying on regexes, which are known to be CPU-intensive.

With this API, users will be able to configure path normalization through a simple interface, providing the paths they care about and tailoring metrics to their specific requirements without compromising memory and security issues.

## Related Items

### Related issues 

Initial low cardinality issue: [#6723](https://github.com/dapr/dapr/issues/6723)
Issue related with low cardinality dropped metrics data:  [#7719](https://github.com/dapr/dapr/issues/7719)

## Expectations and alternatives

The proposed solution adds value to users observability without compromising security and memory usage. The API is designed to be simple to configure, allowing users to configure the paths they care about. We considered other regex-based solutions but these are known to be CPU-intensive and can lead to performance degradation.

## Implementation Details

### Solution

This proposal introduces an opt-in API for path normalization within Dapr HTTP metrics. The goal is to offer a way to normalize metrics without relying on CPU-intensive regexes.

```yaml
spec:
  metric:
    enabled: true
    http:
      increasedCardinality: true
      pathNormalization:
        enabled: true/false
        ingress:
        - /orders/:orderID/items/:itemID
        - /users/:userID
        - /categories/:categoryID/subcategories/:subCategoryID
        - /customers/:customerID/orders/:orderID
        egress:
        - /orders
        - /categories
        - /users
```

#### Features

- `pathNormalization.enabled` users can enable or disable path normalization through a straightforward boolean flag.
- `ingress/egress` users can specify paths for ingress and egress traffic.
- The path matching will use the same patterns as the Go standard library (see https://pkg.go.dev/net/http#hdr-Patterns), ensuring reliable and well-supported path normalization.
- Non-matched paths will be ignored, ensuring that cardinality doesn't grow uncontrolled.


This Path Normalization API empowers users that relly on the metrics and observability scrapped from low cardinality that will soon be the default, providing a controlled means to manage path cardinality. Those indifferent may opt for low cardinality, while legacy high cardinality mode remains available for alternative needs.

### Acceptance Criteria

- The Path Normalization API is successfully integrated into the Dapr runtime, allowing users to enable/disable path normalization via configuration.

## Completion Checklist

What changes or actions are required to make this proposal complete? Some examples:

- [ ] Implementation in daprd
- [ ] API documentation

