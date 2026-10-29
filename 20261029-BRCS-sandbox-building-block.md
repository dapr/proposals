# Sandbox building block: isolated microVMs addressed as Dapr actors

* Author(s): Xavier Geerinck
* State: Draft
* Discussion: [dapr/proposals#110](https://github.com/dapr/proposals/issues/110)
* Proof of concept: [XavierGeerinck/poc-dapr-sandbox](https://github.com/XavierGeerinck/poc-dapr-sandbox) (driver, benchmark, deploy scripts, raw results) and the Dapr branch [feat/sandbox-building-block](https://github.com/XavierGeerinck/dapr/tree/feat/sandbox-building-block)
* Updated: 2026-10-29
* Flags: B (building block), R (runtime), C (components), S (SDKs)

## Overview

> Paths such as `results/…`, `bench/`, `deploy/` and `proposal/dapr-poc-patches/` refer to the [proof-of-concept repository](https://github.com/XavierGeerinck/poc-dapr-sandbox). Result summaries and charts are also in [`resources/20261029-BRCS-sandbox-building-block/`](resources/20261029-BRCS-sandbox-building-block/).

This proposal adds an alpha **Sandbox** building block to Dapr. A sandbox is an isolated
environment, here a microVM, in which an app runs commands it does not trust.

**What it is**
- A sandbox ID is a Dapr actor ID. It belongs to an internal actor type that daprd hosts
  itself, the way it hosts workflow actors.
- The VM comes from a **driver**: a Dapr component behind a gRPC contract that does not depend
  on any one VMM.
- The first call for an unknown ID activates it. The driver then does one of three things:
  - hands out a running VM from a warm pool (POOL);
  - restores one from a template snapshot (RESTORE);
  - boots one (BOOT).

**Areas affected**
- **Runtime (daprd):**
  - a new alpha API, `/v1.0-alpha1/sandboxes`, over HTTP and gRPC;
  - a new internal actor target;
  - a new component category, `sandbox`.
- **Components:** a new pluggable component contract, `dapr.proto.components.v1.Sandbox`. There
  is also a built-in `sandbox.process` driver without isolation, used for tests and as the
  reference implementation; it is opt-in (see [Inside daprd](#inside-daprd)).
- **SDKs:** new alpha client methods.
- **Docs:** a new building block page and component reference.

**What a proof of concept measured**
- **Setup.** The PoC is a `dapr/dapr` branch plus a microVM driver. It ran on Kubernetes: kind on
  a bare-metal Linux host with KVM.
- **Metric.** The time from the caller's sidecar receiving the request to it sending the
  response.
- **Result.** For a new sandbox ID served from the warm pool, the first command ran at:
  - p99 16.44 ms when requests came one after another (482 of 500 requests; the other 18
    found the pool empty and were served by a restore, see the caveats);
  - p99 11.36 ms at 20 new sandboxes per second per host pod (all 3600 from the pool).
- **Routing.** Every request crossed pods, because the caller pod hosts no sandboxes.
- [Acceptance Criteria](#acceptance-criteria) has the details and the caveats.

## Background

**The problem**
- Developers who run untrusted code need one isolated environment per session. Examples: AI
  agents, function execution, notebooks.
- The environment must start fast, behave the same on a laptop and on Kubernetes, and not
  require a separate orchestration system.

**What exists today**
- **Kata Containers** gives each pod its own VM. A new sandbox is then a new pod, so it pays for
  pod scheduling and for a VM boot.
- **Dedicated sandbox products** solve the problem outside Dapr:
  - microsandbox;
  - Azure Container Apps dynamic sessions;
  - kubernetes-sigs/agent-sandbox.

**Why Dapr fits.** Dapr already solves the distributed part of the problem with virtual actors:
- **One live instance.** An actor has exactly one live instance per ID across the cluster.
- **Routing.** Every call for that ID goes to the pod that owns it. The lookup is local: the
  placement table is disseminated to each sidecar, so a call is one mTLS gRPC hop from the
  caller's sidecar to the owner's sidecar.
- **Turn-based concurrency.** Calls to one actor run one at a time.
- **Idle cleanup.** Idle actors are deactivated.
- **Internal actors.** Workflows show that daprd can host actor types itself, with no app in the
  loop. These are internal actor targets in `pkg/actors/targets`, registered from
  `pkg/runtime/wfengine/backends/actors/actors.go`.

A sandbox maps onto this directly: the sandbox is the actor, and its VM is the actor's state in
memory.

## Related Items

### Related proposals

None filed. This proposal builds on two things that exist:
- **Workflows' internal actor targets.** The sandbox target is modelled on them.
- **The binary store alpha API** (`/v1.0-alpha1/binarystore/{name}/{fileName}`). It is the
  candidate target for the flexible profile's save and restore (phase 2).

### Related issues

- **dapr/dapr#9128 (multi-cluster).** A Dapr maintainer confirmed there that multi-cluster
  would need a shared scheduler, a flat network and a shared trust domain. Cross-cluster
  sandboxes are therefore future work here.
- **[dapr/proposals#110](https://github.com/dapr/proposals/issues/110)**, the discussion issue for this proposal.
- **An upstream issue for microsandbox** about a guest-kernel exec delay that the PoC found and
  worked around (see [Findings](#what-the-measurements-showed)). Drafted in
  [`proposal/microsandbox-issue.md`](https://github.com/XavierGeerinck/poc-dapr-sandbox/blob/main/proposal/microsandbox-issue.md); not filed yet.

## Expectations and alternatives

### In scope

- **The fast profile** (functions, AI agents), built and measured in the PoC.
- **The flexible profile** (notebooks), described only.
- **One API for both profiles.** It covers one-shot exec, long-lived sessions, create with a
  sidecar-chosen ID, and delete.
- **The driver contract.** It does not depend on a VMM, so other drivers can implement it.

[Two profiles](#two-profiles) describes both.

### Not in scope (future work)

- **The flexible profile's save and restore to the binary store** (phase 2).
- **Cross-cluster sandboxes.** Placement and the scheduler are per cluster. dapr/dapr#9128
  lists what multi-cluster would need.
- **A Cloud Hypervisor driver for `/dev/mshv`**, which AKS Pod Sandboxing nodes use.
  libkrun and microsandbox do not support mshv; Cloud Hypervisor does.
- **Native Windows (WHP).** Pluggable components use Unix domain sockets, and Dapr's own tests
  skip them on Windows (`pkg/components/pluggable/grpc_test.go`). Windows developers can run
  the Linux setup inside WSL2 with nested virtualization, which provides `/dev/kvm`. The PoC
  did not test WSL2.
- **An unprivileged KVM device plugin.** It would replace the privileged driver container.
- **Capacity-aware placement.** Placement has no weights and no per-host actor limit (see
  [Known limitations](#known-limitations)).
- **Warm-pool refill tuning.** Refill throughput is the next bottleneck (see
  [Findings](#what-the-measurements-showed)).
- **Production hardening:**
  - a per-sandbox owner binding;
  - multi-tenancy policy;
  - a network egress policy beyond "no network".

### Alternatives considered

1. **An app on Dapr actors.** A regular app registers a sandbox actor type, implements the
   app-side actor contract, and drives the VMM.
   - **For:** it needs no runtime change.
   - **Against:**
     - The app sits in every call path, as an extra hop from the sidecar to the app. (This hop
       was not measured.)
     - Every team builds and operates its own.
     - It gets no component YAML, no first-class API and no SDK methods.
     - Each app would rebuild the pool, the readiness gating and the admission control
       described below.
2. **Kata Containers.** It gives strong isolation per pod, but a sandbox is then a pod:
   - creating one goes through the Kubernetes API, the scheduler and a VM boot;
   - there is no warm pool of running VMs to hand out by ID;
   - it isolates pods, so addressing, routing and cleanup of sandboxes still have to be built
     on top.

   Kata could serve as a node-level runtime under a driver, but it does not replace the
   addressing.
3. **kubernetes-sigs/agent-sandbox.** It is Kubernetes-native: custom resources and a
   controller, where each sandbox is a pod and isolation comes from the pod's runtime class.
   - It needs a cluster, so it does not give the same API on a laptop.
   - It does not use Dapr's addressing, mTLS identities or resiliency policies.
4. **Azure Container Apps dynamic sessions.** A managed service with sessions keyed by an
   identifier and a pre-warmed pool. It is Azure-only and cannot run self-hosted or on another
   cluster.
5. **Workflows-style internal actors in daprd. This is the proposal.** It reuses placement,
   routing, turn-based locking, idle deactivation and mTLS. The new parts are:
   - one actor target;
   - one component category;
   - one API group.

### Trade-offs

**Trade-offs made**
- **Speed for durability.** The fast profile keeps no state outside the VM, so the hot path has
  no state-store write. The cost: a sandbox is lost when placement moves it (see
  [Lost sandboxes](#lost-sandboxes)).
- **Generality for simplicity.** The template guarantees a warm disk and page cache only, not a
  running interpreter with libraries already imported (see
  [Known limitations](#known-limitations)).
- **Privileges.** On Kubernetes, the host deployment's driver container runs privileged until a
  KVM device plugin exists (see [Operational requirements](#operational-requirements)).

**Advantages**
- One API on a laptop (self-hosted) and on Kubernetes.
- Configured with component YAML.
- Untrusted code runs only on a known, isolated set of pods: the sandbox host deployment.

**Disadvantages**
- New surface in daprd.
- A pluggable driver per isolation technology.

## Implementation Details

### Design

#### User-facing shape

A sandbox component, fast profile:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: py-fast
spec:
  type: sandbox.microvm          # the pluggable driver registers as sandbox.microvm
  version: v1
  metadata:
  - name: image
    value: python:3.12-slim
  - name: setup                  # runs once in the template VM, before the snapshot
    value: mount -o remount,favordynmods /sys/fs/cgroup
  - name: warmPoolSize
    value: "8"
  - name: refillConcurrency
    value: "4"
  - name: maxSandboxesPerHost
    value: "32"
  - name: memoryMiB
    value: "512"
scopes:
- sandbox-pool                   # only the host deployment loads it
```

Component metadata parsed by daprd (`pkg/components/sandbox/config.go`):

| Key | Default | Meaning |
|---|---|---|
| `image` | required | OCI image of the sandbox |
| `cpus` | `1` | vCPUs per VM |
| `memoryMiB` | `512` | memory per VM |
| `setup` | empty | shell command (`sh -c`) run once in the template VM before the snapshot; with `template: none`, run in every VM |
| `template` | `snapshot` | `snapshot`: pool VMs are restores of one template snapshot. `none`: every VM boots from the image. |
| `warmPoolSize` | `4` | running VMs kept ready per host |
| `refillConcurrency` | `2` | pool refills that run in parallel |
| `maxSandboxesPerHost` | `32` | admission limit per host; activation beyond it fails fast |
| `idleTimeout` | `5m` | a sandbox with no calls for this long is released |

daprd rejects a component with `cpus` or `memoryMiB` of 0, or an `idleTimeout` that is not
positive.

The PoC's microVM driver adds one key of its own: `warmupCommand`. It is off by default (see
[Findings](#what-the-measurements-showed)).

Different environments are different components. A dependency can come from three places:
1. **The image.** Built with a normal Dockerfile.
2. **The template snapshot.** `setup` runs once, before the snapshot.
3. **Runtime installs by the sandbox itself.** In the fast profile they are lost on
   deactivation.

#### API (alpha)

HTTP routes, all under `/v1.0-alpha1/sandboxes`:

| Method and route | gRPC (`dapr.proto.runtime.v1.Dapr`) | What it does |
|---|---|---|
| `POST /{component}` | `SandboxCreateAlpha1` | Creates a sandbox with an ID chosen by the sidecar and returns the ID. If the host is full, it retries with a fresh ID, up to 5 attempts. |
| `POST /{component}/{sandboxId}/exec` | `SandboxExecAlpha1` | Runs a one-shot command. The first call for an unknown ID activates it. |
| `DELETE /{component}/{sandboxId}` | `SandboxDeleteAlpha1` | Releases the sandbox. Idempotent. It does not wait for a running command: that command ends, and its caller gets 410 `ERR_SANDBOX_LOST`. Calls that had not started yet (queued for the turn, or still acquiring) run on a fresh sandbox under the same ID, unless they carry `expectedActivation`. |
| `POST /{component}/{sandboxId}/sessions` | `SandboxSessionStartAlpha1` | Starts a long-lived process and returns a session ID. |
| `POST /{component}/{sandboxId}/sessions/{sessionId}` | `SandboxSessionSendAlpha1` | Writes input to the session, then reads output until a marker (`until`), the process exits, or `timeoutMs` passes. |
| `DELETE /{component}/{sandboxId}/sessions/{sessionId}` | `SandboxSessionCloseAlpha1` | Closes the session. |

**Requests**
- **Exec fields:** `command`, `args`, `env`, `cwd`, `stdin` and `timeoutMs`.
- **Target app ID:** it defaults to the caller's own app ID.
  - **HTTP:** the `dapr-app-id` header or `appId` in the body; the header wins when both are
    set.
  - **gRPC:** the `app_id` field. The header is HTTP-only.
- **`expectedActivation`:** exec, session start and session send take it optionally (see
  [Lost sandboxes](#lost-sandboxes)). Session close and delete do not.

**Responses**
- **Fields:** `stdout`, `stderr`, `exitCode`, `truncated` and `timedOut`.
- **Activation token:** `activation`.
- **Timing:** `timing` (`totalUs`, `hostUs`, `activateUs`, `execUs`, `acquirePath`). The
  benchmark reads this, and it lets any caller see where the time went.

Errors:

| HTTP | gRPC | `errorCode` | Meaning |
|---|---|---|---|
| 400 | InvalidArgument | `ERR_SANDBOX_NOT_FOUND` | no such sandbox component |
| 400 | InvalidArgument | `ERR_SANDBOX_INVALID_REQUEST` | bad request, for example an unknown session |
| 410 | FailedPrecondition | `ERR_SANDBOX_LOST` | the sandbox was lost or deleted, or `expectedActivation` is stale |
| 429 | ResourceExhausted | `ERR_SANDBOX_HOST_FULL` | no host can take it now: no host is ready for this sandbox type, the host app ID is unknown, the host is at capacity, or its driver is unavailable or gave no sandbox in time. Retry later. |
| 500 | Internal | `ERR_SANDBOX_EXEC` | the driver failed (the message carries the first line of the driver's error, shortened), or the call was interrupted by actor rebalancing and the command may have partly run |

#### Hosting model

Dapr only lets an app host the internal actor types it owns, `dapr.internal.<namespace>.<appID>.*`.
So each sandbox component becomes the actor type
**`dapr.internal.<namespace>.<hostAppID>.sandbox.<component>`** on the app that loads it.

- **Kubernetes:** sandboxes run in a dedicated host deployment, for example app ID
  `sandbox-pool`.
  - Only those pods load the sandbox component (component `scopes`).
  - Only those pods get `/dev/kvm` and the driver container.
  - Any other app in the namespace calls them through its own sidecar, passing the target app
    ID. This works like calling actors that another app hosts.
- **Self-hosted (local development):** one daprd both hosts and calls, and the target app ID
  defaults to itself.
- **Callers need no actor state store.** Only the host app needs one, because hosting actors
  requires it.

#### Inside daprd

1. **Component category `sandbox`.** It is loaded like the other categories. A pluggable
   component registers as `sandbox.<name>`, and daprd finds it in the components socket
   folder.
2. **Driver contract** `dapr/proto/components/v1/sandbox.proto`, service `Sandbox`:

   | RPC | Purpose |
   |---|---|
   | `Init`, `Features`, `Ping` | the usual component lifecycle |
   | `Ready` | template and warm pool are ready |
   | `Acquire` | claim from the pool, restore or boot. It returns the instance ID and the path: `POOL`, `RESTORE` or `BOOT`. `RESOURCE_EXHAUSTED` when full. |
   | `Exec` | one-shot command |
   | `SessionStart`, `SessionSend`, `SessionClose` | long-lived processes |
   | `Release` | returns at once; the driver tears the VM down in the background |
   | `Reconcile` | the live instance IDs, so the driver can reap VMs daprd no longer owns |

   No VMM type appears in the contract.

   **Open items in the contract** (future work; the spec asks for the first and third):
   - **Streaming exec output.** `Exec` is unary, so the PoC measures time to completion, not
     time to the first byte of output. The benchmark runs `echo ok`, so the response follows
     its output directly.
   - **A component close or remove RPC**, so that a driver learns that a component went away.
     Today the PoC's driver keeps a removed component's warm pool, across driver restarts,
     until its state file is deleted.
   - **A capacity and stats RPC.**
3. **Internal actor target** (`pkg/actors/targets/sandbox`), one actor type per sandbox
   component:
   - **Activation:** `Acquire` from the driver, bounded at 60 s. Past that bound, or when the
     driver is unavailable, the call answers 429 (retry later).
   - **`InvokeMethod`:** exec and session operations. A release does not wait for the running
     call's turn: it ends the running command at once (see the API table).
   - **Deactivation:** `Release`, asynchronously. Deactivation holds the per-type lock on the
     host, so a slow release would otherwise block new activations of that type.
4. **Readiness before placement** (`pkg/runtime/sandboxhost`). daprd registers the sandbox actor
   type with placement only after the driver's `Ready` reports that the template and the warm
   pool are ready. Until then, calls for that component answer 429 (retry later), whether they
   come from the host app itself or from another app.
   - **At first start,** no sandbox lands on a pod that is still building its pool.
   - **After a driver restart,** daprd does not check `Ready` again and the type stays
     registered. Until the restarted driver serves, a new sandbox ID routed to that pod answers
     429 (retry later), and the sandboxes of the old driver process are lost (410).
5. **Admission.** At `maxSandboxesPerHost`, `Acquire` fails fast with `RESOURCE_EXHAUSTED`, and
   the API answers 429. Callers rely on Dapr resiliency policies. Placement does not change.
6. **The built-in `sandbox.process` driver** (`pkg/components/sandbox/process`) runs commands as
   host processes, without isolation. It is for unit tests, integration tests and CI, and it is
   the reference implementation of the contract. It is opt-in twice:
   - it is not in stable builds, only in builds with the `allcomponents` or `sandbox_process`
     tag (not on Windows);
   - its Init fails unless the component sets the metadata `allowNoIsolation: "true"`.

#### Acquire paths, the warm pool and the template

**On startup, the driver prepares each component:**
1. it pulls the image;
2. it boots one VM;
3. it runs `setup`;
4. it takes a full snapshot of that VM as the template;
5. it fills the warm pool by restoring from the template;
6. it reports ready.

**Acquire paths**
- **POOL (path a).** A new ID takes a running VM from the pool, and the pool refills in the
  background.
- **RESTORE (path c).** When the pool is empty, activation restores from the template on demand.
- **BOOT (path d).** With `template: none`, every VM boots from the image.
- **Warm exec (path b).** An exec on an existing ID goes straight to its VM.

#### Two profiles

**Fast profile** (functions, AI agents). Built in the PoC.
- A fixed environment per component, a template snapshot and a warm pool.
- The VM is thrown away on deactivation.

**Flexible profile** (notebooks). Proposed for phase 2; not in the PoC.
- **Per-sandbox spec.** An explicit create, `PUT /v1.0-alpha1/sandboxes/{component}/{id}` with a
  spec, for a custom image or resources. The spec is stored in actor state, so any pod can
  reactivate the sandbox.
- **Runtime package installs** that survive deactivation, through the save and restore below.
- **Save and restore.** On deactivation, a full VM save (disk and memory) to the binary store;
  on activation, a restore from it.
- **Caveats:**
  - open network connections do not survive;
  - a restore on a different CPU type may fail, in which case it falls back to a disk-only
    restore.

#### Lost sandboxes

Dapr halts actors whenever placement changes for them:
- the placement stream drops;
- the hash ring moves because a host pod joins or leaves, for example on scale-out, a rolling
  update or a failed health check.

**What that means for a sandbox**
- Its VM and sessions are gone, and the next call would silently get a fresh sandbox.
- The PoC does not try to keep VMs across a halt. It makes the loss visible instead, without
  adding a state write to the hot path.

**How the loss is made visible**
- Every response carries the sandbox's **activation token**, a new value per activation.
- A caller may send it back as `expectedActivation`. If the sandbox was recreated in the
  meantime, or the driver no longer knows its VM, the call fails with 410
  `ERR_SANDBOX_LOST` and does not run in a fresh VM.
- Callers that do not send it get the usual virtual-actor behaviour.

**Calls during a rebalance**
- While placement moves a sandbox type's hash ring, new calls for that type stall: the type is
  locked until the new table is in place.
- In-flight calls of that type get the drain timeout to finish: 2 s when the host app sets
  none. Then their placement claim is cancelled, which cancels the driver call, and the driver
  kills the command.
- Such a command never runs again. Its caller gets 500 `ERR_SANDBOX_EXEC` with "sandbox call
  interrupted by actor rebalancing: the command may have partly run", or 410
  `ERR_SANDBOX_LOST` if the sandbox was halted first.
- A call that was still waiting for its turn has not run, and Dapr retries it on the new host
  as usual.
- One edge case remains at-least-once: a remote host that ran the command but whose
  connection drops before the response arrives. See
  [`docs/poc-known-gaps.md`](https://github.com/XavierGeerinck/poc-dapr-sandbox/blob/main/docs/poc-known-gaps.md).

#### Full hosts

**Caller-chosen IDs.** Hashing pins an ID to one host. If that host is at
`maxSandboxesPerHost`, retrying the same ID keeps failing until capacity frees up.

**Sidecar-chosen IDs** (`POST /v1.0-alpha1/sandboxes/{component}`) avoid that:
- On a full host, the sidecar picks a new random ID, which most likely hashes to a different
  host.
- This needs no change to placement.
- It is the recommended way to get a fresh sandbox.

#### The microVM driver in the PoC

- **Language and transport.** Rust, serving the contract over a Unix socket as a pluggable
  component.
- **Backend.** libkrun through microsandbox 0.7.11: KVM on Linux, Hypervisor.framework on macOS
  (Apple silicon). microsandbox is one driver behind the contract, not part of the design.
- **Pool VMs.** Copy-on-write restores of the template snapshot, with no network access.
- **One driver process per daprd.** The driver's `Reconcile` releases every VM of a component
  that the calling daprd does not list, so two daprds sharing one driver (one socket folder)
  destroy each other's sandboxes.
- **Other drivers.** Firecracker and Cloud Hypervisor drivers would implement the same contract.

#### Upstream changes in the PoC branch

**The branch.** `dapr/dapr` at `v1.19.0`, branch `feat/sandbox-building-block`, 27 patches in
[`proposal/dapr-poc-patches/`](https://github.com/XavierGeerinck/poc-dapr-sandbox/tree/main/proposal/dapr-poc-patches), also pushed as the branch [feat/sandbox-building-block](https://github.com/XavierGeerinck/dapr/tree/feat/sandbox-building-block). Patches 1–15 are the PoC as measured; patches 16–27
are the fix wave after the final review (see [Acceptance Criteria](#acceptance-criteria) for
what that means for the results).

| # | Patch |
|---|---|
| 1 | component contract and pluggable client |
| 2 | register the `sandbox` component category |
| 3 | built-in `sandbox.process` driver for tests |
| 4 | per-send session output cap and lifecycle tests |
| 5 | internal actor target backed by sandbox drivers |
| 6 | context-aware turn wait; review fixes |
| 7 | register sandbox actor types once drivers are ready |
| 8 | alpha runtime API (gRPC) |
| 9 | pass context errors through; review fixes |
| 10 | alpha HTTP API |
| 11 | treat an unavailable driver as lost; fail fast before ready |
| 12 | integration tests |
| 13 | lost only on `DAPR_SANDBOX_LOST`; review fixes |
| 14 | let other apps invoke hosted sandbox actors under mTLS |
| 15 | classify hosted sandbox types before workflow suffixes |
| 16 | move the sandbox actor-type helper to `pkg/components/sandbox` |
| 17 | make the no-isolation `sandbox.process` driver opt-in |
| 18 | release stops a running command; bound Acquire to 60 s |
| 19 | a driver unavailable on Acquire is retry later |
| 20 | never re-run a command cut short by a rebalance |
| 21 | name the causes in the 429 message |
| 22 | `ERR_SANDBOX_EXEC` carries the failure's message |
| 23 | reject a zero `idleTimeout`, `cpus` or `memoryMiB` |
| 24 | pluggable `Close` does not panic before Init |
| 25 | close a driver that failed to initialise |
| 26 | process driver `Reconcile` spares fresh instances |
| 27 | test cross-app sandbox calls under mTLS |

**One existing behaviour had to change: the cross-app fix (patches 14 and 15, commits f220cbd5d
and b0ce41dac).**
- **The problem.** In v1.19, `validateSameAppInternalActor` denies any call from another app ID
  to a `dapr.internal.*` actor under mTLS. That includes the sandbox host's own types, which
  exist precisely for other apps to call. The symptom: every cross-app exec on kind returned
  500 `ERR_SANDBOX_EXEC`.
- **What the patch admits.** Cross-app calls from the same namespace, only for this daprd's own
  sandbox actor prefix, `dapr.internal.<ns>.<appID>.sandbox.` followed by a non-empty
  component.
- **What stays as it was:**
  - reminders on these types stay same-app;
  - other namespaces are denied;
  - another app's sandbox types are denied;
  - workflow access policies still apply when loaded.
- **Ordering.** Patch 15 checks sandbox types before the workflow and activity suffix check, so
  a component name ending in `.workflow` cannot take the workflow path.
- **Tests.** Unit tests in `pkg/api/grpc/workflow_acl_test.go`, and an integration suite with
  Sentry identities, `daprd/sandbox/crossappmtls` (patch 27): a caller of the same namespace is
  allowed; callers from another namespace and calls to another app's sandbox type are denied.
  Without patches 14 and 15, its allowed cases fail: the direct actor call with PermissionDenied,
  the call through the sidecar as 500 `ERR_SANDBOX_EXEC` carrying that error.

#### Operational requirements

- **A larger body limit.** daprd needs `--max-body-size 16Mi` (on Kubernetes, the annotation
  `dapr.io/max-body-size: 16Mi`).
  - The driver caps each exec output stream at 4194304 bytes, and a response carries two:
    stdout and stderr.
  - daprd's default limit applies to its HTTP and gRPC servers and also to the pluggable
    component connection, so a capped exec (two streams plus framing) does not fit under it.
  - Over the limit, the client gets 429 instead of `truncated: true`. That mislabel is an open
    item.
- **A privileged driver container.** It needs this to open `/dev/kvm`.
  - An unprivileged container's device cgroup denies the device even when it is mounted, and
    kind has no KVM device plugin.
  - daprd and the caller pods are not privileged.
- **`singleProcessOOMKill: true` on the kubelet.** Without it, on cgroup v2 the kubelet sets
  `memory.oom.group=1` for the container. One OOM kill then takes the driver and every VM in
  the pod. The PoC sets it on the kind nodes, but no OOM kill was triggered live, so the
  behaviour under OOM was not exercised.
- **A postStart hook on the driver container.** It holds daprd back until the driver's socket
  exists, because daprd discovers pluggable components only when it starts.
  - This relies on daprd being a regular container.
  - `dapr.io/enable-native-sidecar` would break it: the injector then makes daprd a native
    sidecar, which starts before the driver.
- **An actor state store on the host app.** Callers need none.
- **macOS on Kubernetes is functional only.** kind on macOS runs in a Lima VM, under nested
  virtualization. It works, but its latency does not represent a real host (see
  [Acceptance Criteria](#acceptance-criteria)).

#### Known limitations

- **Sandboxes are lost on placement changes, and a command running during the change is cut
  short** (see [Lost sandboxes](#lost-sandboxes)). The flexible profile's save and restore is
  the real answer for state that must survive.
- **A full host blocks caller-chosen IDs.** The same ID keeps getting 429 until that host has
  capacity (see [Full hosts](#full-hosts)).
- **The template holds a warm disk and page cache only.**
  - **Why.** `setup` commands are processes that exit, and microsandbox refuses a full snapshot
    while one of its jobs is running. So the template does not hold a running interpreter
    with libraries already imported.
  - **The stretch goal** (`results/mac/notes.md`, "Step 5 stretch") put a running, pre-imported
    Python interpreter into the template, started from `setup` outside microsandbox's job
    mechanism, and it worked: restored VMs kept their own interpreter state.
  - **Why it is not claimed.**
    - The daemon path was bimodal by run: about 2.2–2.6 ms in half the runs and 25–29 ms in
      the others. A cold `python3 -c` that imports the same modules took 34–38 ms.
    - It needs a real session protocol.
    - It needs an explanation of the slow mode.
- **Any authenticated app in the namespace can exec in or delete any sandbox of a host app.**
  - There is no per-sandbox owner binding.
  - Dapr's `accessControl` policies do not cover actor calls.
- **Admission counts VMs, not memory.** `maxSandboxesPerHost` × `memoryMiB` can exceed the
  container's memory limit, and the limit is what bounds a pod.
  - With `singleProcessOOMKill`, an OOM kill should take the VM that grew, and that sandbox
    is lost.
  - If the driver is the largest process, it is killed instead, and every sandbox in the pod is
    lost.
  - Neither case was exercised live.
- **Network.** Pool VMs have no network access, and there is no egress policy beyond that.
- **Smaller gaps** found in the PoC's reviews and not fixed are listed in
  [`docs/poc-known-gaps.md`](https://github.com/XavierGeerinck/poc-dapr-sandbox/blob/main/docs/poc-known-gaps.md).

### Feature lifecycle outline

**Expectations**
- **Alpha.** The API is under `v1.0-alpha1`, the gRPC methods carry the `Alpha1` suffix, and the
  driver contract is a new pluggable component service.
- **Breaking changes** are allowed during alpha.
- **Before beta:**
  - a per-sandbox owner binding;
  - Kubernetes end-to-end tests;
  - SDK support;
  - a decision on where drivers live.

**Compatibility guarantees**
- **Apps that load no `sandbox` component:** no sandbox actor type is registered and no driver
  runs. The alpha routes and gRPC methods still exist in every daprd: a call for the daprd's
  own app answers 400 `ERR_SANDBOX_NOT_FOUND`, and a call with another app ID is routed to that
  app's sandbox actor type.
- **User-defined actors:** their behaviour does not change.
- **The ACL change:** it is a prefix check. Under mTLS it admits callers from the same namespace
  to any actor type that starts with `dapr.internal.<ns>.<thisAppID>.sandbox.` followed by a
  non-empty component, whether or not this daprd loads that component. Reminders on those types
  stay same-app.

**Deprecation and co-existence.** Nothing is deprecated. Sandbox actors use the same internal
actor mechanism as workflows and co-exist with them.

**Feature flags.** The PoC adds none: the building block stays inactive unless a `sandbox.*`
component is loaded. If maintainers want a preview feature flag, it can gate the loading of
`sandbox` components.

### Acceptance Criteria

#### Versions measured

- **Every result below was measured at upstream b0ce41dac** (patches 1–15).
- **The fix wave** (patches 16–27, up to 62e16213e) changes error paths only: release of a
  running command, the Acquire bound, error mapping, and the opt-in of the process driver. It
  does not change the measured fast path (POOL activation and exec), and it was not
  re-measured.

#### The metric and the gate

**The metric**
- **What is timed.** `timing.totalUs`, measured by the **caller's** daprd from receiving the
  exec request to sending the response.
- **What it includes:**
  - Dapr routing, including the cross-pod hop;
  - activation and the pool claim;
  - the driver handover;
  - the command in the guest.
- **The command.** A trivial one, `echo ok`, so the response follows its output directly.
- **The harness clock.** The harness records its own wall clock too, as `client` in
  `results/*/summary.md`.

**The gate.** It is for path (a), a new sandbox ID served from the warm pool, in two modes:
- requests one after another (sequential);
- a sustained 20 new sandboxes per second per host pod (rate 20).

**What passes**
- p99 under 100 ms;
- at most 1% errors;
- at most 1% late ticks in rate mode;
- at least 100 POOL rows.

**Which rows count.** Rows served by RESTORE in path (a) are left out of the gate's percentiles
and reported as `nonpool`. The benchmark is in `bench/`.

**The four paths**

| Path | What it measures | Component |
|---|---|---|
| (a) | new sandbox ID from the warm pool (the gate) | `py-fast` (pool of 8, refilled 4 at a time) |
| (b) | warm exec on an existing ID | `py-fast` |
| (c) | new ID with no pool: restore from the template snapshot | `py-restore` |
| (d) | new ID with no template: fresh boot | `py-boot` |

#### Kubernetes: the gate (bare metal)

`results/kind-linux/summary.md`:

`GATE PASS (sequential n=482 p99=16.44ms err=0.0% nonpool=3.6%; rate20 n=3600 p99=11.36ms err=0.0% nonpool=0.0% late=0.0%)`

**Host and versions**
- **Machine.** Bare metal (`systemd-detect-virt` = none): AMD Ryzen 9 5900X (12 cores,
  24 threads), 60 GiB RAM, Ubuntu 26.04 LTS, kernel 7.0.0-38-generic.
- **Cluster.** kind v0.29.0 (Kubernetes v1.33.1): 1 control plane and 2 workers.
- **Topology.** Two `sandbox-pool` pods, one per worker, and a separate `sandbox-caller` pod
  that runs the benchmark against its own daprd.
- **Software.** Dapr from the PoC branch at b0ce41dac, with mTLS on (the chart default),
  microsandbox 0.7.11 and image `python:3.12-slim`.

**Sequential**
- **Gate basis.** p99 16.44 ms over the 482 POOL rows of 500, with 0 errors.
- **Restores.** 18 of the 500 requests (3.6%) found the pool empty and were served by a restore.
- **Over all 500 rows** the p99 is 103.9 ms, and the restore rows took 92.9–133.3 ms.

**Rate 20 per host pod** (40 requests/s in total)
- p99 11.36 ms over 3600 rows: the 60 s run and the rate-20 point of the sweep, merged by the
  report;
- 0 errors, 100% POOL, 0 late ticks.

**Where the time goes**

Median breakdown of path (a) at rate 20 (`results/kind-linux/summary.md`), in ms:

| total | daprd (total − host) | activate | exec | other on the host |
|---|---|---|---|---|
| 7.18 | 0.54 | 0.32 | 6.29 | 0.03 |

- **Dapr's share.** The caller's routing, the cross-pod hop and the owner's actor activation add
  up to less than 1 ms of the 7.18 ms.
- **Exec.** It covers the driver's handover and the command in the guest; the timing does not
  separate them.

**Other paths and workloads** (sequential, p50 / p99 total):
- **path (b), warm exec:** 1.64 / 2.37 ms;
- **path (c), restore:** 81.98 / 92.16 ms (activation 75.84 ms p50);
- **path (d), cold boot:** 227.84 / 244.74 ms;
- **`python3 -c 'print(1)'` on a new ID:** 35.86 / 39.52 ms;
- **an import turn in a session on a new ID:** 66.25 / 72.27 ms. The turn runs
  `import json, decimal, email.parser; print(1)`, timed after the interpreter has started.

#### Local development: macOS

`results/mac-favordynmods/summary.md`:

`GATE PASS (sequential n=421 p99=17.79ms err=0.0% nonpool=15.8%; rate20 n=1795 p99=16.25ms err=0.0% nonpool=0.3% late=0.0%)`

**Setup**
- **Machine.** Mac16,7, Apple M4 Pro, 14 CPUs, 48 GiB RAM, macOS 27.0.1, Hypervisor.framework.
- **Deployment.** Self-hosted, one daprd that both hosts and calls.
- **Host state.** The Mac was swapping and loaded during the runs: load average about 10.

**Results**
- **Rate 20** (the single 60 s run, 1200 rows): p50 3.0 ms, p99 9.4 ms.
- **Sequential.** The run drained the pool: 79 of 500 rows were restores, so the p99 over all
  rows is 192.0 ms.

#### All four result sets

Total and exec time, p50 / p99, in ms (from `results/kind-linux/notes.md`).

| run | Mac total | Mac exec | Mac + favordynmods total | Mac + favordynmods exec | kind nested total | kind nested exec | kind bare metal total | kind bare metal exec |
|---|---|---|---|---|---|---|---|---|
| path a seq | 34.1 / 41.2 | 33.8 / 41.0 | 6.2 / 192.0 | 4.2 / 17.5 | 244.1 / 886.9 | 242.7 / 883.7 | 8.3 / 103.9 | 7.2 / 12.4 |
| path a rate20 | 35.2 / 42.9 | 34.9 / 42.1 | 3.0 / 9.4 | 2.8 / 9.2 | 20113.8 / 29706.0 | 9164.1 / 27363.0 | 7.2 / 11.4 | 6.3 / 10.6 |
| path b seq | 0.8 / 22.0 | 0.8 / 22.0 | 1.2 / 2.0 | 1.2 / 2.0 | 4.2 / 24.1 | 3.6 / 23.5 | 1.6 / 2.4 | 1.4 / 2.1 |
| path c seq | 93.1 / 111.7 | 29.3 / 38.2 | 97.4 / 127.6 | 2.8 / 4.2 | 290.1 / 538.9 | 149.9 / 323.7 | 82.0 / 92.2 | 5.6 / 6.6 |
| path d seq | 187.6 / 209.7 | 34.0 / 37.8 | 216.4 / 264.8 | 1.8 / 4.2 | 2533.3 / 3205.9 | 51.5 / 113.3 | 227.8 / 244.7 | 1.8 / 2.3 |

**The four sets**

| Set | Folder | Where it ran |
|---|---|---|
| Mac | `results/mac` | self-hosted, before the favordynmods fix |
| Mac + favordynmods | `results/mac-favordynmods` | self-hosted, with the fix |
| kind nested | `results/kind` | Lima VM on the same Mac |
| kind bare metal | `results/kind-linux` | the gate run |

**How to read the table**
- **Sequential rows** are over all rows, restores included. The gate uses POOL rows only:
  p99 16.44 ms on bare metal and 17.79 ms on the Mac with the fix.
- **"path a rate20"** is the single 60 s run:
  - Mac: one host, 20 requests/s in total;
  - kind: two host pods, 40 requests/s in total.
- **Nested kind** ran paths (c) and (d) with 100 requests each instead of 500. Its
  "path a rate20" cells come from the 21 successful rows of 2400: 99.1% of that run's requests
  failed.

**Caveat: kind on macOS under nested virtualization (`results/kind`) failed the gate.**

`GATE FAIL (sequential n=500 p99=886.88ms ...)`
- **The cause is the nested stack, not Dapr.** The microVM runs under KVM in a Lima VM, which
  itself runs under Apple's Virtualization framework, so every guest exit traps through two
  hypervisors.
- **Evidence:**
  - Activation was 0.4 ms p50 there too, but the first exec in a freshly restored VM took
    242.7 ms p50.
  - At rate 20 per pod, 99.1% of requests failed: 429 from the host's admission limit
    ("host is full: 32 sandboxes") while restores lagged, restore timeouts, and client
    timeouts.
  - The same deployment on bare metal passes.
- **The favordynmods fix on the nested cluster.** It cut path (a) from p50 780.9 ms to
  218.2–222.1 ms in live previews (`results/kind/notes.md`; not gate results). The rest is the
  first exec after a restore under nested KVM.

**Result files.** Each set's report, as the benchmark wrote it, is in [`resources/20261029-BRCS-sandbox-building-block/`](resources/20261029-BRCS-sandbox-building-block/): [bare metal](resources/20261029-BRCS-sandbox-building-block/kind-linux-summary.md), [Mac + favordynmods](resources/20261029-BRCS-sandbox-building-block/mac-favordynmods-summary.md), [nested kind](resources/20261029-BRCS-sandbox-building-block/kind-nested-summary.md), [Mac](resources/20261029-BRCS-sandbox-building-block/mac-summary.md). Raw CSVs and notes are in the [PoC repository](https://github.com/XavierGeerinck/poc-dapr-sandbox/tree/main/results).

Rate sweep on bare-metal kind (path (a), per host pod, 2 pods):

![Rate sweep, kind on bare metal](resources/20261029-BRCS-sandbox-building-block/kind-linux-rate-sweep.svg)

#### What the measurements showed

**1. A guest-kernel exec delay, and the fix** (`results/idle-cliff/notes.md`)

*The cause*
- In microsandbox 0.7.11, the guest agent (agentd) moves every exec child into a cgroup by
  writing `cgroup.procs`.
- In the guest kernel, libkrunfw Linux 6.12.113 (aarch64, `CONFIG_HZ=100`), that write waits for
  one RCU grace period whenever the VM has been idle for longer than about one grace period.
- So any exec after more than about 20–25 ms of idle paid about 28 ms. A pooled VM always idles
  that long before its handout.

*The fix*
- Remount cgroup2 with `favordynmods` in the template's `setup`; restored VMs inherit it.
- Measured on the driver's backend without Dapr, on the Mac (Apple M4 Pro, Hypervisor.framework),
  exec p50 after ≥ 20 ms of idle went from 28.1–31.6 ms to 2.2–3.5 ms.
- Under KVM, the nested-kind gap table (`results/kind/notes.md`, live previews) shows the same
  plateau and the same fix.
- In Dapr on the Mac (`results/mac` against `results/mac-favordynmods`):
  - path (a) exec p50 went from 33.8 ms to 4.2 ms;
  - the rate-20 p99 went from 42.9 ms to 9.4 ms.
- Every component in the PoC carries this `setup`.

*Upstream.* The proper fix is for agentd to mount cgroup2 with `favordynmods`. A draft issue
for microsandbox is in [`proposal/microsandbox-issue.md`](https://github.com/XavierGeerinck/poc-dapr-sandbox/blob/main/proposal/microsandbox-issue.md).

**2. Side effect: a non-empty `setup` gives the template a network device**
- **Why.** The template VM keeps its network while `setup` runs, and a full restore cannot drop
  a captured device. Restored VMs carry an `eth0`, though their network policy refuses every
  connection.
- **Path (c), Mac baseline to Mac with the fix:**
  - restore activation p50 went from 63.6 ms to 94.5 ms;
  - total went from 93.1 / 111.7 ms to 97.4 / 127.6 ms (p50 / p99).
- **This is a reading, not an isolated measurement.** Each is one run on a loaded Mac.
- **Path (d)** runs `setup` in every booted VM, and its activation p50 rose from 154.4 ms to
  214.5 ms.
- **A possible fix.** The driver, or the upstream agentd fix, could apply the remount without a
  network device.

**3. Warm-up: `warmupCommand`, opt-in and off by default**
- **What it does.** It runs a no-op in each pooled VM before handout.
- **Its premise was wrong on the Mac.** The cost was not a first exec but the idle delay in
  finding 1. With the warm-up, the first exec after handout was 29.6–31.0 ms; without it,
  32.0–32.7 ms (p50, 3 runs). It cost about 30–33 ms per refill (`results/mac/notes.md`).
- **On nested kind** (live, not gate results; `results/kind/notes.md`):
  - **Sparse traffic, where it helps.** With 10 IDs spaced 5 s apart, p50 went from 199.9 ms
    to 72.5 ms.
  - **Back-to-back traffic, where it hurts.** p50 went from 237.6 ms to 489.7 ms and p90 from
    371.4 ms to 1171.5 ms.
  - **Why it hurts.** Each refill's warm-up took 0.3–3.8 s and competed with live requests.
- **Bare metal.** It was not measured there.
- **The committed configuration leaves it off.**

**4. The next bottleneck is warm-pool refill, not Dapr** (`results/kind-linux/notes.md`)

*Rate sweep on bare metal (per host pod; pool of 8, 4 refills in parallel, per pod)*

| Rate per pod | What happened |
|---|---|
| 10 and 20/s | every request was POOL; p99 7.90 and 11.36 ms |
| 40/s | 19.7% fell back to RESTORE; p99 208.6 ms |
| 80 and 160/s | the pool was exhausted and restores lagged, so the hosts reached their admission limit (`maxSandboxesPerHost`); 48.6% and 75.5% of requests got 429 |

*Sequential back-to-back requests on bare metal* hit RESTORE 18 times in 500.
- A sequential client takes about 8 ms per request across two pods.
- A restore takes about 76 ms of activation, 4 at a time per pod.
- So the client drains warm VMs faster than the pool restores them. This is the run's reading,
  not a separate measurement.

*Future work:* capacity-aware placement and refill tuning (pool size, refill concurrency).

## Completion Checklist

**Runtime (`dapr/dapr`)**
- [x] Component category `sandbox` and the pluggable contract `dapr/proto/components/v1/sandbox.proto` (PoC).
- [x] Internal actor target `pkg/actors/targets/sandbox`, and `pkg/runtime/sandboxhost` for readiness, registration and reconcile (PoC).
- [x] Alpha runtime API: HTTP routes and gRPC methods, error codes (PoC).
- [x] Built-in `sandbox.process` reference driver (PoC).
- [x] Cross-app access to hosted sandbox actor types under mTLS (PoC, patches 14, 15 and 27; needs security review).
- [x] Unit tests (`pkg/actors/targets/sandbox`, `pkg/api/{grpc,http,universal}`, `pkg/components/sandbox/...`, `pkg/runtime/sandboxhost`, `pkg/runtime/processor/sandbox`) and integration tests (`tests/integration/suite/daprd/sandbox`, including `crossappmtls` with Sentry) (PoC).
- [ ] The smaller gaps from the PoC's reviews, listed in [`docs/poc-known-gaps.md`](https://github.com/XavierGeerinck/poc-dapr-sandbox/blob/main/docs/poc-known-gaps.md), including sign-offs and a history cleanup before a dapr/dapr PR.
- [ ] Per-sandbox owner binding, or access control for sandbox actor calls.
- [ ] Report an oversized response as an output error, not 429.
- [ ] Optional preview feature flag.
- [ ] Metrics: pool depth, acquire path, activation and exec latency.
- [ ] End-to-end tests on Kubernetes in the dapr/dapr e2e suite.
- [ ] Phase 2: the flexible profile (per-sandbox spec, save and restore to the binary store).

**Components and drivers**
- [x] A microVM driver as a pluggable component (Rust, microsandbox 0.7.11, KVM and Hypervisor.framework): proof of concept, outside `dapr/dapr`.
- [ ] Decide where drivers live: `components-contrib` or separate repositories.
- [ ] A Cloud Hypervisor driver (`/dev/mshv`, AKS Pod Sandboxing).
- [ ] An unprivileged KVM device plugin route.

**SDKs**
- [ ] Alpha client methods in the Go, Python, .NET, Java and JavaScript SDKs.

**Documentation (`dapr/docs`)**
- [ ] Building block overview and API reference.
- [ ] Component reference for `sandbox` components.
- [ ] Operations: `--max-body-size 16Mi`, the privileged driver container, `singleProcessOOMKill`, and the sidecar start order.
- [ ] The limitations above: lost sandboxes, full hosts, the template's scope, and the security model.
