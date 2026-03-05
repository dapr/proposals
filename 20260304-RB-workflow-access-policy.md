# Dapr WorkflowAccessPolicy Resource

* Author(s): Josh van Leeuwen (@JoshVanL)
* State: Ready for Implementation
* Updated: 03/04/2026

## Overview

This proposal introduces a new standalone Kubernetes CRD and Dapr resource: **WorkflowAccessPolicy**. This resource controls which app IDs are permitted to start specific workflows and activities on a target application. It acts as an ingress access control mechanism for the Dapr Workflow building block, analogous to how the existing `AccessControlSpec` in the `Configuration` resource governs service invocation.

Areas affected:

- **Runtime**: Enforcement of workflow access policies in the Durable Task API service.
- **Building block**: Workflow building block gains access control semantics.
- **Components**: New CRD type registered with the Dapr control plane.

## Background

### Motivation

Dapr Workflows currently have **no dedicated access control mechanism**. Any application that can reach another application's Dapr sidecar can start, and by extension interact with, any workflow or activity on that application. In multi-tenant environments or platforms with shared infrastructure, this is a significant security gap.

The existing service invocation ACL (`Configuration.spec.accessControl`) only covers service-to-service direct invocation calls. Whilst possible to limit actor invocations using this mechanism, it does not use the workflow primitives as first class concepts.

### Goals

- Operators can define per-app policies that restrict which source app IDs may start specific workflows and activities.
- Policies support glob patterns for workflow and activity names (e.g., `order-*`).
- Caller identity is established via SPIFFE IDs extracted from mTLS certificates, consistent with existing Dapr security primitives.
- The policy uses `scopes` to associate with target apps, consistent with how Components, Subscriptions, and HTTPEndpoints work.
- The policy is a standalone CRD, decoupled from the monolithic `Configuration` resource.
- When a `WorkflowAccessPolicy` exists, the default action is **deny** — a blank policy with no scopes in a namespace denies all workflow invocations by default, providing a secure-by-default posture.
- When no `WorkflowAccessPolicy` exists for a target app, workflows are unrestricted (backward compatible).
- Policies cover both local workflow invocation and the upcoming cross-app workflow/activity invocation (being added this release) that routes through the Durable Task API service.

### Current Shortfalls

- **No workflow-level access control**: The only protection is API token authentication and namespace isolation, neither of which provides per-workflow or per-caller granularity.
- **Monolithic Configuration resource**: Adding more policy sections to the `Configuration` CRD increases its complexity and couples unrelated concerns. A standalone CRD follows the Dapr pattern of purpose-built resources (Resiliency, Subscription, Component).

## Related Items

### Related proposals

- Service invocation access control (existing `Configuration.spec.accessControl`): This proposal takes architectural inspiration from the service invocation ACL but defines a separate, workflow-specific resource.
- **Workflow history context propagation** (separate proposal): Enables application-level authorization by propagating caller context through the workflow history. `WorkflowAccessPolicy` provides infrastructure-level (sidecar) authorization, while history context propagation enables application-level authorization. These two mechanisms are complementary — the sidecar enforces coarse-grained access control, and the application can make fine-grained authorization decisions based on the propagated caller context.

## Expectations and Alternatives

### What is in scope

- Definition of the `WorkflowAccessPolicy` CRD and its schema.
- Enforcement logic in the Dapr runtime at the target sidecar's Durable Task API service.
- SPIFFE-based caller identity verification.
- Glob pattern matching for workflow and activity names.
- Secure-by-default: a `WorkflowAccessPolicy` defaults to `deny` when no rule matches.
- Hot-reload enabled by default for this resource.

### What is deliberately *not* in scope

- **Workflow management operations**: This proposal covers only the `Start` operation (starting workflows and activities). Controlling access to `Pause`, `Resume`, `Terminate`, `Purge`, `Get`, and `RaiseEvent` is deferred to a future iteration.
- **Cross-namespace workflow invocation**: Dapr does not currently support cross-namespace workflow invocation. Policy enforcement operates within a single namespace.
- **Non-mTLS identity**: This proposal requires mTLS for caller identification. Environments without mTLS will not be able to use workflow access policies.
- **Source-side enforcement**: Policies are enforced only at the target sidecar. The calling sidecar does not pre-check policies.

### Alternatives considered

#### 1. Extend the existing `Configuration.spec.accessControl`

The most obvious alternative is to add workflow-specific rules to the existing service invocation ACL section of the `Configuration` resource.

**Why not chosen:**
- The `Configuration` resource is already overloaded with tracing, metrics, middleware, features, logging, and access control for service invocation. Adding workflow policies further bloats it.
- Service invocation ACLs are deeply tied to the operation path + HTTP verb model. Workflow access control has a fundamentally different domain model (workflow names, activity names, operation types) that maps poorly to operation paths.
- A standalone CRD can be applied, updated, and deleted independently without restarting the sidecar's entire configuration.
- Following the Dapr trend: Resiliency, Subscriptions, and HTTP Endpoints were all broken out into standalone CRDs rather than being embedded in Configuration.

**Trade-offs:**
- Users must learn a new resource type.
- Policy configuration is spread across multiple resources rather than centralized.

#### 2. Reuse service invocation ACLs by routing workflows through service invocation

An alternative approach is to not create a workflow-specific policy at all, but instead route cross-app workflow calls through the existing service invocation path, which would automatically apply the existing ACLs.

**Why not chosen:**
- This would require workflows to be surfaced as service invocation operations, which conflates two distinct building blocks.
- Workflow names don't naturally map to HTTP paths/operations. A workflow named `ProcessOrder` would need to be artificially exposed as an operation like `/workflows/ProcessOrder` in the ACL.
- Activity invocations within workflows are internal to the engine and don't pass through service invocation handlers. They would be impossible to control this way.
- Users would need to understand the internal mapping between workflow names and service invocation paths, which is a leaky abstraction.

#### 3. Annotation-based or in-code policies

Another alternative is to let applications declare their own workflow access policies via code (e.g., middleware in the workflow handler) or Kubernetes annotations.

**Why not chosen:**
- Code-based policies are application-specific and cannot be managed centrally by platform operators.
- Annotations lack the expressiveness needed for complex rules (multiple callers, glob patterns, different actions per workflow).
- This shifts security responsibility to application developers rather than platform operators, which is the wrong layer for access control in a sidecar model.

#### 4. OPA/external policy engine integration

Delegate workflow access decisions to an external policy engine like Open Policy Agent (OPA) via a gRPC callout.

**Why not chosen for this proposal:**
- Adds a runtime dependency on an external system, increasing operational complexity.
- Introduces latency on every workflow invocation for the policy evaluation callout.
- OPA is powerful but overkill for most use cases that just need "app X can call workflow Y".
- This could be a future enhancement (a pluggable policy backend) but should not be the baseline mechanism.

**Trade-offs of this proposal's approach:**
- Policies are Dapr-specific and cannot leverage existing organization-wide policy infrastructure (e.g., OPA, Kyverno).
- No dynamic policy evaluation — policies are loaded at sidecar startup (and via hot-reload) and evaluated in-memory.

### Advantages

- **Consistent with Dapr patterns**: Follows the same SPIFFE-based identity model as service invocation ACLs and uses `scopes` for target app association, consistent with Components, Subscriptions, and HTTPEndpoints.
- **Standalone CRD**: Can be managed independently, supports GitOps workflows, and doesn't require `Configuration` changes.
- **Multi-app policies**: A single policy can protect multiple apps via `scopes`, reducing configuration duplication for apps with identical access requirements.
- **Glob patterns**: Provides flexibility without requiring exhaustive enumeration of every workflow and activity.
- **Minimal performance overhead**: Policies are parsed into in-memory data structures (trie or map) at load time; enforcement is a fast lookup, not a network call.
- **Secure by default**: When a policy exists, the default action is `deny`. A blank policy with no scopes locks down an entire namespace.
- **Non-breaking**: Existing deployments without any `WorkflowAccessPolicy` resources are unaffected — workflows remain unrestricted.
- **Complementary to application-level authorization**: Works alongside the upcoming workflow history context propagation to provide defense in depth — infrastructure-level policy at the sidecar, application-level authorization in workflow code.

### Disadvantages

- **mTLS required**: Without mTLS, caller identity cannot be verified, and the policy cannot be enforced. This is a hard requirement.
- **Start-only scope**: This initial version only controls who can start workflows/activities, not who can manage them. This may leave management operations unprotected.
- **Glob pattern complexity**: Glob matching adds implementation complexity and may have edge cases with overlapping patterns. The most specific match must win.

### Performance implications

- **Startup**: Parsing `WorkflowAccessPolicy` resources and building the in-memory lookup structure adds a small amount of time to sidecar initialization. This is negligible for typical policy sizes.
- **Request path**: Each workflow/activity invocation must perform a policy lookup. Using a trie or pre-compiled glob patterns, this is O(n) in the length of the workflow name, not O(n) in the number of rules. Expected overhead: **<1ms per request**.
- **Memory**: The in-memory policy structure is proportional to the number of rules. For typical deployments (tens to low hundreds of rules), memory impact is negligible.
- **No network overhead**: Unlike external policy engines, evaluation is entirely in-process.

## Implementation Details

### Design

#### CRD Schema

```yaml
apiVersion: dapr.io/v1alpha1
kind: WorkflowAccessPolicy
metadata:
  name: order-service-workflow-policy
# Apps whose workflows are protected by this policy.
# If empty, the policy applies to all apps in the namespace.
scopes:
  - order-service
spec:
  # Default action when no rule matches. Defaults to "deny" if omitted.
  defaultAction: deny

  # Ingress rules defining which callers can start which workflows/activities.
  rules:
    - # Callers that this rule applies to.
      callers:
        - appID: checkout-service
        - appID: inventory-service
      # Operations that the matched callers are allowed/denied to perform.
      operations:
        - type: workflow          # "workflow" or "activity"
          name: "ProcessOrder"    # exact name or glob pattern
          action: allow
        - type: workflow
          name: "Cancel*"
          action: allow
        - type: activity
          name: "ChargePayment"
          action: allow

    - callers:
        - appID: admin-service
      operations:
        - type: workflow
          name: "*"               # all workflows
          action: allow
        - type: activity
          name: "*"               # all activities
          action: allow
```

A single policy can protect multiple apps with identical access requirements:

```yaml
apiVersion: dapr.io/v1alpha1
kind: WorkflowAccessPolicy
metadata:
  name: shared-workflow-policy
scopes:
  - order-service
  - billing-service
  - shipping-service
spec:
  defaultAction: deny
  rules:
    - callers:
        - appID: admin-service
      operations:
        - type: workflow
          name: "*"
          action: allow
```

#### Go Types

```go
package workflowaccesspolicy

import (
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"

    "github.com/dapr/dapr/pkg/apis/common"
)

// +kubebuilder:object:root=true
type WorkflowAccessPolicy struct {
    metav1.TypeMeta   `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitempty"`
    Spec              WorkflowAccessPolicySpec `json:"spec,omitempty"`
    common.Scoped     `json:",inline"`
}

type WorkflowAccessPolicySpec struct {
    DefaultAction PolicyAction               `json:"defaultAction,omitempty"`
    Rules         []WorkflowAccessPolicyRule `json:"rules,omitempty"`
}

type PolicyAction string

const (
    PolicyActionAllow PolicyAction = "allow"
    PolicyActionDeny  PolicyAction = "deny"
)

type WorkflowAccessPolicyRule struct {
    Callers    []WorkflowCaller           `json:"callers"`
    Operations []WorkflowOperationRule    `json:"operations"`
}

type WorkflowCaller struct {
    AppID string `json:"appID"`
}

type WorkflowOperationType string

const (
    WorkflowOperationTypeWorkflow WorkflowOperationType = "workflow"
    WorkflowOperationTypeActivity WorkflowOperationType = "activity"
)

type WorkflowOperationRule struct {
    Type   WorkflowOperationType `json:"type"`
    Name   string                `json:"name"`
    Action PolicyAction          `json:"action"`
}

// +kubebuilder:object:root=true
type WorkflowAccessPolicyList struct {
    metav1.TypeMeta `json:",inline"`
    metav1.ListMeta `json:"metadata"`
    Items           []WorkflowAccessPolicy `json:"items"`
}
```

#### Cross-App Workflow Invocation

Cross-app workflow and activity execution is being added this release. It works through the **actor system** via the Durable Task API service:

1. An orchestrator on App A calls `CallActivity("myActivity", WithActivityAppID("app-b"))` or creates a sub-orchestration targeting another app.
2. The SDK injects a `Router` proto into the history event, carrying `SourceAppID` and `TargetAppID`.
3. The orchestrator actor constructs a cross-app actor type: `dapr.internal.<namespace>.<targetAppId>.activity` (or `.workflow`).
4. The placement table resolves this actor type to the target sidecar's address.
5. The source sidecar calls `CallActor` on the target sidecar via the `daprinternal` gRPC service.
6. The target sidecar invokes the activity/workflow actor locally.
7. Results are routed back using the `SourceAppID` from the `Router`.

#### Enforcement Flow

All workflow invocations — both local and cross-app — are handled through the Durable Task API service and the actor system. The deprecated HTTP/gRPC Dapr API endpoints for workflows are not relevant to this proposal.

```
  Local invocation                          Cross-app invocation
  ────────────────                          ────────────────────

  ┌──────────────────┐                      ┌──────────────────┐
  │  App schedules   │                      │  App A workflow  │
  │  workflow on     │                      │  calls           │
  │  own sidecar via │                      │  CallActivity()  │
  │  Durable Task    │                      │  with target     │
  │  API             │                      │  app ID          │
  └────────┬─────────┘                      └────────┬─────────┘
           │                                         │
    Actor invocation                          Orchestrator actor
    on local sidecar                          builds cross-app
           │                                  actor type
           │                                         │
           │                                         ▼
           │                                ┌──────────────────┐
           │                                │  daprinternal    │
           │                                │  CallActor on    │
           │                                │  Target Sidecar  │
           │                                └────────┬─────────┘
           │                                         │
           ▼                                         ▼
    ┌─────────────┐                           ┌─────────────┐
    │  Extract    │                           │  Extract     │
    │  caller     │                           │  caller      │
    │  app ID     │◄── SPIFFE ID              │  app ID      │◄── SPIFFE ID
    └──────┬──────┘                           └──────┬───────┘
           │                                         │
           │         ┌───────────────────┐           │
           └────────►│  Lookup policy    │◄──────────┘
                     │  scoped to this   │
                     │  app              │
                     └────────┬──────────┘
                              │
                       ┌──────▼───────┐
                       │ Policy found?│
                       └──────┬───────┘
                         No   │   Yes
                     ┌────────┤
                     ▼        ▼
                  ALLOW   ┌───────────────────┐
                          │ Match caller to   │
                          │ rule by appID     │
                          │ (from SPIFFE ID)  │
                          └────────┬──────────┘
                                   │
                            ┌──────▼───────┐
                            │ Caller match?│
                            └──────┬───────┘
                              No   │   Yes
                        ┌──────────┤
                        ▼          ▼
                     DENY       ┌───────────────────┐
                  (default)     │ Match workflow/   │
                                │ activity name     │
                                │ against glob      │
                                │ patterns          │
                                └────────┬──────────┘
                                         │
                                  ┌──────▼───────┐
                                  │ Match found? │
                                  └──────┬───────┘
                                    No   │   Yes
                              ┌──────────┤
                              ▼          ▼
                           DENY       action from
                        (default)     matched rule
```

#### Namespace-wide Deny-All

A blank `WorkflowAccessPolicy` with no scopes applied to a namespace provides a powerful security primitive — it denies all workflow invocations for every app in that namespace by default:

```yaml
apiVersion: dapr.io/v1alpha1
kind: WorkflowAccessPolicy
metadata:
  name: deny-all
  namespace: production
# No scopes — applies to all apps in the namespace.
spec: {}
# defaultAction defaults to "deny", no rules defined.
# Result: all workflow/activity invocations are denied.
```

This allows operators to lock down an entire namespace and then selectively grant access through additional, scoped policies.

#### Enforcement Integration Point

Policy enforcement is performed in the Durable Task API service when the target sidecar receives a workflow or activity invocation via `CallActor` on the `daprinternal` gRPC service. This is the single enforcement point for both local and cross-app invocations, since all workflow scheduling flows through the actor system.

1. Extracts the caller's app ID from the SPIFFE ID in the gRPC peer certificate.
2. Determines whether this is a workflow or activity invocation from the actor type (e.g., `dapr.internal.<ns>.<appid>.workflow` vs `.activity`).
3. Extracts the workflow/activity name from the request payload.
4. Looks up the `WorkflowAccessPolicy` scoped to the target app.
5. Evaluates the policy rules against the caller's app ID and returns `PermissionDenied` if denied.

```go
func (a *api) CallActor(ctx context.Context, in *internalv1pb.InternalInvokeRequest) (*internalv1pb.InternalInvokeResponse, error) {
    // Extract workflow/activity type and name from the actor type and request
    if wfType, wfName, ok := extractWorkflowInfo(in); ok {
        if err := a.enforceWorkflowAccessPolicy(ctx, wfType, wfName); err != nil {
            return nil, err
        }
    }

    // ... existing CallActor logic ...
}
```

#### Glob Pattern Matching

Glob patterns use `path.Match` semantics (the Go stdlib `path.Match` function):

- `*` matches any sequence of non-separator characters.
- `?` matches any single non-separator character.
- `[abc]` matches any character in the set.
- `ProcessOrder` matches exactly `ProcessOrder`.

When multiple rules match (e.g., `Process*` and `*`), the **most specific match wins**. Specificity is determined by the length of the literal prefix before the first wildcard character.

#### Policy Loading

- **Kubernetes**: The Dapr Operator watches `WorkflowAccessPolicy` resources and streams them to sidecars, consistent with how other Dapr resources (Components, Subscriptions, Resiliency) are loaded.
- **Self-hosted**: Policies are loaded from YAML files in the resources directory, following the same pattern as other Dapr resources.
- **Hot-reload**: Enabled by default for this resource. Policy changes are picked up dynamically via the Operator's watch stream without requiring sidecar restart. In self-hosted mode, file system watches detect changes and reload policies automatically.

### Feature Lifecycle Outline

#### Alpha (v1alpha1)

- CRD registered and loadable by the Dapr runtime.
- Enforcement on workflow and activity start invocations via the Durable Task API service.
- Glob pattern support for workflow and activity names.
- SPIFFE-based caller identity.
- Hot-reload enabled by default.
- Feature gated behind a feature flag: `WorkflowAccessPolicy`.
- No guarantees on schema stability.

#### Beta (v1beta1)

- Schema stabilized based on community feedback.
- Consider extending to workflow management operations (pause, resume, terminate, etc.).
- Metrics for policy evaluation (allow/deny counts per caller/workflow).
- Performance benchmarks published.

#### Stable (v1)

- Schema frozen with backward compatibility guarantees.
- Full coverage of workflow operations.
- Comprehensive documentation and SDK examples.

### Acceptance Criteria

- **Correctness**: A `WorkflowAccessPolicy` scoped to an app with `defaultAction: deny` blocks all unlisted callers from starting workflows on that app. Listed callers with `action: allow` can start the matching workflows.
- **Scopes**: The policy is only loaded by sidecars whose app ID appears in the `scopes` list. An empty `scopes` list applies the policy to all apps in the namespace.
- **Identity verification**: Caller app ID is extracted from the SPIFFE ID in the mTLS certificate. Requests without valid mTLS credentials are denied when a policy exists with `defaultAction: deny`.
- **Glob matching**: Patterns like `Process*` match `ProcessOrder`, `ProcessRefund`, etc. The most specific pattern takes precedence.
- **Default deny**: When a `WorkflowAccessPolicy` exists and no rule matches, the invocation is denied.
- **Namespace-wide deny**: A blank `WorkflowAccessPolicy` with no scopes denies all workflow invocations for every app in the namespace.
- **No policy = unrestricted**: When no `WorkflowAccessPolicy` exists for an app, all workflow invocations are allowed (backward compatible).
- **Performance**: Policy evaluation adds <1ms latency to workflow invocations. Benchmark tests verify this.
- **Hot-reload**: Policy changes are applied dynamically without sidecar restart.

## Completion Checklist

- [ ] CRD type definitions (`pkg/apis/workflowaccesspolicy/`)
- [ ] CRD registration with the Dapr Operator
- [ ] Policy parsing and in-memory data structure (trie/map with glob support)
- [ ] Enforcement logic in `CallActor` handler (Durable Task API service / `daprinternal`)
- [ ] SPIFFE ID extraction utility (reuse from `pkg/acl/`)
- [ ] Operator watch/stream support for `WorkflowAccessPolicy` resources
- [ ] Self-hosted file loading support
- [ ] Hot-reload support
- [ ] Unit tests for policy parsing, glob matching, and enforcement
- [ ] Integration tests (allowed/denied workflow start with various policy configurations)
- [ ] E2E tests in Kubernetes
- [ ] Feature flag gating
- [ ] Metrics for policy evaluation results
- [ ] Documentation (Dapr docs site)
