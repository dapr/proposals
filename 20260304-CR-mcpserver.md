# MCPServer: A First-Class Dapr Resource for Durable MCP Server Integration

* Author(s): Samantha Coyle (@sicoyle)
* State: Ready for Implementation
* Updated: 04/02/2026

## Overview

MCP is rapidly becoming the standard protocol for exposing tools, resources, and prompts to LLM-based agents.
This proposal makes Dapr the first platform to provide **enterprise-grade, durable MCP execution** as a first-class primitive:

- **For developers**: Declare an `MCPServer` resource, get durable tool calls immediately — no connection boilerplate, no credential management in application code, no separate proxy service to deploy or maintain.
- **For operators and platform teams**: The same Dapr governance model that applies to StateStores, PubSubs, HTTPEndpoints, etc now applies to MCPServers — namespace-scoped access control, secret store integration, and hot-reload without sidecar restarts.
- **Dapr's unique value**: Dapr Workflow's `wait_for_external_event` primitive is the only viable way to implement `elicitation` and `sampling` — the [MCP core client features](https://modelcontextprotocol.io/docs/learn/client-concepts#core-client-features) that require suspending execution mid tool call for human or LLM input. I'm not aware of any other platform that can model this durably today.

**Areas affected:**
- `dapr/dapr` — new `MCPServer` CRD type, Operator API extensions, runtime loading, a built-in MCP workflow worker embedded in daprd, and new Operator gRPC proto definitions for MCPServer list/watch/get
- `dapr/cli` - support for MCP stdio transport to work, and maybe a `dapr mcp` CLI cmd
- `dapr-agents` — new `DaprMCPWorkflowClient` that replaces direct MCPClient connections for durable agent workflows for MCP interactions

**What is being proposed:**

1. A new first-class Dapr resource `MCPServer` that lets operators declare MCP server connection details as Kubernetes CRDs or standalone YAML files. daprd discovers and loads these at startup and watches for updates via the Operator.

2. Built-in `dapr.mcp.<mcpserver-name>.ListTools` and `dapr.mcp.<mcpserver-name>.CallTool` workflow orchestrations registered inside daprd's workflow engine. These execute MCP protocol calls as durable activities, provide replay-safe checkpointing for long-running tools, and are architected to support elicitation and sampling suspensions in a follow-on phase.

3. A `DaprMCPWorkflowClient` in dapr-agents that wraps the built-in workflows as `WorkflowContextInjectedTool` instances, giving `DurableAgent` workflows access to MCP servers via `ctx.call_child_workflow("dapr.mcp.<mcpserver-name>.CallTool", ...)` — the same pattern used for agent to agent calls, or agents as tools usage today.

---

## Background

The Model Context Protocol (MCP) has rapidly become the standard interface for exposing tools, resources, and prompts to LLM-based agents. Dapr Agents today supports MCP via direct client connections (`MCPClient`), but this approach has several gaps for enterprise and production use:

**Durability gap.** Direct MCP connections in Python are not durable. If the agent process or sidecar crashes mid tool call, the call is lost. Dapr's workflow engine provides durable execution with checkpoint-and-resume semantics, but MCP calls made outside the workflow context do not benefit from this.

**Human-in-the-loop gap.** The MCP specification includes `elicitation/create` (server requests user input mid-call) and `sampling/createMessage` (server requests LLM completion mid-call). Both require the ability to suspend execution and wait — potentially for minutes or hours — before resuming. A simple call with retry/resiliency policy cannot model this. Workflow orchestrations with `wait_for_external_event` can.

**Governance gap.** There is currently no centralized place for operators to declare which MCP servers an application is allowed to use, with what credentials, at what timeout, and with what transport. Each application hardcodes its own connection logic. A `MCPServer` CRD creates a single source of truth, enables namespace-scoped access control, and gives users a native resource to surface its connection details.

**Operational gap.** Existing MCP connections require manual credential management in application code. The `MCPServer` resource enables secret store references, so API keys and headers are never hardcoded.

**Why workflow rather than activity + resiliency?**

For today's common case (fast, idempotent tool calls), an activity with a resiliency policy is technically sufficient. Workflow is the right choice for three reasons:

1. **API stability**: If `dapr.mcp.<mcpserver-name>.CallTool` is implemented as `ctx.call_activity(...)` today and later needs to support elicitation (which requires `wait_for_external_event`), the call site in every agent must change to `ctx.call_child_workflow(...)`. Implementing as a workflow from the start means the call site — and every agent's code — never changes as the implementation gains capabilities.

2. **Long-running tools**: MCP tools that execute database queries, code runners, or external jobs may run for minutes. If daprd restarts in the middle of this, the MCP tool execution retries from scratch. A workflow activity within a child workflow checkpoints its progress; if daprd restarts, execution resumes at the last completed step.

3. **Elicitation and sampling (follow-on phase)**: These MCP features require pausing mid-call and resuming after human or LLM input. This maps directly to Dapr Workflow's `raise_event` / `wait_for_external_event` primitives.

---

## Related Items

### Related proposals

None.
You can see MCP interactions within Dapr Agents repo in our [quickstarts](https://github.com/dapr/dapr-agents/blob/main/quickstarts/04_agent_mcp_tools.py),
or in this [release note agent](https://github.com/sicoyle/release-note-agent).

### Related issues

[MCP Server for Dapr APIs](https://github.com/dapr/dapr/issues/8783)

---

## Expectations and alternatives

### In scope

- `MCPServer` CRD type with spec fields defined in yaml examples below
- Operator gRPC extensions: `ListMCPServers`, `GetMCPServer`, `WatchMCPServer` (streaming `MCPServerUpdate` events)
- Kubernetes and standalone (disk) loaders for `MCPServer`
- Secret processing for headers and env vars via existing `auth.secretStore`
- Runtime loading in `loadMCPServers()` during daprd startup, with hot-reload via Operator watch
- Built-in `dapr.mcp.<mcpserver-name>.ListTools` and `dapr.mcp.<mcpserver-name>.CallTool` workflows defined and registered with daprd's wfengine using the `modelcontextprotocol/go-sdk`
    - This is a fundamental change that brings the idea of a Managed Workflow in essence into the runtime.
- Built-in MCP middleware pipelines — `middleware.beforeCallTool` / `middleware.afterCallTool` / `middleware.beforeListTools` / `middleware.afterListTools` hook arrays on `MCPServerSpec` provide ordered extensibility points for custom workflows (RBAC, audit, sanitization, rate limiting). Each hook is an `MCPMiddlewareHook` containing a `workflow` (with optional `appID` for remote invocation). "before" hooks abort the chain on error; "after" hooks log errors without affecting the result
- Transport support: `streamable_http`, `sse`, `stdio`

> NOTE: I did choose the [official modelcontextprotocol go-sdk pkg](https://github.com/modelcontextprotocol/go-sdk); however, there is the [Mark3Labs alternative](https://github.com/mark3labs/mcp-go) which is more "popular" and established.

### Explicitly not in scope (initial phase)

- Elicitation (`elicitation/create`) callback handling — architecturally enabled by the workflow design but not implemented in v1
- Sampling (`sampling/createMessage`) callback handling — same
- MCP resource subscriptions and prompt listing (tools only in v1) - https://modelcontextprotocol.io/docs/learn/server-concepts#core-server-features
- Cross-namespace MCPServer access
- `endpoint.appID` — routing to a Dapr-managed MCP server by app ID rather than raw URL; deferred due to URL construction complexity across Kubernetes and standalone modes
- `endpoint.httpEndpointName` — referencing an existing `HTTPEndpoint` resource as the MCP server's connection details; deferred due to cross-resource dependency handling and header merge semantics

### Alternatives considered

**Alternative 1: Reuse the existing Component CRD with a new `mcpserver` category.**

Reusing the Component infrastructure would reduce the code to write significantly. 
However: 
- MCPServer semantic is fundamentally different from I/O components. It is a tool provider, not a StateStore or Binding
- This would then require a `components-contrib` implementation, which adds unnecessary indirection for a protocol-level integration.

**Alternative 2: External proxy workflow appid.**

An external appid to potentially host the dapr.mcp.<mcpserver-name>.ListTools/dapr.mcp.<mcpserver-name>.CallTool workflows avoids embedding MCP in daprd and provides independent scaling. 
Disadvantages:
- Requires an additional Kubernetes deployment if using Kubernetes mode
- Access policy configuration between two appids

Implementation within daprd runtime is simpler operationally and durability is transparent within the sidecar.

**Alternative 3: Not wrapping dapr.mcp.<mcpserver-name>.ListTools and dapr.mcp.<mcpserver-name>.CallTool within a workflow**

Simpler to implement and sufficient for the common case.
However, ruled out because:
- it creates an API cliff when elicitation/sampling support is needed
- it cannot model mid-call suspension
- it provides no checkpointing for long-running tools if the app were to go down mid MCP tool execution

### Trade-offs

- **Embedding MCP in daprd** 
This couples the MCP protocol client to the daprd release cycle. 
MCP is evolving rapidly, so there is risk we have to cut dapr release patches more frequently to get MCP pkg fixes in.

Mitigated by using `modelcontextprotocol/go-sdk` (the official SDK) and the orchestration API surface (workflow names, activities, etc) is decoupled from SDK internals.
They are also already [v1.0](https://pkg.go.dev/github.com/modelcontextprotocol/go-sdk/mcp?tab=versions) so we should theoretically be fine here.

- **stdio transport in sidecar**
Stdio requires spawning a subprocess.
This is flagged as a "local dev / testing" transport and for non-Kubernetes mode.
Production use is expected to use `streamable_http` or `sse`.
An error will be used when stdio is used in Kubernetes mode.
This will require CLI work to support.

---

## Implementation Details

### Design

#### MCPServer CRD

The full spec for MCP server:

```yaml
apiVersion: dapr.io/v1alpha1
kind: MCPServer
metadata:
  name: payments-mcp
  namespace: prod
  labels:
    env: prod
    team: fin-platform
spec:
  # catalog: UI-facing governance metadata that is purely informational for end users.
  catalog:
    displayName: "Payments MCP Server"
    description: >
      Provides MCP tools for payment lookup, refund initiation,
      and transaction status queries.
    owner:
      team: fin-platform
      contact: fin-platform-oncall@company.com
    tags:
      - payments
      - pii
      - production
    links:
      docs: https://internal.company.com/docs/payments-mcp
      runbook: https://internal.company.com/runbooks/payments-mcp
      dashboard: https://grafana.company.com/d/payments-mcp
      repo: https://github.com/company/payments-mcp

  endpoint:
    sse:
      url: http://payments-mcp.prod.svc.cluster.local/sse
      # protocolVersion pins the MCP spec version the server implements.
      protocolVersion: "2026-03-26"

      # Auth — pick one option; comment out the others.

      # Option 1: OAuth2 client credentials (production; secret resolved via auth.secretStore)
      auth:
        secretStore: kubernetes
        oauth2:
          issuer: https://auth.company.com
          audience: mcp://payments
          scopes:
            - payments.read
            - payments.refund
          secretKeyRef:
            name: payments-mcp-oauth
            key: clientSecret

      # Option 2: SPIFFE workload identity — Sentry issues an SVID; no secret needed.
      # auth:
      #   spiffe:
      #     jwt:
      #       header: Authorization
      #       headerValuePrefix: "Bearer "
      #       audience: mcp://payments

      # Option 3: env var injection — local dev or stdio; no secret store required.
      # headers:
      #   - name: Authorization
      #     envRef: PAYMENTS_MCP_TOKEN

scopes:
  - my-agent-app
```

External (non-Dapr) MCP server with a bearer token:

```yaml
apiVersion: dapr.io/v1alpha1
kind: MCPServer
metadata:
  name: github-mcp
  namespace: default
spec:
  endpoint:
    streamableHTTP:
      url: https://api.githubcopilot.com/mcp/
      protocolVersion: "2025-03-XX"
      # Headers are resolved via auth.secretStore (defaults to kubernetes).
      # Bearer tokens are expressed as a plain Authorization header.
      headers:
        - name: Authorization
          secretKeyRef:
            name: github-token
            key: token
      auth:
        secretStore: kubernetes
scopes:
  - my-agent-app
```

Local stdio MCP server (standalone / dev mode only):

```yaml
apiVersion: dapr.io/v1alpha1
kind: MCPServer
metadata:
  name: local-tools
spec:
  endpoint:
    stdio:
      command: python
      args: ["-m", "my_tools.py"]
      env:
        - name: EXAMPLE
          value: EXAMPLE
```

MCPServer with `beforeCallTool` authz and `afterCallTool` audit hook pipelines:

```yaml
apiVersion: dapr.io/v1alpha1
kind: MCPServer
metadata:
  name: payments-mcp-guarded
  namespace: prod
spec:
  endpoint:
    streamableHTTP:
      url: https://payments.internal/mcp
  middleware:
    # Hooks are executed in array order. Errors in "before" hooks abort the chain.
    beforeCallTool:
      # Example: rate limit, then RBAC check.
      - workflow:
          workflowName: rate-limiter
      - workflow:
          workflowName: payments-authz-workflow
    afterCallTool:
      - workflow:
          workflowName: payments-audit-workflow
          appID: remote-audit-app  # optional: run on a remote Dapr app
scopes:
  - my-agent-app
```

The `beforeCallTool` workflow receives:
```json
{ "mcpServer": "payments-mcp-guarded", "toolName": "refund_payment", "arguments": { "amount": 50 } }
```
Return any non-nil error to abort; return nil to allow the call to proceed.

The `afterCallTool` workflow receives:
```json
{ "mcpServer": "payments-mcp-guarded", "toolName": "refund_payment", "arguments": { "amount": 50 }, "result": { ... } }
```
The return value is ignored. Internal errors (e.g. audit store unavailable) should be handled within the workflow itself.

#### Middleware scenarios

The middleware pipeline is designed to support a range of use cases. Below are
concrete scenarios that informed the API design and can guide future extensions.

**Per-tool RBAC (beforeCallTool)**
An enterprise wants to check if the calling agent has permission to invoke
`payments.refund`. A `beforeCallTool` workflow looks up the agent identity and
tool name against a policy store. If denied, the workflow returns an error and
the tool call is aborted with `CallToolResult{isError: true}`.

**Audit logging (afterCallTool)**
Write every tool invocation and result to an immutable audit store. An
`afterCallTool` workflow receives the tool name, arguments, and result, and
writes an audit record. Errors in the audit workflow are logged but do not
affect the result returned to the caller.

**Input validation (beforeCallTool)**
Reject tool arguments that contain disallowed PII or fail sanitization rules
before they reach the MCP server. A `beforeCallTool` workflow inspects the
arguments and returns an error to abort if validation fails.

**Result enrichment (afterCallTool)**
Annotate tool results with metadata such as latency, cost, or model version.
An `afterCallTool` workflow receives the result and writes enriched metadata to
a state store or publishes to a topic.

**Rate limiting (beforeCallTool)**
Throttle how often an agent can call a specific tool. A `beforeCallTool`
workflow checks a counter in a state store and returns an error if the limit is
exceeded.

**Tool discovery filtering (beforeListTools / afterListTools)**
Filter which tools an agent can discover based on identity or context. A
`beforeListTools` hook could check permissions before the list is fetched, or
an `afterListTools` hook could filter the result set.

**Chained middleware**
Multiple hooks can be composed in a pipeline. For example, rate limiting
followed by RBAC:
```yaml
middleware:
  beforeCallTool:
    - workflow:
        workflowName: rate-limiter
    - workflow:
        workflowName: rbac-check
        appID: auth-service
```
Hooks execute in array order. If `rate-limiter` returns an error, `rbac-check`
is never invoked.

**Future hook types**
The `MCPMiddlewareHook` struct is a one-of and currently only supports
`workflow`. Future extensions could include:
- **HTTP callback** (`httpCallback`): POST to a URL instead of invoking a
  workflow, for stateless checks like RBAC lookups.
- **CEL/Rego policy** (`policy`): Inline policy evaluation without a workflow
  round-trip (e.g. `cel: "request.toolName != 'dangerous_tool'"`).
- **Built-in rules**: Declarative fields like `allowedTools`, `deniedTools`,
  `maxArgumentSize` that require no user code.

#### Go types (`pkg/apis/mcp/v1alpha1/types.go`)

```go
type MCPServer struct {
    metav1.TypeMeta   `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitempty"`
    Spec              MCPServerSpec   `json:"spec,omitempty"`
    Status            MCPServerStatus `json:"status,omitempty"`
    common.Scoped     `json:",inline"`
}

type MCPServerSpec struct {
    Catalog    *MCPServerCatalog `json:"catalog,omitempty"`
    Endpoint   MCPEndpoint       `json:"endpoint"`
    // Middleware defines optional workflow hooks invoked around each tool call.
    Middleware *MCPMiddleware    `json:"middleware,omitempty"`
}

// MCPEndpoint describes how to reach the MCP server.
// Exactly one of StreamableHTTP, SSE, or Stdio must be set.
type MCPEndpoint struct {
    StreamableHTTP *MCPStreamableHTTP `json:"streamableHTTP,omitempty"`
    SSE            *MCPSSE            `json:"sse,omitempty"`
    Stdio          *MCPStdio          `json:"stdio,omitempty"`
}

// MCPStreamableHTTP configures the streamable_http transport.
type MCPStreamableHTTP struct {
    URL             string                 `json:"url"`
    ProtocolVersion *string                `json:"protocolVersion,omitempty"`
    Timeout         *metav1.Duration       `json:"timeout,omitempty"`
    Headers         []common.NameValuePair `json:"headers,omitempty"`
    Auth            *MCPAuth               `json:"auth,omitempty"`
}

// MCPSSE configures the legacy SSE transport.
type MCPSSE struct {
    URL             string                 `json:"url"`
    ProtocolVersion *string                `json:"protocolVersion,omitempty"`
    Timeout         *metav1.Duration       `json:"timeout,omitempty"`
    Headers         []common.NameValuePair `json:"headers,omitempty"`
    Auth            *MCPAuth               `json:"auth,omitempty"`
}

// MCPStdio configures the stdio subprocess transport.
type MCPStdio struct {
    Command string                 `json:"command"`
    Args    []string               `json:"args,omitempty"`
    Env     []common.NameValuePair `json:"env,omitempty"`
}

// MCPAuth configures authentication for the MCP server connection.
// Nested under the HTTP transport (StreamableHTTP or SSE).
type MCPAuth struct {
    // SecretStore names the Dapr secret store used to resolve secretKeyRef entries
    // in transport headers. Defaults to "kubernetes".
    SecretStore *string     `json:"secretStore,omitempty"`
    OAuth2      *MCPOAuth2  `json:"oauth2,omitempty"`
    SPIFFE      *SPIFFE     `json:"spiffe,omitempty"`
}

type MCPOAuth2 struct {
    Issuer       string               `json:"issuer"`
    Audience     *string              `json:"audience,omitempty"`
    Scopes       []string             `json:"scopes,omitempty"`
    SecretKeyRef *common.SecretKeyRef `json:"secretKeyRef,omitempty"`
}

type SPIFFE struct {
    //+optional
    JWT *SPIFFEJWT `json:"jwt,omitempty"`
}

type SPIFFEJWT struct {
    Header            string  `json:"header" validate:"required"`
    Audience          string  `json:"audience" validate:"required"`
    //+optional
    HeaderValuePrefix *string `json:"headerValuePrefix,omitempty"`
}

// MCPMiddleware defines optional hook pipelines invoked around tool and list operations.
// Hooks are executed in array order.
type MCPMiddleware struct {
    BeforeCallTool  []MCPMiddlewareHook `json:"beforeCallTool,omitempty"`
    AfterCallTool   []MCPMiddlewareHook `json:"afterCallTool,omitempty"`
    BeforeListTools []MCPMiddlewareHook `json:"beforeListTools,omitempty"`
    AfterListTools  []MCPMiddlewareHook `json:"afterListTools,omitempty"`
}

// MCPMiddlewareHook is a single middleware hook. Exactly one field must be set.
type MCPMiddlewareHook struct {
    Workflow *MCPMiddlewareWorkflow `json:"workflow,omitempty"`
    // Future: HTTPCallback, Policy, etc.
}

// MCPMiddlewareWorkflow identifies a workflow to invoke as a middleware hook.
type MCPMiddlewareWorkflow struct {
    WorkflowName string  `json:"workflowName"`
    // AppID targets the workflow on a remote Dapr app. When unset, runs locally.
    AppID        *string `json:"appID,omitempty"`
}

type MCPServerCatalog struct {
    DisplayName string            `json:"displayName,omitempty"`
    Description string            `json:"description,omitempty"`
    Owner       MCPCatalogOwner   `json:"owner,omitempty"`
    Tags        []string          `json:"tags,omitempty"`
    Links       map[string]string `json:"links,omitempty"`
}

type MCPCatalogOwner struct {
    Team    string `json:"team,omitempty"`
    Contact string `json:"contact,omitempty"`
}
```
#### Workflow runtime activation and routing

Today, daprd only starts the workflow runtime (wfengine) when an application connects as a workflow client. With this proposal, the workflow runtime must also start when any `MCPServer` manifest is present — even when no user application has connected — because the built-in MCP worker registers its own orchestrations and activities inside daprd.

This requires two related changes within the sidecar:

1. **Activation condition**: `loadMCPServers()` must trigger wfengine startup when MCPServer CRDs are found, in addition to the existing trigger from a workflow client connecting.

2. **Workflow router**: Within daprd, there are two workflow clients: the internal one (used by daprd to schedule built-in MCP orchestrations) and the external one (the proxy for user app workflow requests arriving via the Dapr API). A router on the sidecar-side client must dispatch workflow names prefixed `dapr.mcp.*` to the built-in MCP worker, while all other workflow names continue to route to the user app's registered worker. Without this router, a `call_child_workflow("dapr.mcp.<name>.ListTools", ...)` issued from a user workflow would be treated as an unknown type.

#### Child workflow instance IDs

Child workflow instance IDs for the built-in `dapr.mcp.<mcpserver-name>.ListTools` and `dapr.mcp.<mcpserver-name>.CallTool` orchestrations are **auto-generated by the runtime**. When a caller does not supply an explicit `instanceID`, Durabletask generates a deterministic ID of the form `<parent-instance-id>:<task-sequence-number>`. This enables:

- **No collision**: sequence numbers are monotonically increasing within a parent instance.
- **Replay determinism**: on re-execution the same parent ID + same sequence number produces the same child ID, so workflow history remains consistent. A timestamp-based ID would differ on replay and corrupt the history.
- **Natural correlation**: the child ID encodes the parent, so tool call spans are trivially traceable back to the agent workflow that issued them.

Callers (SDK or user workflows) must not pass a caller-supplied timestamp or random value as the instance ID for these child workflows.

#### Built-in workflow orchestration flow

```
dapr.agents.<agent_name>.workflow (DurableAgent)
  └─ ctx.call_child_workflow("dapr.mcp.<mcpserver-name>.ListTools", {mcp: "github-mcp"})
       └─ [daprd built-in dapr.mcp.<mcpserver-name>.ListTools orchestration]
            └─ ctx.CallActivity("list-tools", input)
                 └─ [daprd built-in activity]
                      ├─ looks up MCPServer CRD from CompStore
                      ├─ dials MCP server via go-sdk
                      └─ returns []mcp.Tool as JSON

dapr.agents.<agent_name>.workflow
  └─ ctx.call_child_workflow("dapr.mcp.<mcpserver-name>.CallTool", {mcp: "github-mcp", tool: "search_code", arguments: {...}})
       └─ [daprd built-in dapr.mcp.<mcpserver-name>.CallTool orchestration]
            ├─ for each hook in middleware.beforeCallTool:
            │    ctx.call_child_workflow(hook.workflow.workflowName, {mcpServer, toolName, arguments})
            │    → error returned? chain stops, return CallToolResult{isError: true, content: [error]}
            ├─ ctx.CallActivity("call-tool", input)
            │    └─ [daprd built-in activity]
            │         ├─ looks up MCPServer CRD from CompStore
            │         ├─ dials MCP server, calls tool
            │         │    → auth failure (401/403, OAuth2 exchange, SPIFFE rejected)?
            │         │         return CallToolResult{isError: true, content: [error]} to caller
            │         └─ returns tool result
            └─ for each hook in middleware.afterCallTool:
                 ctx.call_child_workflow(hook.workflow.workflowName, {mcpServer, toolName, arguments, result})
                 → errors logged, result still returned to caller
```

#### Error propagation contract

All auth/identity errors — whether from `beforeCallTool` middleware hooks or from the transport layer (OAuth2 token exchange failure, 401/403 from the MCP server, SPIFFE JWT rejection) — **are returned as `CallToolResult{isError: true}`**, not as a workflow execution failure.

This is required so the calling agent's execution loop receives a structured tool error it can forward to the LLM. The LLM can then reason about the denial (e.g. missing scope, wrong identity) and decide to retry with different parameters, request elevation, or surface the error to the user. A workflow termination status of `FAILED` would instead halt the agent loop.

The `content` field of the error result should carry a human-readable description, e.g.:

```json
{
  "isError": true,
  "content": [
    { "type": "text", "text": "identity denied: agent 'my-agent' is not permitted to call tool 'payments.refund' (beforeCallTool: payments-authz-workflow)" }
  ]
}
```

Transport-level errors follow the same shape:

```json
{
  "isError": true,
  "content": [
    { "type": "text", "text": "auth failure calling payments-mcp: 401 Unauthorized (OAuth2 token exchange failed)" }
  ]
}
```

#### dapr-agents usage

```python
# Import path subject to change
from dapr_agents.tool.mcp.dapr_workflow_client import DaprMCPWorkflowClient

# At agent startup (before workflow execution):
client = DaprMCPWorkflowClient()
await client.connect("github-mcp")   # invokes dapr.mcp.<mcpserver-name>.ListTools workflow, caches schemas
tools = client.get_all_tools()       # returns WorkflowContextInjectedTool instances

agent = DurableAgent(
    name="my_agent",
    tools=tools,   # each tool calls dapr.mcp.<mcpserver-name>.CallTool workflow when invoked by LLM under the hood
    llm=DaprChatClient(component_name="openai"),
)
```

The `DurableAgent`'s existing dispatch loop handles `WorkflowContextInjectedTool` with no changes:

```python
# durable.py — no changes needed
if isinstance(tool_obj, WorkflowContextInjectedTool):
    workflow_tasks.append(tool_obj(ctx=ctx, _source_agent=self.name, **args))
    # → internally calls ctx.call_child_workflow("dapr.mcp.<mcpserver-name>.CallTool", {...})
```

#### Architecture diagram

```
┌─────────────────────────────────────────────────────────---------------------┐
│  User app (dapr-agents DurableAgent)                                         │
│                                                                              │
│  dapr.agents.<agent_name>.workflow                                                 │
│    └─ call_child_workflow("dapr.mcp.<mcpserver-name>.ListTools", {mcp: ...}) │
│    └─ call_child_workflow("dapr.mcp.<mcpserver-name>.CallTool",  {mcp: ...}) │
└────────────────────┬────────────────────────────────────---------------------┘
                     │ Dapr Workflow API
┌────────────────────▼────────────────────────────────────---------┐
│  daprd sidecar                                                   │
│                                                                  │
│  ┌─ wfengine (actor backend) ─────────────────────┐              │
│  │  Built-in MCP worker                           │              │
│  │    dapr.mcp.<mcpserver-name>.ListTools workflow orchestration │
│  │      └─ dapr-mcp-list-tools activity           │              │
│  │    dapr.mcp.<mcpserver-name>.CallTool workflow orchestration  |
│  │      └─ dapr-mcp-call-tool activity            │              │
│  └────────────────────────────────────────────────┘              │
│                                                                  │
│  CompStore: MCPServer CRDs (loaded from Operator/disk)           │
└────────────┬────────────────────────────────────────────---------┘
             │ MCP protocol (streamable_http / SSE / stdio)
    ┌────────▼──────┐  ┌──────────────┐  ┌──────────────┐
    │ GitHub MCP    │  │ Dapr MCP     │  │ Custom MCP   │
    │ Server        │  │ Server       │  │ Server       │
    └───────────────┘  └──────────────┘  └──────────────┘
```

### Feature lifecycle outline

**Alpha (initial release):**
- `MCPServer` resource gated behind a feature flag `MCPServerResource` in Dapr Configuration
- Built-in `dapr.mcp.<mcpserver-name>.ListTools` and `dapr.mcp.<mcpserver-name>.CallTool` workflows available when flag is enabled
- All three transports supported but stdio in Kubernetes mode errors
- Dapr CLI to error if using slim mode as workflows are needed for this resource
- No elicitation or sampling support yet
- MCPServer appears in daprd metadata API response

**Future:**
- Feature flag removed; MCPServer enabled by default
- Elicitation callback support added (workflow `wait_for_external_event` pattern)
- Sampling support added
- `endpoint.appID` — route to a Dapr-managed MCP server by app ID; sidecar constructs the service invocation URL automatically
- `endpoint.httpEndpointName` — reference an existing `HTTPEndpoint` resource by name; MCP worker reads its base URL and resolved headers from the CompStore, eliminating duplication of auth/header config
- Streaming MCPServer discovery for agents — rather than requiring `client.connect("github-mcp")` at startup, agents could watch the Operator's `WatchMCPServer` stream and auto-discover or reload tool schemas as new `MCPServer` CRDs are applied, without restarting the agent workflow

**Compatibility:**
- The `dapr.mcp.<mcpserver-name>.ListTools` and `dapr.mcp.<mcpserver-name>.CallTool` workflow names are stable API. Agents coded against them will not need changes as internal implementation evolves (e.g., elicitation support is added inside the workflow without changing the input/output contract).
- `MCPServer` CRD spec fields are additive; existing fields will not be removed in minor versions.
- The existing `MCPClient` in dapr-agents will be removed. `DaprMCPWorkflowClient` is recommended approach for durable workflows and MCP interactions.

### Acceptance Criteria

**Functional:**
- A `DurableAgent` can call tools on a remote `streamable_http` MCP server declared as a `MCPServer` CRD, with the MCP tool call durably recorded in workflow history
- A standard (non-durable) MCP client can also call tools on a `MCPServer`-declared server; internally the call is routed through the built-in workflow orchestration, providing durability and audit without requiring the client to be workflow-aware
- If daprd restarts mid tool call, the activity retries automatically since it's within a workflow
- `MCPServer` CRDs are hot-reloaded when updated in Kubernetes (no sidecar restart required)
- Secret store references in `MCPServer` resolve correctly via the referenced secret store
- `dapr.mcp.<mcpserver-name>.ListTools` and `dapr.mcp.<mcpserver-name>.CallTool` workflow design agreed upon

**Compatibility:**
- No changes required to existing Dapr resources
- Works in both Kubernetes mode and standalone (self-hosted) mode
- daprd starts successfully when the `MCPServer` CRD is not installed; the absence of the CRD is treated as "no MCPServer resources" and produces no startup error (same behaviour expected of `HTTPEndpoint` and `Resiliency`)

---

## Completion Checklist

### dapr/dapr
- [ ] Proto: add `ListMCPServers`, `GetMCPServer`, `MCPServerUpdate` RPCs and message types to `dapr/proto/operator/v1/operator.proto`; regenerate
- [ ] CRD type: `pkg/apis/mcp/v1alpha1/types.go` — `MCPServer`, `MCPServerSpec`, `MCPServerList`, `Auth`; `register.go` files
- [ ] Loaders: `pkg/internal/loader/kubernetes/mcpservers.go`, `pkg/internal/loader/disk/mcpservers.go`
- [ ] Component store: `pkg/runtime/compstore/mcp.go` — Add/Get/List/Delete; add `mcpServers` field to `compstore.go`
- [ ] Processor: `pkg/runtime/processor/mcp.go` — `AddPendingMCPServer`, `processMCPServers`; extend `processor.go` with channel and goroutine
- [ ] Operator API: `pkg/operator/api/mcp.go` — List, Get, streaming Watch; wire informer in `api.go`
- [ ] Runtime: `loadMCPServers()`, `flushOutstandingMCPServers()`, hot-reload watch subscription in `pkg/runtime/runtime.go`
- [ ] Built-in MCP worker: `pkg/runtime/mcp/worker.go`, `transport.go`, `types.go` — `dapr.mcp.<mcpserver-name>.ListTools` + `dapr.mcp.<mcpserver-name>.CallTool` orchestrations and activities using `modelcontextprotocol/go-sdk`; include `beforeCallTool`/`afterCallTool`/`beforeListTools`/`afterListTools` middleware hook pipelines executed in array order with abort-on-error (before) / log-and-continue (after) semantics
- [ ] WFEngine activation: extend wfengine startup condition so that the presence of any loaded MCPServer manifest also triggers the workflow runtime (in addition to the existing workflow-client-connected trigger) — `pkg/runtime/runtime.go`
- [ ] Workflow router: add a router on the sidecar-side workflow client that dispatches `dapr.mcp.*` workflow names to the built-in MCP worker and all other names to the user app's registered worker — `pkg/runtime/wfengine/` or `pkg/runtime/wfruntime/`
- [ ] WFEngine integration: start built-in MCP worker in `pkg/runtime/wfengine/wfengine.go`
- [ ] Feature flag: add `MCPServerResource` feature flag in `pkg/config/`
- [ ] Unit tests: `pkg/runtime/mcp/worker_test.go` with mock backend and CompStore
- [ ] Integration tests: standalone mode with local stdio MCP server binary
- [ ] Kubernetes e2e test: `MCPServer` CRD → `dapr.mcp.<mcpserver-name>.ListTools` workflow → validate tool schemas returned
- [ ] Kubernetes e2e test: `MCPServer` CRD → `dapr.mcp.<mcpserver-name>.CallTool` workflow

### dapr-agents
- [ ] `DaprMCPWorkflowClient`: `dapr_agents/tool/mcp/dapr_workflow_client.py` — `connect()`, `get_all_tools()`, `_make_call_tool()`
- [ ] Reuse `create_pydantic_model_from_schema` from `dapr_agents/tool/mcp/schema.py` for tool arg models
- [ ] Unit tests: mock `DaprClient.start_workflow`, assert tools are `WorkflowContextInjectedTool` with correct names and that invocation calls `ctx.call_child_workflow("dapr.mcp.<mcpserver-name>.CallTool", ...)`
- [ ] Quickstart example: update or add an example showing `DaprMCPWorkflowClient` with a `MCPServer` CRD

### Documentation
- [ ] Dapr docs: new `MCPServer` resource reference page (spec fields, transport options, examples)
- [ ] Dapr docs: "Using MCP servers with Dapr Agents Durably" how-to guide
- [ ] Dapr Agents docs: `DaprMCPWorkflowClient` API reference
