# Error Code Metrics

* Author(s): @jake-engelberg
* State: Ready for Implementation
* Updated: 2024-10-29

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
	recordErrorCodes: true # <-- new opt-in flag
	http:
		...
	...
...
```
From the user's perspective, the metric `dapr_error_code_count` could be split by `app-id`, `error_code`, and `error_type` like so:
```go
dapr_error_code_count{app_id="service-a", error_code="ERR_ACTOR_RUNTIME_NOT_FOUND", error_type="ACTOR_API"} 4
dapr_error_code_count{app_id="service-a", error_code="ERR_ACTOR_INVOKE_METHOD", error_type="ACTOR_API"} 12
dapr_error_code_count{app_id="service-b", error_code="ERR_ACTOR_RUNTIME_NOT_FOUND", error_type="ACTOR_API"} 55
dapr_error_code_count{app_id="service-b", error_code="ERR_DIRECT_INVOKE", error_type="SERVICE_INVOCATION_API"} 2
```
Error codes are NOT currently centrally defined in a single package/file and will be a future endeavor.

However, there are two main ways they are defined:

1. Defined as a struct in `predefined.go`, with the specific code as `APIError.tag`:
	```go
	ErrHealthNotReady  =  APIError{message: "dapr is not ready",  tag: "ERR_HEALTH_NOT_READY", httpCode: http.StatusInternalServerError,  grpcCode: grpcCodes.Internal}
	```
2. Defined, sometimes repeatedly, inline as primitive strings:
	```go
	if err != nil {
		msg := NewErrorResponse("ERR_INTERNAL", "failed to encode response as JSON: "+err.Error())
		respondWithData(w, http.StatusInternalServerError, msg.JSONErrorValue())
		log.Debug(msg)
		return
	}
	```

In the future, a structure similar to `APIError` will be used to align all errors to equivalent definition.

Additionally, these errors are printed generally as-is with `log.Debug(msg)`.
```go
func  (a *api)  onGetHealthz(w http.ResponseWriter, r *http.Request)  {
	if !a.readyStatus {
		msg := messages.ErrHealthNotReady
		respondWithError(w, msg)
		log.Debug(msg)
		return
	}
...
}
```

By utilizing a function for each type of error code implementation, metric recording would occur whenever error codes are used with the intention of being logged:
1. With retrieval of the `APIError` object:
	```go
	if err != nil {
		return nil, messages.ErrInternal.RecordAndGet().WithFormat(err)
	}
	```
	- *NOTE: This is important to maintain as the object is often used with `WithFormat()`.*

2. Or with usage of the inline primitive string, when no `APIError` object exists:
	```go
	if err != nil {
		return nil, status.Error(messages.RecordAndGet(messages.ErrInternal))
	}
	```
	- *NOTE: This would eventually be phased out and converted to option 1 above once an object exists.*

And the respective functions defined like so:
```go
// This will record the error as a metric and return the APIError
func (err APIError) RecordAndGet() APIError {
	diag.DefaultErrorCodeMonitoring.RecordErrorCode(err.tag)
	return err
}

// This will record the error as a metric and return the APIError string
func RecordAndGet(errorCode string) string {
	diag.DefaultErrorCodeMonitoring.RecordErrorCode(errorCode)
	return errorCode
}
```

With `RecordError()` recording OpenCensus metrics like other metric recorders found in `pkg/diagnostics`, and assigning the correct tag based on the code:
```go
func (m *errorCodeMetrics) RecordErrorCode(code string) {
	if m.enabled {
		_ = stats.RecordWithTags(
			m.ctx,
			diagUtils.WithTags(m.errorCodeCount.Name(), appIDKey, m.appID, errorCodeKey, code, errorTypeKey, GetErrorType(code)),
			m.errorCodeCount.M(1),
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
