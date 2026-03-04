# MCPServer: A First-Class Dapr Resource for Durable MCP Server Integration

* Author(s): Samantha Coyle (@sicoyle)
* State: Ready for Implementation
* Updated: 03/04/2026

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

2. Built-in `ListTools` and `CallTool` workflow orchestrations registered inside daprd's workflow engine. These execute MCP protocol calls as durable activities, provide replay-safe checkpointing for long-running tools, and are architected to support elicitation and sampling suspensions in a follow-on phase.

3. A `DaprMCPWorkflowClient` in dapr-agents that wraps the built-in workflows as `WorkflowContextInjectedTool` instances, giving `DurableAgent` workflows access to MCP servers via `ctx.call_child_workflow("CallTool", ...)` — the same pattern used for agent to agent calls, or agents as tools usage today.

---

## Background

The Model Context Protocol (MCP) has rapidly become the standard interface for exposing tools, resources, and prompts to LLM-based agents. Dapr Agents today supports MCP via direct client connections (`MCPClient`), but this approach has several gaps for enterprise and production use:

**Durability gap.** Direct MCP connections in Python are not durable. If the agent process or sidecar crashes mid tool call, the call is lost. Dapr's workflow engine provides durable execution with checkpoint-and-resume semantics, but MCP calls made outside the workflow context do not benefit from this.

**Human-in-the-loop gap.** The MCP specification includes `elicitation/create` (server requests user input mid-call) and `sampling/createMessage` (server requests LLM completion mid-call). Both require the ability to suspend execution and wait — potentially for minutes or hours — before resuming. A simple call with retry/resiliency policy cannot model this. Workflow orchestrations with `wait_for_external_event` can.

**Governance gap.** There is currently no centralized place for operators to declare which MCP servers an application is allowed to use, with what credentials, at what timeout, and with what transport. Each application hardcodes its own connection logic. A `MCPServer` CRD creates a single source of truth, enables namespace-scoped access control, and gives users a native resource to surface its connection details.

**Operational gap.** Existing MCP connections require manual credential management in application code. The `MCPServer` resource enables secret store references, so API keys and headers are never hardcoded.

**Why workflow rather than activity + resiliency?**

For today's common case (fast, idempotent tool calls), an activity with a resiliency policy is technically sufficient. Workflow is the right choice for three reasons:

1. **API stability**: If `CallTool` is implemented as `ctx.call_activity(...)` today and later needs to support elicitation (which requires `wait_for_external_event`), the call site in every agent must change to `ctx.call_child_workflow(...)`. Implementing as a workflow from the start means the call site — and every agent's code — never changes as the implementation gains capabilities.

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
- Operator gRPC extensions: `ListMCPServers`, `GetMCPServer`, `UpdateMCPServer`
- Kubernetes and standalone (disk) loaders for `MCPServer`
- Secret processing for headers and env vars via existing `auth.secretStore`
- Runtime loading in `loadMCPServers()` during daprd startup, with hot-reload via Operator watch
- Built-in `ListTools` and `CallTool` workflows defined and registered with daprd's wfengine using the `modelcontextprotocol/go-sdk`
    - This is a fundamental change that brings the idea of a Managed Workflow in essence into the runtime.
- Transport support: `streamable_http`, `sse`, `stdio`

> NOTE: I did choose the [official modelcontextprotocol go-sdk pkg](https://github.com/modelcontextprotocol/go-sdk); however, there is the [Mark3Labs alternative](https://github.com/mark3labs/mcp-go) which is more "popular" and established.

### Explicitly not in scope (initial phase)

- Elicitation (`elicitation/create`) callback handling — architecturally enabled by the workflow design but not implemented in v1
- Sampling (`sampling/createMessage`) callback handling — same
- MCP resource subscriptions and prompt listing (tools only in v1) - https://modelcontextprotocol.io/docs/learn/server-concepts#core-server-features
- Cross-namespace MCPServer access
- MCP authorization / per-tool RBAC

### Alternatives considered

**Alternative 1: Reuse the existing Component CRD with a new `mcpserver` category.**

Reusing the Component infrastructure would reduce the code to write significantly. 
However: 
- MCPServer semantic is fundamentally different from I/O components. It is a tool provider, not a StateStore or Binding
- This would then require a `components-contrib` implementation, which adds unnecessary indirection for a protocol-level integration.

**Alternative 2: External proxy workflow appid.**

An external appid to potentially host the ListTools/CallTool workflows avoids embedding MCP in daprd and provides independent scaling. 
Disadvantages:
- Requires an additional Kubernetes deployment if using Kubernetes mode
- Access policy configuration between two appids

Implementation within daprd runtime is simpler operationally and durability is transparent within the sidecar.

**Alternative 3: Not wrapping ListTools and CallTool within a workflow**

Simpler to implement and sufficient for the common case.
However, ruled out because:
- it creates an API cliff when elicitation/sampling support is needed
- it cannot model mid-call suspension
- it provides no checkpointing for long-running tools if the app where to go down mid MCP tool execution

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
A error will be used when stdio is used in Kubernetes mode.
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
  # appID: if set, routes to a Dapr app via service invocation rather than a raw URL.
  # The sidecar constructs the endpoint automatically:
  #   http://<appID>.<namespace>.svc.cluster.local:3500/v1.0/invoke/<appID>/method/mcp
  # Omit appID and use endpoint.url for external (non-Dapr) MCP servers.
  appID: payments-mcp

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
    transport: sse
    # protocolVersion pins the MCP spec version the server implements.
    # Used by the go-sdk client to negotiate correctly.
    protocolVersion: "2026-03-26"

  auth:
  # Option 1 using OAuth2:
    oauth2:
      issuer: https://auth.company.com
      audience: mcp://payments
      scopes:
        - payments.read
        - payments.refund
      secretRef:
        name: payments-mcp-oauth
        key: clientSecret
  # Option 2 env vars checked if Oauth2 fields not found.

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
    transport: streamable_http
    url: https://api.githubcopilot.com/mcp/
    protocolVersion: "2025-03-XX"
  headers:
    - name: Authorization
      secretRef:
        name: github-token
        key: token
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
    transport: stdio
  stdio:
    command: python
    args: ["-m", "my_tools.py"]
    env:
      - name: EXAMPLE
        value: EXAMPLE
```

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
    // AppID routes to a Dapr app via service invocation.
    // Mutually exclusive with endpoint.url.
    AppID    string            `json:"appID,omitempty"`
    Catalog  *MCPServerCatalog `json:"catalog,omitempty"`
    Endpoint MCPEndpoint       `json:"endpoint"`
    Auth     MCPAuth           `json:"auth,omitempty"`
    // Stdio is only valid when endpoint.transport is "stdio".
    Stdio    *MCPStdioSpec     `json:"stdio,omitempty"`
}

type MCPEndpoint struct {
    Transport       string `json:"transport"`
    URL             string `json:"url,omitempty"`
    ProtocolVersion string `json:"protocolVersion,omitempty"`
    Timeout         string `json:"timeout,omitempty"`
}

type MCPStdioSpec struct {
    Command string                 `json:"command"`
    Args    []string               `json:"args,omitempty"`
    Env     []common.NameValuePair `json:"env,omitempty"`
}

type MCPAuth struct {
    OAuth2    *MCPOAuth2    `json:"oauth2,omitempty"`
}

type MCPOAuth2 struct {
    Issuer   string   `json:"issuer"`
    Audience string   `json:"audience,omitempty"`
    Scopes   []string `json:"scopes,omitempty"`
    SecretRef *MCPSecretRef `json:"secretRef,omitempty"`
}

type MCPSecretRef struct {
    Name string `json:"name"`
    Key  string `json:"key"`
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
#### Built-in workflow orchestration flow

```
<agent_name>_agent_workflow (DurableAgent)
  └─ ctx.call_child_workflow("ListTools", {mcp: "github-mcp"})
       └─ [daprd built-in ListTools orchestration]
            └─ ctx.CallActivity("list-tools", input)
                 └─ [daprd built-in activity]
                      ├─ looks up MCPServer CRD from CompStore
                      ├─ dials MCP server via go-sdk
                      └─ returns []mcp.Tool as JSON

<agent_name>_agent_workflow
  └─ ctx.call_child_workflow("CallTool", {mcp: "github-mcp", tool: "search_code", arguments: {...}})
       └─ [daprd built-in CallTool orchestration]
            └─ ctx.CallActivity("call-tool", input)
                 └─ [daprd built-in activity]
                      ├─ looks up MCPServer CRD from CompStore
                      ├─ dials MCP server, calls tool
                      └─ returns tool result
```

#### dapr-agents usage

```python
from dapr_agents.tool.mcp.dapr_workflow_client import DaprMCPWorkflowClient

# At agent startup (before workflow execution):
client = DaprMCPWorkflowClient()
await client.connect("github-mcp")   # invokes ListTools workflow, caches schemas
tools = client.get_all_tools()       # returns WorkflowContextInjectedTool instances

agent = DurableAgent(
    name="my_agent",
    tools=tools,   # each tool calls CallTool workflow when invoked by LLM under the hood
    llm=DaprChatClient(component_name="openai"),
)
```

The `DurableAgent`'s existing dispatch loop handles `WorkflowContextInjectedTool` with no changes:

```python
# durable.py — no changes needed
if isinstance(tool_obj, WorkflowContextInjectedTool):
    workflow_tasks.append(tool_obj(ctx=ctx, _source_agent=self.name, **args))
    # → internally calls ctx.call_child_workflow("CallTool", {...})
```

#### Architecture diagram

```
┌─────────────────────────────────────────────────────────┐
│  User app (dapr-agents DurableAgent)                    │
│                                                         │
│  <agent_name>_agent_workflow                            │
│    └─ call_child_workflow("ListTools", {mcp: ...})      │
│    └─ call_child_workflow("CallTool",  {mcp: ...})      │
└────────────────────┬────────────────────────────────────┘
                     │ Dapr Workflow API
┌────────────────────▼────────────────────────────────────┐
│  daprd sidecar                                          │
│                                                         │
│  ┌─ wfengine (actor backend) ─────────────────────┐     │
│  │  Built-in MCP worker                           │     │
│  │    ListTools workflow orchestration            │     │
│  │      └─ dapr-mcp-list-tools activity           │     │
│  │    CallTool workflow orchestration             |     │
│  │      └─ dapr-mcp-call-tool activity            │     │
│  └────────────────────────────────────────────────┘     │
│                                                         │
│  CompStore: MCPServer CRDs (loaded from Operator/disk)  │
└────────────┬────────────────────────────────────────────┘
             │ MCP protocol (streamable_http / SSE / stdio)
    ┌────────▼──────┐  ┌──────────────┐  ┌──────────────┐
    │ GitHub MCP    │  │ Dapr MCP     │  │ Custom MCP   │
    │ Server        │  │ Server       │  │ Server       │
    └───────────────┘  └──────────────┘  └──────────────┘
```

### Feature lifecycle outline

**Alpha (initial release):**
- `MCPServer` resource gated behind a feature flag `MCPServerResource` in Dapr Configuration
- Built-in `ListTools` and `CallTool` workflows available when flag is enabled
- All three transports supported but stdio in Kubernetes mode errors
- Dapr CLI to error if using slim mode as workflows are needed for this resource
- No elicitation or sampling support yet
- MCPServer appears in daprd metadata API response

**Future:**
- Feature flag removed; MCPServer enabled by default
- Elicitation callback support added (workflow `wait_for_external_event` pattern)
- Sampling support added

**Compatibility:**
- The `ListTools` and `CallTool` workflow names are stable API. Agents coded against them will not need changes as internal implementation evolves (e.g., elicitation support is added inside the workflow without changing the input/output contract).
- `MCPServer` CRD spec fields are additive; existing fields will not be removed in minor versions.
- The existing `MCPClient` in dapr-agents will be removed. `DaprMCPWorkflowClient` is recommended approach for durable workflows and MCP interactions.

### Acceptance Criteria

**Functional:**
- A `DurableAgent` can call tools on a remote `streamable_http` MCP server declared as a `MCPServer` CRD, with the MCP tool call durably recorded in workflow history
- If daprd restarts mid tool call, the activity retries automatically since it's within a workflow
- `MCPServer` CRDs are hot-reloaded when updated in Kubernetes (no sidecar restart required)
- Secret store references in `MCPServer` resolve correctly via the referenced secret store
- `ListTools` and `CallTool` workflow design agreed upon

**Compatibility:**
- No changes required to existing Dapr resources
- Works in both Kubernetes mode and standalone (self-hosted) mode

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
- [ ] Built-in MCP worker: `pkg/runtime/mcp/worker.go`, `transport.go`, `types.go` — `ListTools` + `CallTool` orchestrations and activities using `modelcontextprotocol/go-sdk`
- [ ] WFEngine integration: start built-in MCP worker in `pkg/runtime/wfengine/wfengine.go`
- [ ] Feature flag: add `MCPServerResource` feature flag in `pkg/config/`
- [ ] Unit tests: `pkg/runtime/mcp/worker_test.go` with mock backend and CompStore
- [ ] Integration tests: standalone mode with local stdio MCP server binary
- [ ] Kubernetes e2e test: `MCPServer` CRD → `ListTools` workflow → validate tool schemas returned
- [ ] Kubernetes e2e test: `MCPServer` CRD → `CallTool` workflow

### dapr-agents
- [ ] `DaprMCPWorkflowClient`: `dapr_agents/tool/mcp/dapr_workflow_client.py` — `connect()`, `get_all_tools()`, `_make_call_tool()`
- [ ] Reuse `create_pydantic_model_from_schema` from `dapr_agents/tool/mcp/schema.py` for tool arg models
- [ ] Unit tests: mock `DaprClient.start_workflow`, assert tools are `WorkflowContextInjectedTool` with correct names and that invocation calls `ctx.call_child_workflow("CallTool", ...)`
- [ ] Quickstart example: update or add an example showing `DaprMCPWorkflowClient` with a `MCPServer` CRD

### Documentation
- [ ] Dapr docs: new `MCPServer` resource reference page (spec fields, transport options, examples)
- [ ] Dapr docs: "Using MCP servers with Dapr Agents Durably" how-to guide
- [ ] Dapr Agents docs: `DaprMCPWorkflowClient` API reference
