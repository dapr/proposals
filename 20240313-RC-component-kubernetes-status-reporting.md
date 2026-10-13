# Component Kubernetes Status Reporting

* Author(s): @joshvanl
* State: Review
* Updated: 2024-03-13

## Overview

This proposal outlines a mechanism for reporting and surfacing Component status health to users in Kubernetes.
The reported status will also detail the App IDs & Pods which are currently consuming that Component, and the initialisation status of each Pod.
Information is reported on the Component `status` sub-resource field on each Component resource.
Surfacing information in this way is useful for both humans and controllers to understand the health of a Component and take action if needed or fire alerts.
Kubernetes users are familiar with, and expect, the status of resources to be updated with the current state of the resource.

## Background

Currently, the only mechanism for a user to understand the status of a Component being loading into a Daprd in Kubernetes is by inspecting the logs of the Daprd process or relying on a third-party system (or the Dashboard) to provide this information.
Systems retrieve this information by querying the Daprd metadata API, however this port should remain private as it exposes sensitive data and is not practical to be used in all scenarios.
This mechanism is still problematic as this does not report the "observed generation" of the Component loaded, meaning a user will not know whether a Daprd has picked up the latest version of a Component spec or not.
This mechanism also relies on a pull based system, meaning it does not have real time information.

Resolving issues with loading Components always requires a user to inspect the logs of the Daprd process, which may be in crash-loop-backoff or not accessible to the user.

## Expectations and alternatives

The Dapr operator will be responsible for patching the `status` subresource field of Components with the initialization result from Daprds.
Daprds will report the status to the operator via the gRPC API server.
While Daprds _could_ update Component resources themselves, this is problematic
for 2 reasons:

1. Expanding the RBAC permissions of Daprds to allow them to patch Component status (and thus all Dapr enabled applications in Kubernetes), would mean an application could read all Components in their namespacing (breaking sever side scope filtering security posture), as well as allowing a malicious application to change configuration of other applications or replicas.
2. Daprds would need to have informers for the Component resource, creating a large resource burden on the Kubernetes API server as well as ballooning the Daprd process memory usage.

The Component status will only report the Initialization status and the observed generation of each App ID replica.
Active health checking can be incorporated into the status report with [this proposal](https://github.com/dapr/proposals/pull/34).

This proposal focuses on the Component resource, but can extended in future to be used for other resource types as well.

## Solution

Rough PoC can be found [here](https://github.com/dapr/dapr/commit/0effb939e9b4a6797bb3a2b665cd3d99e6bf10fe).

### Operator

#### API

The Operator gRPC API will be extended to include the new following RPC streaming method and messages.
This streaming RPC method will be served by the Operator API server.
When a Daprd replica begins a stream on this RPC, the operator will set the Pod `Init` condition status of every Component which is scoped for this App ID to `Unknown`.
Upon disconnection, the operator will also set the Pod `Init` condition status of every Component which is scoped for this App ID to `Unknown`.
See [Sentry](###sentry) for more information on how the operator will know the App ID and Pod Name of the connecting Daprd.

The operator will reject the request if it cannot determine the Pod name of the requester.
The operator will reject the request with a well-known `FailedPrecondition` error code if the receiving Operator is not the Operator Kubernetes controller elected leader.
If the operator looses leadership, it will close all streams from Daprds.
Upon a non-leader error, Daprds will then attempt to connect to the next Operator API server, see [Daprd](###daprd) for more information.

When status reporting requests are received by the operator on the stream by Daprds, the operator will update the relevant Component statuses with the status condition of the Pod.
This stream handler will _not_ check against observed generations and will patch "as is".
Reporting observed generation miss-match is handled by the [controller](####controller).

For privacy, all component statuses will be removed before sending them to clients through `ListComponents` and `ComponentUpdate`.

```proto
package dapr.proto.common.v1;

// Condition represents the status of a Condition.
enum ConditionStatus {
  UNKNOWN = 0;
  TRUE = 1;
  FALSE = 2;
}
```

```proto
package dapr.proto.operator.v1;

service Operator {
  // ComponentReport is a streaming RPC to report the status of a Component.
  rpc ComponentReport(stream ComponentReportRequest) returns (google.protobuf.Empty) {}
}

// ComponentReportRequest is the request to report component status.
message ComponentReportRequest {
  enum Type{
    // Not specified - use the default value.
    UNKNOWN = 0;
    // INIT indicates reporting the component initialization.
    INIT = 1;
  }

  // type is the type of the report.
  Type type = 1;

  // status is the status of the component.
  common.v1.ConditionStatus status = 2;

  // component_name is the name of the component.
  string component_name = 3;

  // observed_generation is the observed generation of the component.
  int64 observed_generation = 4;

  // reason is the optional machine readable reason for the status.
  // Typically empty if no error occurred.
  optional string reason = 5;

  // message is the optional human readable message for the status.
  // Typically empty if no error occurred.
  optional string message = 6;
}
```

### Controller

A controller will be added to the Operator which will be responsible for managing the status sub-resource of Component resources.
The controller will be reconciled on any Component or Dapr enabled Pod event.

On a Pod event, the controller will reconcile all Components which are scoped for the Pod's Dapr App ID.

On a Component sync, the controller will:

1. Ensure that all Pods that are scoped for the Component have at least an `Unknown` status condition under the appropriate App ID.
Any condition referencing a Pod that does not exist will be removed.

2. Update the `status.initializedAppIDs` and `status.notInitializedAppIDs` fields based on the conditions of each App ID consuming the Component.
All App IDs must be present in either field.
An App ID may only appear in the `status.initializedAppIDs` field if and only if all of the Pod replicas have a `True` status condition _and_ has the same `observed generation` as the Component metadata.
A Component will only have a `True` `Ready` condition if and only if the `status.notInitializedAppIDs` field is empty.

The Kubernetes API definitions can be tagged to allow for pretty print output
for kubectl:

```bash
$ kubectl get components
NAME        READY   InitializedAppIDs   NotInitializedAppIDs
mycomponent True    foo, bar
another     False   baz                 abc, def
foobar      True
```

The following is the proposed updated API definition for the Component status subresource:

```go
//+genclient
//+kubebuilder:subresource:status
//+kubebuilder:object:root=true

// Component describes an Dapr component type.
type Component struct {
  ...

  // Status of the Component.
  // This is set and managed automatically.
  // Read-only.
  // More info: https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#spec-and-status
  // +optional
  Status ComponentStatus `json:"status,omitempty"`
}

// ComponentStatus is the status for a Component resource.
type ComponentStatus struct {
  // Conditions is a list of conditions indicating the state of the Component.
  // Known condition types are (`Ready`).
  // +listType=map
  // +listMapKey=type
  // +optional
  Conditions []ComponentCondition `json:"conditions,omitempty"`

  // AppIDConditions is a list of conditions indicating the state of each app
  // ID consuming the Component.
  // +listType=map
  // +listMapKey=appID
  // +optional
  AppIDConditions []ComponentAppIDCondition `json:"appIDConditions,omitempty"`

  // InitializedAppIDs is a list of app IDs that have all initialized this
  // Component.
  // +optional
  InitializedAppIDs []string `json:"initializedAppIDs,omitempty"`

  // NotInitializedAppIDs is a list of app IDs that have not initialized this
  // Component due to an error.
  // +optional
  NotInitializedAppIDs []string `json:"notInitializedAppIDs,omitempty"`
}

// ComponentStatus is the status for a Component resource.
type ComponentCondition struct {
  // Type of the condition, known values are (`Ready`).
  Type ComponentConditionType `json:"type"`

  // Status of the condition, one of (`True`, `False`, `Unknown`).
  Status common.ConditionStatus `json:"status"`

  // LastTransitionTime is the timestamp corresponding to the last status
  // change of this condition.
  LastTransitionTime *metav1.Time `json:"lastTransitionTime,omitempty"`

  // Reason is a brief machine readable explanation for the condition's last
  // transition.
  // +optional
  Reason *string `json:"reason,omitempty"`

  // Message is a human readable description of the details of the last
  // transition, complementing reason.
  // +optional
  Message *string `json:"message,omitempty"`

  // If set, this represents the .metadata.generation that the condition was
  // set based upon.
  // For instance, if .metadata.generation is currently 12, but the
  // .status.condition[x].observedGeneration is 9, the condition is out of date
  // with respect to the current state of the Component.
  ObservedGeneration int64 `json:"observedGeneration,omitempty"`
}

// ComponentAppIDCondition describes the state of an app ID consuming
// the Component.
type ComponentAppIDCondition struct {
	//AppID is the ID of the app consuming the Component.
	AppID string `json:"appID"`

	// ReplicaConditions is a list of conditions indicating the state of
	// each replica consuming this Component.
	// +listType=map
	// +listMapKey=podName
	ReplicaConditions []ComponentAppIDReplicaCondition `json:"replicaConditions,omitempty"`
}

// ComponentAppIDReplicaCondition describes the state of an app ID replica
// consuming the Component.
type ComponentAppIDReplicaCondition struct {
  // PodName is the name of the pod consuming the Component.
  PodName string `json:"podName"`

  // Type of the condition, known values are (`Init`).
  Type ComponentAppIDReplicaConditionType `json:"type"`

  // Status of the condition, one of (`True`, `False`, `Unknown`).
  Status common.ConditionStatus `json:"status"`

  // LastTransitionTime is the timestamp corresponding to the last status
  // change of this condition.
  LastTransitionTime *metav1.Time `json:"lastTransitionTime,omitempty"`

  // Reason is a brief machine readable explanation for the condition's last
  // transition.
  // Typically empty if no error occurred.
  // +optional
  Reason *string `json:"reason,omitempty"`

  // Message is a human readable description of the details of the last
  // transition, complementing reason.
  // Typically empty if no error occurred.
  // +optional
  Message *string `json:"message,omitempty"`

  // If set, this represents the .metadata.generation that the condition was
  // set based upon.
  // For instance, if .metadata.generation is currently 12, but the
  // .status.condition[x].observedGeneration is 9, the condition is out of date
  // with respect to the current state of the Component.
  ObservedGeneration int64 `json:"observedGeneration,omitempty"`
}

// ComponentConditionType is a type of condition for a Component.
type ComponentConditionType string

const (
  // ComponentConditionTypeReady indicates a condition describing the
  // readiness of the Component.
  ComponentConditionTypeReady ComponentConditionType = "Ready"
)

// ComponentAppIDReplicaConditionType is a type of condition for an app replica
type ComponentAppIDReplicaConditionType string

const (
  // ComponentConditionAppIDReplicaInit indicates a condition describing the
  // initialization state of the Component.
  ComponentConditionAppIDReplicaInit ComponentAppIDReplicaConditionType = "Init"
)
```

```go
// +kubebuilder:object:generate=true

// ConditionStatus represents a condition's status.
// +kubebuilder:validation:Enum=True;False;Unknown
type ConditionStatus string

// These are valid condition statuses. "ConditionTrue" means a resource is in
// the condition; "ConditionFalse" means a resource is not in the condition;
// "ConditionUnknown" means kubernetes can't decide if a resource is in the
// condition or not. In the future, we could add other intermediate
// conditions, e.g. ConditionDegraded.
const (
	// ConditionTrue represents the fact that a given condition is true
	ConditionTrue ConditionStatus = "True"

	// ConditionFalse represents the fact that a given condition is false
	ConditionFalse ConditionStatus = "False"

	// ConditionUnknown represents the fact that a given condition is unknown
	ConditionUnknown ConditionStatus = "Unknown"
)
```

### Sentry

Today, Sentry signs peer certificates with the SPIFFE ID in the form of `spiffe://<trust-domain>/ns/<namespace>/<app-id>` for all validators.
In order for the Operator API server to also determine the Pod Name of the calling peer, Sentry needs to include the Pod name in the SPIFFE certificate for Daprds.
Currently, specifically for service/actor invocation, Daprds strongly string match on the existing path format, meaning that the pod name cannot be appended to the path like: `spiffe://<trust-domain>/ns/<namespace>/<app-id>/<pod-name>`.

To resolve this, all security client and server mTLS SPIFFE matchers will be updated to accept both pod name and non-pod name formats.
Sentry will sign peer certificates with the Pod name set as as the last DNS SAN.
After two releases with the client and server code accepting both formats, Sentry will be updated to only sign IDs with the new Pod name format and drop putting the Pod name in DNS SANS.

The Sentry proto RPC `SignCertificateRequest` message will be extended with a new field `pod_name`.
The Kubernetes validator will validate that strings matches the Kubernetes Pod name of the requesters bound service account token.
The JWKS Validator will cross reference this value with the `subject` of the signed JWT.

Daprd will derive the `pod_name` field of the request via the `POD_NAME` environment variable.
Control plane services will not set the pod name field, and will not have the pod name appear in the SPIFFE ID or DNS SAN.
Daprd will not set this field in self-hosted mode.

```proto
message SignCertificateRequest {
  ...
  // pod_name is an optional field to request the pod name to be included in the
  // SPIFFE peer certificate.
  optional string pod_name 7;
}
```

### Daprd

In Kubernetes mode, the Dapr runtime processor on boot will connect to the Operator and call the streaming API `ComponentReport`.
Dapr will not process any Components until this stream has been successfully made.

A new [headless service](https://kubernetes.io/docs/concepts/services-networking/service/#headless-services) will be created for the Operator API.
If Dapr calls one of the Operator IPs from this service and returns an error that it is not the leader, Dapr will attempt to connect to the next IP in the list in a round robin fashion.
This reporting stream may not be the same gRPC connection as used for other Operator API calls to help share the load of the Operator API servers.

After a Component has been reconciled by the runtime and initialization attempted, the result of that attempt will be reported to the Operator on the stream.
The runtime will track the initialization status of Components, and upon reconnecting to the Operator for any reason, will report the status of all loaded Components again.
Status reporting will also occur during hot reloading of a Component.

## Completion Checklist

- [ ] Add Kubernetes API and CRD definitions.
- [ ] Update sentry to sign peer certificates with the Pod name as the last DNS SAN entry.
- [ ] Add Opterator Component controller updating the status subresource.
- [ ] Add Operator gRPC API streaming method for Daprds to report Component status to Operator.
- [ ] Add `ComponentReport` client implementation to Daprd.
- [ ] After two releases, move pod name to the last DNS name in the SPIFFE ID.
