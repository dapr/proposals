# HTTP Metrics Path Normalization

* Author(s): @nelson-parente @jmprusi 
* State: Ready for Implementation
* Updated: 2024-05-17

## Overview

This is a design proposal to implement a new opt-in API for path normalization within Dapr HTTP metrics. By enabling path normalization users can define paths that will be normalized without being at risk of unbounded path cardinality and other issues that motivate the introduction of the low cardinality mode in Dapr. This will enable users to have more meaningful and manageable metrics in a controlled way. 

## Background

In [#6723](https://github.com/dapr/dapr/issues/6723), Dapr reduced the cardinality of its HTTP metrics in order to address memory issues users reported and restrain unbounded path cardinality which posed as a security threat. This change introduced two cardinality modes (high/low) controlled by the `increasedCardinality` flag.

The caveat with low cardinality is that it dropped paths since they were one of the sources for the high cardinality. While this is a reasonable approach, it leads in the loss of important data needed for monitoring, performance analysis, and troubleshooting. To address this, we opened [#7719](https://github.com/dapr/dapr/issues/7719).

This proposal introduces an opt-in API that allows users to define the paths that matter the most, effectively normalizing metrics without relying on regex's, which are known to be CPU-intensive.

With this API, users will be able to configure path normalization through a simple interface, providing the paths they care about and tailoring metrics to their specific requirements without compromising memory and security issues.

## Related Items

### Related issues 

Initial low cardinality issue: [#6723](https://github.com/dapr/dapr/issues/6723)
Issue related with low cardinality dropped metrics data: [#7719](https://github.com/dapr/dapr/issues/7719)

## Expectations and alternatives

The proposed solution adds value to users' observability without compromising security and memory usage. The API is designed to be simple to configure, allowing users to configure the paths they care about. We considered other regex-based solutions but these are known to be CPU-intensive and can lead to performance degradation.

## Implementation Details

### Solution

This proposal introduces an opt-in API for path normalization within Dapr HTTP metrics. The goal is to offer a way to normalize metrics without relying on CPU-intensive regex's.

```yaml
spec:
  metric:
    enabled: true
    http:
      increasedCardinality: true
      pathNormalization:
        ingress:
        - /orders/{orderID}/items/{itemID}
        - /users/{userID}
        - /categories/{categoryID}/subcategories/{subCategoryID}
        - /customers/{customerID}/orders/{orderID}
        egress:
        - /orders
        - /orders/{orderID}/items/{itemID}
        - /categories
        - /users
```

##### Examples

Examples of how the Path Normalization API can be used to normalize metrics. The examples compare the metric `dapr_http_server_request_count` with the possible configuration combinations: low and high cardinality, with and without path normalization.

- Low Cardinality Without Path Normalization


```yaml
http:
  increasedCardinality: false
```

```
dapr_http_server_request_count{app_id="ping",method="InvokeService/ping",status="200"} 5
```
- Low Cardinality With Path Normalization

```yaml
http:
  increasedCardinality: false
  pathNormalization:
    ingress:
    - /orders/{orderID}
    egress:
    - /orders/{orderID}
```

```
dapr_http_server_request_count{app_id="ping",method="GET",path="/orders/{orderID}",status="200"} 4
dapr_http_server_request_count{app_id="ping",method="GET",path="_",status="200"} 1
```

- High Cardinality Without Path Normalization

```yaml
http:
  increasedCardinality: true
```

```
dapr_http_server_request_count{app_id="ping",method="GET",path="/items/123456",status="200"} 1
dapr_http_server_request_count{app_id="ping",method="GET",path="/orders/1234",status="200"} 1
dapr_http_server_request_count{app_id="ping",method="GET",path="/orders/12345",status="200"} 1
dapr_http_server_request_count{app_id="ping",method="GET",path="/orders/123456",status="200"} 1
dapr_http_server_request_count{app_id="ping",method="GET",path="/orders/1234567",status="200"} 1
```

- High Cardinality With Path Normalization

```yaml
http:
  increasedCardinality: true
  pathNormalization:
    ingress:
    - /orders/{orderID}
    egress:
    - /orders/{orderID}
```

```
dapr_http_server_request_count{app_id="ping",method="GET",path="/items/123456",status="200"} 1
dapr_http_server_request_count{app_id="ping",method="GET",path="/orders/{orderID}",status="200"} 4
```
#### Features

- `pathNormalization.ingress/pathNormalization.egress` users can specify paths for ingress and egress path matching.

The path matching will use the same patterns as the Go standard library (see https://pkg.go.dev/net/http#hdr-Patterns), ensuring reliable and well-supported path normalization.

When the increasedCardinality flag is set to false (default in 1.14), non-matched paths are transformed into a catch-all bucket to control and limit cardinality, preventing unbounded growth. On the other hand, when increasedCardinality is true, non-matched paths are passed through as they normally would be, allowing for potentially higher cardinality but preserving the original path data. This is the main difference in the feature behavior when used with low versus high cardinality.

This Path Normalization API empowers users that rely on the metrics and observability scrapped from low cardinality that will soon be the default, providing a controlled means to manage path cardinality. Those indifferent may opt for low cardinality, while legacy high cardinality mode remains available for alternative needs.

### Acceptance Criteria

- The Path Normalization API is successfully integrated into the Dapr runtime, allowing users to enable/disable path normalization via configuration.

## Completion Checklist

- [ ] Implementation in daprd
- [ ] API documentation
- [ ] Integration, E2E tests

