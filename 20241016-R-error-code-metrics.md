# Error Code Metrics

* Author(s): @jake-engelberg
* State: Ready for Implementation
* Updated: 2024-09-16

## Overview

This is a design proposal to implement a new opt-in set of metrics for counts of [Dapr API error codes](https://docs.dapr.io/reference/api/error_codes/).

When opted-in, in tandem with the current behavior, where the errors are logged, a new metric `dapr_error_code_count` would provide the # of times an error code has been thrown for each `app-id`. This opt-in is independent from the `debug` log mode and thus can be used separately.

## Background

Currently, error codes are only visible as logs when Dapr runs in `debug` log mode, which is impractical for both production apps and observability. 

Additionally, the error message is a field in JSON response body as such:
```
{
	...
    "errorCode": "ERR_STATE_GET",
    ...
}
```
...which means observation of these codes requires increased log noise via the `debug` flag, as well as parsing the logs to find this field. There also wouldn't be automatic data retention, so logs would have to be manually stored to provide a historical view.


Dapr uses Prometheus by default, and so having these errors as Prometheus metrics will allow querying, aggregation, and data retention of the codes.

## Related Items

Initial Error Codes Proposal: [proposals/#21](https://github.com/dapr/proposals/pull/21)

Error Codes Implementation Parent: [dapr/#6068](https://github.com/dapr/dapr/issues/6068)

## Expectations and alternatives

### Expectations
- When opted-in, it would add up to  `N * app`  (N = number of error codes) metrics for scraping, today that's around `52 * app`.
- If a component-based error, codes could repeat for multiple components used by the same app, and wouldn't be differentiated by the metrics, which only provide the `app-id`.
- Many logged error codes are currently defined in [pkg/messages/predefined.go](https://github.com/dapr/dapr/blob/master/pkg/messages/predefined.go), but not all. Implementation requires case-by-case analysis of usage of error codes in that uppercase format `ERR_NAME_TAG`.

### Alternatives
- One alternative solution which retains error codes as logs only would be to change the level of logs containing these error codes from `debug` to `info`. However, this increases noise and raises logs which may be unhelpful at `info` level.
- Another alternative solution would be to log error codes separately with opt-in functionality, but this achieves the same result as adding metrics, but increase log noise and doesn't introduce a simple way to actually work with the errors.

## Implementation Details

### Solution

The feature is enabled via an opt-in flag within Dapr HTTP metrics (disabled by default). The goal of the opt-in functionality is to prevent the increased cost of additional metrics if they aren't being used.

```yaml
spec:
  metric:
    enabled: true
    http:
      errorCodeMetrics: true
```
From the user's perspective, the metric `dapr_error_code_count` could be split by `app-id` and `error_code` like so:
```go
dapr_error_code_count{app_id="service-a", error_code="ERR_ACTOR_RUNTIME_NOT_FOUND"} 4
dapr_error_code_count{app_id="service-a", error_code="ERR_ACTOR_INVOKE_METHOD"} 12
dapr_error_code_count{app_id="service-b", error_code="ERR_ACTOR_RUNTIME_NOT_FOUND"} 55
dapr_error_code_count{app_id="service-b", error_code="ERR_DIRECT_INVOKE"} 2
```

Error codes are currently defined as a struct, with the specific code as `APIError.tag`:
```go
ErrHealthNotReady  =  APIError{message: "dapr is not ready",  tag: "ERR_HEALTH_NOT_READY", httpCode: http.StatusInternalServerError,  grpcCode: grpcCodes.Internal}
```

...and are currently implemented case-by-case across Dapr, but generally as-is with `log.Debug(msg)`. Metric recording would occur in tandem with the log statement:
```go
func  (a *api)  onGetHealthz(w http.ResponseWriter, r *http.Request)  {
	if !a.readyStatus {
		msg := messages.ErrHealthNotReady
		respondWithError(w, msg)
		log.Debug(msg)
		ErrorCodeMetrics.RecordError(msg.tag)
		return
	}
...
}
```

With `RecordError()` simply recording OpenCensus metrics like other metric recorders found in `pkg/diagnostics`:
```go
func (m *errorCodeMetrics) RecordError(errorCode) {
	if m.enabled {
		_ = stats.RecordWithTags(
			m.ctx,
			diagUtils.WithTags(m.errorCodesCount.Name(), appIDKey, m.appID, errorCodeKey, errorCode),
			m.errorCodesCount.M(1),
		)
	}
}
```

### Acceptance Criteria

- The error code metrics recorder is successfully integrated into the Dapr runtime, allowing users to enable/disable the recording of dapr error codes via configuration.

## Completion Checklist

- [ ] Implementation in daprd
- [ ] API documentation
- [ ] Integration, E2E tests
