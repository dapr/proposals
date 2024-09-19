# Resiliency Policy Error Code Retries

* Author(s): Anton Troshin (@antontroshin), Taction (@taction)
* Updated: 2024-09-18

## Overview

This is a design proposal to provide additional functionality for Dapr Resiliency Policy Retries to be able to enforce policy only on specific response error codes.
It only focuses on the `retries` (https://docs.dapr.io/operations/resiliency/policies/#retries) part of the policy.

## Background

In some applications, some status codes may be used to indicate the business error, and retrying the operation might not be necessary or otherwise desirable.
Customizing retry behavior will allow a more granular way to handle error codes that suit each use case.
Currently, all errors are retried when the policy is applied.
Some status codes are not retryable, and subsequent calls will result in the same error. Avoiding these retry calls will reduce the overall number of requests, traffic, and errors.

## Related Items

https://github.com/dapr/dapr/issues/6683
https://github.com/dapr/dapr/issues/6428
https://github.com/dapr/dapr/issues/7697

PR:
https://github.com/dapr/dapr/pull/7132

Docs:
https://github.com/dapr/docs/issues/4254
https://github.com/dapr/docs/issues/3859

## Expectations and alternatives

* What is in scope for this proposal?
  - HTTP and gRPC Service Invocation, direct and proxied
  - Bindings
  - Pub/Sub

## Implementation Details

### Design

Add a new object field to the `retries` policy Spec to allow the user to specify the status codes that should be retried.
Separate fields for HTTP and gRPC. The new fields should be optional and will default to the existing behavior, which is to retry on all errors.

### Example 1:
In this example, the retry policy will retry **_only_** on HTTP 500 and HTTP status code range 502-504 (inclusive) and gRPC status code range 2-4 (inclusive).
The rest of the status codes will not be retried.

```yaml
apiVersion: dapr.io/v1alpha1
kind: Resiliency
metadata:
  name: myresiliency
scopes:
  - app1
spec:
  policies:
    retries:
      pubsubRetry:
        policy: constant
        duration: 5s
        maxRetries: 10
        matching:
          httpStatusCodes: "500,502-504"
          gRPCStatusCodes: "2-4"
```

### Example 2:
In this example, the retry policy will retry **_only_** on gRPC status code range 1-15 (inclusive).
However, this policy will not apply to the HTTP status codes, and they will be retried according to the default behavior, which is to retry on all errors.

```yaml
apiVersion: dapr.io/v1alpha1
kind: Resiliency
metadata:
  name: myresiliency
scopes:
  - app1
spec:
  policies:
    retries:
      pubsubRetry:
        policy: constant
        duration: 5s
        maxRetries: 10
        matching:
          gRPCStatusCodes: "1-15"
```

### Acceptable Values
The acceptable values are the same as the ones defined in the [HTTP Status Codes](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status) and [gRPC Status Codes](https://grpc.io/docs/guides/status-codes/) documentation.

- HTTP: from 100 to 599
- gRPC: from 1 to 16

### Setting Format
Both the `httpStatusCodes` and `gRPCStatusCodes` fields are of type string and optional and can be set to a comma-separated list of status codes and/or ranges of status codes.
The range must be in the format `<start>-<end>` (inclusive). Having more than one dash in the range is not allowed.

### Parsing the configuration

The configuration values will be first parsed as comma-separated lists.
Each entry in the list will be then parsed as a single status code or a range of status codes.
Invalid entries will be logged and the application will fail to start.

Example:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Resiliency
metadata:
  name: myresiliency
scopes:
  - app1
spec:
  policies:
    retries:
      pubsubRetry:
        policy: constant
        duration: 5s
        maxRetries: 10
        matching:
          httpStatusCodes: "500,502-504,15,404-405-500,-1,0,"
```
The steps to parse the configuration are:
1. Split the `httpStatusCodes` configuration string `"500,502-504,15,404-405-500,-1,0,"` by the comma character resulting in the following list: `["500", "502-504", "15", "404-405-500", "-1", "0"]` ignoring the empty strings.
2. For each entry in the list, parse it as a single status code or a range of status codes.
3. If the entry is a single status code, add it to the list of status codes to retry.
4. If the entry is a range of status codes (each field for the relevant HTTP or gRPC status codes), add all the status codes in the range to the list of status codes to retry.
- 500 is **valid** code for HTTP
- 502-504 **valid** range of codes for HTTP
- 15 is **invalid** code for HTTP, error logged and application will fail to start
- 404-405-500 is **invalid** range contains more than one dash, error logged and application will fail to start
- -1 is ignored is **invalid** code for HTTP, error logged and application will fail to start
- 0 is ignored is **invalid** code for HTTP, error logged and application will fail to start

### Acceptance Criteria

Integration and unit tests will be added to verify the new functionality.

## Completion Checklist

* Code changes
* Tests added (e2e, unit)
* Documentation
