# Components Healthcheck In Dapr

* Author(s): Deepanshu Agarwal (@DeepanshuA)
* State: Draft
* Updated: 07/20/2023

## Overview
As a user, I should be able to enquire the health of dapr system, which is possible currently - but this health check should also include health of connected dapr components as well.

## Tenets
1. It should be extensible i.e. tomorrow if any feature to check health of Actors etc. is required, it should not require a new endpoint.
2. It should not give any false positives or false negatives.

## Current Scenario
There are many components in Dapr which don't yet implement Ping. 
Earlier, Ping was made mandatory due to which it used to give false positive in various components. But, then it was made optional and only those components have Ping function, which properly implement it.
So, how to provide a correct answer to users, if we can't enquire all components?


## Moving Ahead
If we try to give a one-word Yes/No kind of answer to user for the Question: "Are all components healthy in my system?", that means that if he/she has some component(s), which don't yet implment Ping, then it can't be a single word answer "Yes" or "No". We definitely need to provide him/her the information that what all components couldn't be checked through this check, as they don't yet implement Ping function.
We should definitely have some Work Items to ensure that most Components start implementing Ping, but that doesn't mean that we can't go ahead with Components health check for now, we just need to ensure that we are providing a proper info about those components explicitly to user which are in his system but could not be verified via the health check.


## API Design

Most of the Approaches underneath work with a query parameter, in addition to `healthz` endpoint. Instead of, an additional endpoint.

Following are the possible responses for `healthz` API:

| HTTP | Response Codes | 
| -------- | -------- | 
| 204     | dapr is healthy     | 
| 500     | dapr is not healthy     | 

Hence, if dapr is healthy, then the response code changes, as per the enlisted case below.
If dapr is NOT healthy, then the compoenents Health check should anyways NOT be checked.


### Approach 1:
List only failed or those componnets for which Ping is not implemented. Create separate sections for "Failed" and "Ping Not Implemented".

HTTP Endpoint: http://localhost:3500/v1.0/healthz?components=true
GRPC Endpoint: GetComponentHealthAlpha1 (It will be separate endpoint for grpc)

**Case 1:** When ALL components in system implement Ping and are Healthy

Result:

Response code: 204 OK No Content
GRPC Response code (if implemented): 0 OK

**Case 2:** When ALL components in system implement Ping BUT some components have failed Ping check: Here we report only failed components and not passed components, we don't treat this API as a way to list components.

HTTP Response code: 500 Internal Server Error
GRPC Response code (if implemented): 2 UNKNOWN

```
{
    "componentsHealth": 
        "failedComponents": [
            {
                "componentName": "txnstore",
                "type": "state",
                "status": "NOT_OK",
                "message": "redis store: error connecting to redis at localhost:6379: dial tcp 127.0.0.1:6379: connect: connection refused"
            },
            {
                "componentName": "smsbinding",
                "type": "bindings",
                "status": "NOT OK",
                "message": "redis binding: error connecting to redis at localhost:6379: dial tcp 127.0.0.1:6379: connect: connection refused"
            }
        ]
}
```
**Case 3:** When SOME components in system implement Ping AND some components DON'T implement Ping, AND some components have failed Ping check as well: Here we report failed components and those components as well for which Ping is not implemented, we don't treat this API as a way to list components.

HTTP Response code: 500 Internal Server Error
GRPC Response code (if implemented): 2 UNKNOWN

```
{
    "componentsHealth": 
        "failedComponents": [
            {
                "componentName": "txnstore",
                "type": "state",
                "status": "NOT_OK",
                "message": "redis store: error connecting to redis at localhost:6379: dial tcp 127.0.0.1:6379: connect: connection refused"
            },
            {
                "componentName": "smsbinding",
                "type": "bindings",
                "status": "NOT OK",
                "message": "redis binding: error connecting to redis at localhost:6379: dial tcp 127.0.0.1:6379: connect: connection refused"
            }
        ],
        "pingNotImplemented": [
            {
                "componentName": "mockABC",
                "type": "pubsub",
            },
            {
                "componentName": "mockXYX",
                "type": "bindings",
            }
        ]
}
```

**Case 4:** When ANY component in system DOES NOT implement Ping: Here we report ALL those components for which Ping is not implemented.

HTTP Response code: 405 Method Not Allowed
GRPC Response code (if implemented): 12 UNIMPLEMENTED

```
{
    "componentsHealth": 
        "pingNotImplemented": [
            {
                "componentName": "txnstore",
                "type": "state",
            },
            {
                "componentName": "smsbinding",
                "type": "bindings",
            },
            {
                "componentName": "mockABC",
                "type": "pubsub",
            },
            {
                "componentName": "mockXYX",
                "type": "bindings",
            }
        ]
}
```
**Case 5:** When SOME components in system implement Ping and whatever Compoenents implement Ping are Healthy: Here we report only those components for which Ping is not implemented.

HTTP Response code: 200 OK
GRPC Response code (if implemented): 0 OK

```
{
    "componentsHealth": 
        "pingNotImplemented": [
            {
                "componentName": "mockABC",
                "type": "pubsub",
            },
            {
                "componentName": "mockXYX",
                "type": "bindings",
            }
        ]
}
```

**Pros of this Approach:**
1. Only required components (failed or Ping not implemented) are listed in result.


**Cons of this Approach:**
1. Different umbrellas for "Failed Components" and "Ping Not Implemented" - doesn't give a consistent view and may require very custom parsing logic, if users have some downstream system to listen to this heartbeat.

### Approach 2:
Instead of listing under umbrellas, like "Failed components" or "Ping Not Implemented", list all components and report accordingly for them.

HTTP Endpoint: http://localhost:3500/v1.0/healthz?components=true
GRPC Endpoint: GetComponentHealthAlpha1 (It will be separate endpoint for grpc)
**Case 1:** When ALL components in system implement Ping and are Healthy

Result:

Response code: 204 OK No Content
GRPC Response code (if implemented): 0 OK
```
{
    "componentsHealth": 
        {
            "componentName": "txnstore",
            "type": "state",
            "status": "OK",
        },
        {
            "componentName": "smsbinding",
            "type": "bindings",
            "status": "OK",
        }
}
```

**Case 2:** When ALL components in system implement Ping BUT some/all components have failed Ping check:

HTTP Response code: 500 Internal Server Error
GRPC Response code (if implemented): 2 UNKNOWN

```
{
    "componentsHealth": 
        {
            "componentName": "txnstore",
            "type": "state",
            "status": "NOT_OK",
            "message": "redis store: error connecting to redis at localhost:6379: dial tcp 127.0.0.1:6379: connect: connection refused"
        },
        {
            "componentName": "smsbinding",
            "type": "bindings",
            "status": "NOT OK",
            "message": "redis binding: error connecting to redis at localhost:6379: dial tcp 127.0.0.1:6379: connect: connection refused"
        },
        {
            "componentName": "mockABC",
            "type": "pubsub",
            "status": "OK",
        },
        {
            "componentName": "mockXYX",
            "type": "bindings",
            "status": "OK",
        }
}
```
**Case 3:** When SOME components in system implement Ping AND some components DON'T implement Ping, AND some components have failed Ping check as well:

HTTP Response code: 500 Internal Server Error
GRPC Response code (if implemented): 2 UNKNOWN

```
{
    "componentsHealth": 
        {
            "componentName": "txnstore",
            "type": "state",
            "status": "NOT_OK",
            "message": "redis store: error connecting to redis at localhost:6379: dial tcp 127.0.0.1:6379: connect: connection refused"
        },
        {
            "componentName": "smsbinding",
            "type": "bindings",
            "status": "NOT OK",
            "message": "redis binding: error connecting to redis at localhost:6379: dial tcp 127.0.0.1:6379: connect: connection refused"
        },
        {
            "componentName": "mockABC",
            "type": "pubsub",
            "status": "UNDEFINED",
            "errorCode": "ERR_PING_NOT_IMPLEMENTED"
        },
        {
            "componentName": "mockXYX",
            "type": "bindings",
            "status": "OK",
        }
}
```

**Case 4:** When ANY component in system DOES NOT implement Ping: Here we report ALL those components for which Ping is not implemented.

HTTP Response code: 405 Method Not Allowed
GRPC Response code (if implemented): 12 UNIMPLEMENTED

```
{
    "componentsHealth": 
        {
            "componentName": "txnstore",
            "type": "state",
            "status": "UNDEFINED",
            "errorCode": "ERR_PING_NOT_IMPLEMENTED"
        },
        {
            "componentName": "smsbinding",
            "type": "bindings",
            "status": "UNDEFINED",
            "errorCode": "ERR_PING_NOT_IMPLEMENTED"
        },
        {
            "componentName": "mockABC",
            "type": "pubsub",
            "status": "UNDEFINED",
            "errorCode": "ERR_PING_NOT_IMPLEMENTED"
        },
        {
            "componentName": "mockXYX",
            "type": "bindings",
            "status": "UNDEFINED",
            "errorCode": "ERR_PING_NOT_IMPLEMENTED"
        }
}
```
**Case 5:** When SOME components in system implement Ping and whatever Compoenents implement Ping are Healthy:

HTTP Response code: 200 OK
GRPC Response code (if implemented): 0 OK

```
{
    "componentsHealth":
        {
            "componentName": "txnstore",
            "type": "state",
            "status": "OK",
        },
        {
            "componentName": "smsbinding",
            "type": "bindings",
            "status": "OK",
        },
        {
            "componentName": "mockABC",
            "type": "pubsub",
            "status": "UNDEFINED",
            "errorCode": "ERR_PING_NOT_IMPLEMENTED"
        },
        {
            "componentName": "mockXYX",
            "type": "bindings",
            "status": "UNDEFINED",
            "errorCode": "ERR_PING_NOT_IMPLEMENTED"
        }
}
```
**Pros of this Approach:**
1. Consistent view of result.


**Cons of this Approach:**
1. May look repetitive, and user may need to search specifically for failed components, when components used  list would be huge.

### Approach 3:
Don't list under umbrellas, like "Failed components" or "Ping Not Implemented", but also don't enlist Healthy componetnts in response.

HTTP Endpoint: http://localhost:3500/v1.0/healthz?components=true
GRPC Endpoint: GetComponentHealthAlpha1 (It will be separate endpoint for grpc)
**Case 1:** When ALL components in system implement Ping and are Healthy

Result:

Response code: 204 OK No Content
GRPC Response code (if implemented): 0 OK

**Case 2:** When ALL components in system implement Ping BUT some/all components have failed Ping check: Report only failed components:

HTTP Response code: 500 Internal Server Error
GRPC Response code (if implemented): 2 UNKNOWN

```
{
    "componentsHealth": 
        {
            "componentName": "txnstore",
            "type": "state",
            "status": "NOT_OK",
            "message": "redis store: error connecting to redis at localhost:6379: dial tcp 127.0.0.1:6379: connect: connection refused"
        },
        {
            "componentName": "smsbinding",
            "type": "bindings",
            "status": "NOT OK",
            "message": "redis binding: error connecting to redis at localhost:6379: dial tcp 127.0.0.1:6379: connect: connection refused"
        }
}
```
**Case 3:** When SOME components in system implement Ping AND some components DON'T implement Ping, AND some components have failed Ping check as well: Report Failed and Ping Not IMplemented components:

HTTP Response code: 500 Internal Server Error
GRPC Response code (if implemented): 2 UNKNOWN

```
{
    "componentsHealth": 
        {
            "componentName": "txnstore",
            "type": "state",
            "status": "NOT_OK",
            "message": "redis store: error connecting to redis at localhost:6379: dial tcp 127.0.0.1:6379: connect: connection refused"
        },
        {
            "componentName": "smsbinding",
            "type": "bindings",
            "status": "NOT OK",
            "message": "redis binding: error connecting to redis at localhost:6379: dial tcp 127.0.0.1:6379: connect: connection refused"
        },
        {
            "componentName": "mockABC",
            "type": "pubsub",
            "status": "UNDEFINED",
            "errorCode": "ERR_PING_NOT_IMPLEMENTED"
        }
}
```

**Case 4:** When ANY component in system DOES NOT implement Ping: Here we report ALL those components for which Ping is not implemented.

HTTP Response code: 405 Method Not Allowed
GRPC Response code (if implemented): 12 UNIMPLEMENTED

```
{
    "componentsHealth": 
        {
            "componentName": "txnstore",
            "type": "state",
            "status": "UNDEFINED",
            "errorCode": "ERR_PING_NOT_IMPLEMENTED"
        },
        {
            "componentName": "smsbinding",
            "type": "bindings",
            "status": "UNDEFINED",
            "errorCode": "ERR_PING_NOT_IMPLEMENTED"
        },
        {
            "componentName": "mockABC",
            "type": "pubsub",
            "status": "UNDEFINED",
            "errorCode": "ERR_PING_NOT_IMPLEMENTED"
        },
        {
            "componentName": "mockXYX",
            "type": "bindings",
            "status": "UNDEFINED",
            "errorCode": "ERR_PING_NOT_IMPLEMENTED"
        }
}
```
**Case 5:** When SOME components in system implement Ping and whatever Compoenents implement Ping are Healthy:

HTTP Response code: 200 OK
GRPC Response code (if implemented): 0 OK

```
{
    "componentsHealth":
        {
            "componentName": "mockABC",
            "type": "pubsub",
            "status": "UNDEFINED",
            "errorCode": "ERR_PING_NOT_IMPLEMENTED"
        },
        {
            "componentName": "mockXYX",
            "type": "bindings",
            "status": "UNDEFINED",
            "errorCode": "ERR_PING_NOT_IMPLEMENTED"
        }
}
```
**Pros of this Approach:**
1. Consistent view of result and only Required components are listed.


**Cons of this Approach:**
1. May look repetitive for Failed and Ping Not implemented components, and user may need to search specifically for failed components, when components list for "Ping Not Implemented" would be huge.

### Approach XYZ:

There can be other approaches, with different kind of endpoints. Or a dedicated endpoint, like: `http://localhost:3500/v1.0-alpha1/healthz/component`


### RECOMMENDED APPROACH
Approach 3, as it seems to be a Hybrid of Approach 1 and Approach 2. It only reports required components and keeps a consistent view.
And, to implement only http endpoint.

### All Approaches with Question:
Should we support query for a partular component Health check?
I don't think that is required specifically, as of now.
It is more about overall health of the system.

### What Building Blocks will be covered in 1.12 ?
- State
- Pubsub
- Bindings
- Secrets
- Configuration Store

## Open Questions
1. Is it only for kubernetes users? Is it only needed for http endpoint? Or we should cover gRPC endpoint as well?
2. 
