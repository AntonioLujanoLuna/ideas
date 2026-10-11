# Isolated and Distributed Agent Runtime Architecture

**Status:** Design proposal  
**Updated:** 2026-10-11  
**Scope:** FastAPI portal, LangGraph and GitHub Copilot SDK agents, local execution, remote containers, microVMs, and dedicated agent compute hosts.

## 1. Summary

The portal currently executes LangGraph and Copilot SDK agents in its backend runtime. Introduce an **Agent Runtime Manager** to decouple an agent's implementation from its execution location. The same logical agent can then run:

1. **In-process**, as it does today, for trusted and lightweight tasks.
2. **In a persistent remote worker container**, for isolated or resource-intensive agents.
3. **In a session-scoped container or microVM**, for coding agents and other workloads that run untrusted code.
4. **In-process with isolated tool execution**, for agents that only occasionally need to run Python, shell commands, or manipulate a workspace.

The preferred first implementation is a **dedicated CPU/RAM agent host using rootless Podman**, with a small authenticated worker-control service, while FastAPI and the existing vLLM gateway remain where they are. Kubernetes/K3s and Kata Containers can be added when the workload justifies them.

**Core rule:** Share agent logic and an execution contract, not process memory, across local and remote implementations. All isolation and placement decisions belong to the backend, never to the user prompt.

This proposal extends [portal_agent_tool_execution_architecture.md](portal_agent_tool_execution_architecture.md) and is complementary to [openclaw_copilot_agents_automation_architecture.md](openclaw_copilot_agents_automation_architecture.md).

## 2. Existing environment and desired outcome

Assumed environment:

- A multi-user portal with a FastAPI backend running LangGraph and GitHub Copilot SDK agents.
- A central model gateway routing requests to vLLM on separate GPU nodes (H100 and H200). Inference is **not** moved to agent execution workers.
- Existing native and MCP-backed tools, plus client-side tools for local filesystem, spreadsheet, and code operations.
- Corporate network controls, reverse proxies (potentially only port `9080` exposed between certain machines), and user-scoped Jira and other service permissions.
- State and observability infrastructure that should be reused rather than embedded in ephemeral workers.

Desired properties:

- One agent implementation works with local and isolated execution.
- Different agents and sessions can use different isolation levels.
- Agent compute nodes can be on entirely separate machines.
- Conversation state survives worker restarts or migration when the framework supports it.
- Streaming, cancellation, tool calls, audit events, and user identity continue to work.
- No sandbox gets unrestricted access to the FastAPI host, model host, host filesystem, or corporate network.
- Operators can begin without Kubernetes and expand later.

Non-goals: rewriting existing agents in another framework; placing the model itself inside an agent pod; granting agents direct control of infrastructure; enforcing identical persistence semantics across fundamentally different agent SDKs.

## 3. Logical architecture

~~~mermaid
flowchart TB
    UI[Portal / IDE clients] <-->|HTTPS, SSE, WebSocket| API[FastAPI backend]
    API --> AUTH[Identity, policies, session registry]
    API --> RM[Agent Runtime Manager]
    RM --> LOCAL[Local agent adapter]
    RM --> CTRL[Worker control / scheduler]
    CTRL --> HOST[Dedicated agent compute host]
    HOST --> POOL[Persistent agent workers]
    HOST --> EPHEMERAL[Per-session containers]
    HOST --> VM[MicroVM / hardened sandbox workers]
    LOCAL --> FRAMEWORKS[LangGraph / Copilot SDK]
    POOL --> FRAMEWORKS2[LangGraph / Copilot SDK]
    EPHEMERAL --> FRAMEWORKS3[LangGraph / Copilot SDK]
    VM --> TOOLS[Isolated code and shell tools]
    FRAMEWORKS --> GW[LLM gateway]
    FRAMEWORKS2 --> GW
    FRAMEWORKS3 --> GW
    GW --> GPU[vLLM GPU nodes]
    API <--> STATE[(External sessions, checkpoint store, artifacts)]
    POOL <--> STATE
    EPHEMERAL <--> STATE
    API <--> ROUTER[Central tool registry / router]
    POOL <--> ROUTER
    EPHEMERAL <--> ROUTER
    ROUTER --> MCP[Internal MCP / RAG / Jira services]
    ROUTER <-->|Authenticated relay| CLIENT[Client-side tool executor]
~~~

This is a logical diagram, not a recommendation to allow every worker to contact every service directly. Network and identity policies restrict each connection.

**Control plane:** FastAPI, agent catalog, placement policy, worker controller, session registry, provisioning, quotas, reconciliation and termination.

**Data plane:** Worker execution, model requests, tools, output events, checkpoint writes and artifacts. Model traffic may go directly from workers to the existing gateway, but **never** directly to the underlying GPU node unless explicitly allowed by infrastructure policy.

### Physical placement

A valid deployment uses three distinct categories of machines:

| Machine | Workloads | Hardware |
| --- | --- | --- |
| Portal/backend host | FastAPI, authenticated APIs, session registry and runtime manager | Ordinary CPU/RAM |
| Dedicated agent compute host | Container workers and optionally microVMs, workspaces, execution controller | CPU/RAM/storage; no GPU required by default |
| Existing model servers | vLLM and LLM inference | H100/H200 GPUs |

The worker host may be elsewhere in the data center or in another reachable site. Requirements are authenticated bidirectional or appropriately proxied connections for request/response/streaming, stable service discovery, acceptable latency, and access to approved external state. Shared filesystem mounts are **not** required if artifacts and checkpoints are externally stored.

Do not make the worker host share the FastAPI process, the gateway host, or the GPUs. A dedicated host is explicitly supported by this design.

## 4. Choose the isolation boundary deliberately

| Execution mode | Isolation boundary | Appropriate cases | Trade-offs |
| --- | --- | --- | --- |
| `local` | Existing FastAPI process | Trusted read-only agents | Lowest overhead; least isolation |
| `remote-shared` | Persistent worker container/process | Most corporate agents with controlled tools | Efficient, but requires strong per-session logical isolation |
| `remote-session` | Separate container per user/session | Coding agents, dedicated workspaces | Higher RAM/startup cost, easier file isolation |
| `microvm-session` | VM per session or execution | Untrusted shell/code/dependencies, higher-risk tenants | Stronger boundary; VM orchestration/KVM needed |
| `local-agent-remote-tool` | Agent stays local, dangerous tool executes remotely | Search/Jira agent that occasionally runs Python | Lightweight; tool routing becomes central |

**Important:** Containers primarily isolate with kernel namespaces, cgroups, filesystem and capability controls. They are not equivalent to hardware-enforced isolation. For adversarial or truly untrusted code, prefer hardened sandboxing with gVisor or microVMs based on the environment and threat model.

For most portal agents, a **hybrid** is preferable. Do not start a new microVM for each conversational turn. Distinguish a trusted agent process from an isolated code-execution tool.

## 5. Software abstraction: agent code versus runtime transport

Keep framework-specific logic in adapters and place execution transport behind a common interface:

~~~text
portal/
  api/
    chat.py
    runs.py
  agents/
    registry.py
    langgraph/
    copilot/
  runtimes/
    contracts.py
    manager.py
    local.py
    remote.py
    placement.py
  worker/
    api.py
    runner.py
    lifecycle.py
  tools/
    registry.py
    router.py
  persistence/
    sessions.py
    events.py
    artifacts.py
~~~

The backend chooses the runtime from a **server-owned policy**, for example by agent ID, approved capabilities, workspace needs, user/tenant quota, data classification and worker availability.

### 5.1 Stable contracts

Represent each execution request as a transport-independent envelope:

~~~json
{
  "run_id": "run-73f2",
  "session_id": "session-224",
  "agent_id": "code-reviewer",
  "framework": "copilot",
  "message": "Review this change and run the tests",
  "workspace_ref": "artifact://workspaces/session-224",
  "policy_ref": "coding-isolated-v1",
  "trace_id": "trace-60bc",
  "idempotency_key": "turn-993"
}
~~~

The example identifiers are illustrative. **Do not accept `policy_ref`, `agent_id`, tenant or workspace access as authoritative simply because the client supplied them.** FastAPI must bind the request to its authenticated session and resolve the effective policy. The worker should validate a short-lived, scoped capability or signed job claim issued by the backend.

The common application-facing interface is conceptually:

~~~python
from collections.abc import AsyncIterator
from typing import Literal, Protocol
from pydantic import BaseModel


class RunRequest(BaseModel):
    """Authenticated, normalized execution request."""
    run_id: str
    session_id: str
    agent_id: str
    framework: Literal["langgraph", "copilot"]
    message: str


class RunEvent(BaseModel):
    """Framework-independent event delivered to the portal."""
    run_id: str
    sequence: int
    type: Literal[
        "started", "token", "tool_started", "tool_result",
        "approval_required", "checkpoint", "completed",
        "cancelled", "failed"
    ]
    payload: dict[str, object]


class AgentRuntime(Protocol):
    """Transport contract for local and remote agent execution."""

    async def start(self, request: RunRequest) -> str:
        """Start an execution and return a durable run identifier."""
        ...

    def events(self, run_id: str, after: int = 0) -> AsyncIterator[RunEvent]:
        """Read ordered execution events after the given sequence."""
        ...

    async def cancel(self, run_id: str) -> None:
        """Request cancellation of an active execution."""
        ...
~~~

The protocol above is an interface specification, not deployable code. Local and remote implementations must provide complete implementations of these methods. Keep the agent logic independent of transport; do not embed Podman or Kubernetes calls inside LangGraph graph nodes or Copilot tool handlers.

### 5.2 Runtime Manager

Responsibilities:

1. Authenticate and authorize the user, agent, session and tools.
2. Resolve runtime policy, resource requirements and workspace capability.
3. Reuse or provision an appropriate worker.
4. Persist `run_id → worker_id` and `session_id → runtime/state locator`.
5. Dispatch the run and relay streamed events.
6. Handle client disconnect, cancellation, timeouts and recovery.
7. Clean up isolated workspaces and terminate idle workers.
8. Enforce quotas and audit all placement and tool decisions.

The agent-facing code should depend on a common `AgentRuntime` contract. A deployment configuration chooses `LocalRuntime` or `RemoteRuntime`, while the manager may choose between them per agent. This also allows a migration without changing client endpoints.

An example policy, not an end-user-controlled flag:

~~~yaml
agents:
  jira-summary:
    framework: langgraph
    runtime: local
    allowed_tools: [jira_read, search]
  repository-maintainer:
    framework: copilot
    runtime: remote-session
    sandbox: container
    allowed_tools: [git, tests, approved_mcp]
    cpu_limit: 2
    memory_limit: 4Gi
    idle_ttl_seconds: 1200
  python-analysis:
    framework: langgraph
    runtime: local-agent-remote-tool
    allowed_tools: [rag_search, sandbox_python]
~~~

These resource allocations are **initial policy examples, not capacity estimates**. Load-test real tasks before selecting quotas.

## 6. Framework-specific considerations

### LangGraph

- Reuse existing graph construction, tools and prompts inside a worker-facing adapter.
- A normal graph can run locally or in a remote Python process without being rewritten.
- Move any backend-only imports, singleton objects or global in-memory session state into explicit dependencies.
- Preserve the existing checkpointer if appropriate; a central Postgres-backed checkpointer is an alternative, not a requirement to migrate away from an existing supported store.
- Use stable thread IDs, versioned graph definitions, external checkpoint storage and concurrency controls.
- A checkpoint is not a universal process snapshot: ongoing Python tool calls and uncheckpointed side effects may not resume safely after worker failure.
- Keep tool invocations idempotent or implement deduplication for external side effects such as creating Jira tickets.

### GitHub Copilot SDK

There are **two materially different remote deployments**.

**A. Remote Copilot CLI/runtime only:** The SDK client remains on the backend, connected to a separate headless Copilot server over a runtime connection. Useful when wanting a separate CLI process and modest architectural change. It does **not** automatically isolate application-side SDK callback handlers, local filesystem access, or tools that still execute in the backend.

For currently documented Python SDK versions, the pattern is:

~~~python
from copilot import CopilotClient, RuntimeConnection

client = CopilotClient(
    mode="empty",
    connection=RuntimeConnection.for_uri("agent-worker.internal:4321"),
)
await client.start()
~~~

A headless runtime can be launched with `copilot --headless --host 0.0.0.0 --port 4321`, but only on a restricted private network with authentication/proxy controls. Verify the exact SDK/CLI version and connection authentication in the deployment.

**B. Entire Copilot agent in a worker:** Move the Copilot SDK client, CLI/runtime, workspace and relevant SDK-side tool handlers into the isolated container. This is the **preferred mode for true agent isolation**, particularly coding agents. The backend communicates only with the worker service, not directly with the Copilot runtime port.

For multi-user services, use `mode="empty"` as the SDK's restrictive baseline and explicitly allow tools, MCPs and workspace paths. Do not rely solely on this SDK setting for OS or network isolation. Use per-user/session credentials where possible. Avoid `approve_all` permission handlers for sensitive tools. Configure the model provider to reach the existing enterprise LLM gateway, and test that endpoint's API compatibility.

Session persistence for the SDK/runtime must be configured deliberately. When using an external CLI runtime, local SDK `base_directory` settings may not govern the remote server's session files. Use runtime-owned persistent storage/session filesystem capabilities as supported by the deployed version, or pin a session to its living worker and explicitly accept that recovery may be limited.

**Separate runtime connection is a transport feature; separate execution host is the security boundary.**

Documentation:
- [Copilot SDK Python README](https://github.com/github/copilot-sdk/blob/main/python/README.md)
- [Copilot SDK backend services](https://github.com/github/copilot-sdk/blob/main/docs/setup/backend-services.md)
- [Copilot SDK multi-tenancy](https://github.com/github/copilot-sdk/blob/main/docs/setup/multi-tenancy.md)

## 7. Worker service and lifecycle

Recommended worker API (private, authenticated, versioned):

| Endpoint | Behavior |
| --- | --- |
| `POST /v1/runs` | Accept normalized run and return `run_id`, placement and state |
| `GET /v1/runs/{id}/events?after=N` | Stream ordered events using SSE, with replay/cursor support |
| `POST /v1/runs/{id}/cancel` | Request cooperative cancellation, then enforce timeout |
| `GET /v1/runs/{id}` | Return authoritative run status |
| `POST /v1/sessions/{id}/terminate` | End a session and dispose of eligible resources |
| `GET /health/live`, `GET /health/ready` | Host/process health and readiness |

For a single host, the controller can provision workers and register their ephemeral network addresses. The backend should not expose or accept arbitrary worker URLs from users. Workers must only accept runs whose signed claims match their identity and assigned scope.

### Workload patterns

- **Persistent worker pool:** A few prestarted LangGraph/Copilot processes. Lowest cold-start latency, but tightly restrict cross-user data, tool callback state and filesystem access.
- **Warm container pool:** Precreated worker environments with no assigned user. Claim a fresh workspace when a session starts; sanitize or destroy after use.
- **Per-session container:** One private process and workspace for a user's coding session; stop after configured idle TTL.
- **Per-execution sandbox:** Spawn for code tools, return outputs/artifacts, destroy immediately.
- **MicroVM pool:** Use only where stronger isolation is justified and available.

Prefer session affinity while a stateful worker lives, but do not confuse affinity with durable persistence. On host failure, reconcile orphaned runs and recover only at supported checkpoints.

### Run state model

~~~text
QUEUED -> STARTING -> RUNNING -> COMPLETED
                     |       |
                     |       +-> CANCELLING -> CANCELLED
                     +-----------------------> FAILED
RUNNING -> TIMED_OUT
RUNNING -> LOST (lease expired, recovery required)
~~~

Implement leases/heartbeats, run deadlines and a reconciliation loop. Record transitions externally so a restarted backend does not lose track of sessions.

## 8. Streaming, tools and human approvals

FastAPI remains the client-facing stream endpoint. Internally, workers emit normalized events with a monotonically increasing per-run sequence number; the backend stores/replays events and translates them into its existing SSE/WebSocket format.

- Cancellation must reach the agent framework **and** any executing tool process. Force-terminate a sandbox after a grace period.
- Client disconnect does not automatically imply cancellation; make this an explicit portal policy.
- The worker-to-backend stream may disconnect, so retain a bounded durable event buffer or event store, replaying by sequence.
- Model calls may have emitted tokens before an error; distinguish a partial answer from successful completion.
- Approval events must be resolved by the authenticated backend, not by blindly accepting worker-supplied user identities.

### Keep existing server and client tools

The existing centralized tool architecture should remain intact:

1. **Server-side tools:** RAG, Jira, centralized web search and MCP services execute on approved service hosts.
2. **Client-side tools:** User-approved Excel and filesystem operations execute on the client via the portal's existing authenticated client-tool relay.
3. **Isolated tools:** Python, shell, tests, package installation and repository manipulation can run in a container/microVM on the agent host.

A remotely hosted agent calling a **client-side tool** should send an authenticated tool request back through the backend, which authorizes the requested capability and relays it to the intended client connection. The client result travels back through the same route. Do not assume remote containers can access a user's laptop directly.

In a remote Copilot SDK deployment, check **where each SDK callback actually executes**. Moving the CLI alone does not move SDK-registered Python/TypeScript callback code; moving the entire SDK application into a worker does.

Related design: [portal_agent_tool_execution_architecture.md](portal_agent_tool_execution_architecture.md).

## 9. Workspace, artifacts and persistence

Persist independently of disposable workers:

| Data | Recommended ownership |
| --- | --- |
| Identity, session ownership, quotas, worker placement | Backend database |
| LangGraph checkpoint/thread state | Existing supported shared checkpointer |
| Copilot runtime/session storage | Runtime-owned durable store or explicit session affinity |
| Streaming events and audit history | Append-only/event storage with retention policy |
| Workspaces and produced files | Scoped artifact storage, or encrypted persisted volume if necessary |
| Trace context | Existing Langfuse/telemetry pipeline |
| Secrets | Secret manager / scoped credential broker |

Workspaces require more than a shared directory name. Each run obtains an authorized workspace capability; prevent traversal, symlink escapes, cross-tenant reuse, unapproved mounts and artifact access after session revocation.

For code agents, choose between:

- **Clone-on-start** from an authorized Git repository and upload only intended artifacts/results.
- **Snapshot/restore** using a scoped workspace volume or object store.
- **Persistent session volume** for long editing sessions where filesystem continuity matters.

Never mount the backend source tree or infrastructure credentials into a general-purpose untrusted worker. Keep scratch directories and artifact stores subject to disk quota and retention policies.

## 10. Networking and the existing `9080` constraint

The dedicated host needs reachable network paths to the backend/controller and approved dependencies, not direct Internet access by default.

~~~text
Portal users
   -> FastAPI
   -> authenticated worker API at agent-host:9080/agent-runtime/
   -> internal worker / session process
   -> approved enterprise LLM gateway
   -> existing vLLM infrastructure

Worker -> policy-controlled tool services / state store
Worker -> corporate proxy only if the agent actually needs web access
~~~

If corporate infrastructure exposes only `9080`, use Nginx on the worker host to terminate/forward authenticated requests, including SSE and WebSocket upgrades where relevant. Worker processes can listen on loopback or private container networks. The Nginx path rewrite and streaming timeouts must be tested explicitly; do not reuse a vLLM proxy config unchanged.

Prefer **TLS with mTLS between services**, or a private network with strong service authentication and scoped signed workload tokens. A static bearer token can suffice for a local prototype but should not be the production authorization model.

Avoid exposing the Podman socket, Kubernetes credentials, Copilot headless CLI TCP port, or sandbox container port to portal users. The backend and its worker controller should never be able to request arbitrary images or arbitrary host mounts merely because a prompt asks for them.

If hosts are in separate buildings/data centers, validate firewall routes, proxy behavior, latency, DNS, clock synchronization, certificate trust, and recovery under temporary network partition.

## 11. Security model

A prompt, fetched document, repository README, tool output or agent-generated shell command is **untrusted input**. Isolation prevents execution from inheriting the backend's authority, but it does not replace authorization.

Minimum controls:

- Image allowlist and immutable digests; scan/sign images where possible.
- Run as non-root; read-only root filesystem, writable scoped workspace, dropped Linux capabilities, seccomp, cgroup CPU/RAM/PID limits, no privileged mode.
- Never mount Podman/Docker sockets, host root directories, SSH agent sockets or broad credentials.
- Per-run network egress policy and DNS controls. Deny unrestricted access to internal management subnets, metadata services and unrelated worker sessions.
- End-to-end identity binding: authenticated caller -> allowed agent -> scoped tools -> downstream per-user or delegated credential.
- Short-lived credentials with explicit expiration and revocation; redact model prompts, tool outputs and traces as needed.
- Human approvals for privileged side effects, e.g. Jira writes or code pushes, with audit logging.
- Time, disk, output size, process and recursive-agent limits. Prevent fork bombs, runaway package installs, resource exhaustion and event-stream abuse.
- Namespace artifact IDs, log streams and checkpoint keys by authorized tenant/session.
- Threat-model prompt injection, malicious dependencies, workspace tampering, data exfiltration and cross-user session leakage.

**Container isolation alone may be insufficient for untrusted generated code.** Where stricter isolation is required, evaluate:

| Technology | Role | Caveat |
| --- | --- | --- |
| Rootless Podman + OCI runtime | Simple container workers on a dedicated host | Shared kernel; isolate network, mounts and host API carefully |
| gVisor | Stronger container syscall mediation | Compatibility and performance testing required |
| Kata Containers | Kubernetes/containerd workload executed in lightweight VMs | Requires compatible runtime and KVM/virtualization support |
| Firecracker | KVM microVMs with explicit lifecycle/network/filesystem management | More platform engineering if used directly |

Kata/Firecracker generally depend on host virtualization support, `/dev/kvm` and infrastructure administrator assistance. Being able to run rootless Podman does **not** imply permission to run hardware-isolated microVMs. Check nested virtualization if the agent compute host is itself a VM.

References: [Kata installation prerequisites](https://github.com/kata-containers/kata-containers/blob/main/docs/installation.md), [Kata with containerd](https://github.com/kata-containers/kata-containers/blob/main/docs/how-to/containerd-kata.md), [Firecracker](https://github.com/firecracker-microvm/firecracker).

## 12. Orchestration decision: Podman first, Kubernetes later

### Stage A: Dedicated Podman worker host

Use a small worker controller responsible for a tightly scoped subset of operations:

- Authenticate control requests from FastAPI.
- Choose an image from a fixed registry, assign session ID and constrained resources.
- Create or reuse rootless Podman containers.
- Track worker health, leases, logs and termination.
- Publish reachable worker addresses only to the backend.
- Enforce idle shutdown and clean up stale containers/workspaces.

The worker controller should run under a dedicated OS account and must not expose arbitrary Podman API access. The FastAPI application should not be given a remote Podman root socket.

A single dedicated server is sufficient to begin. Its CPU/RAM sizing is a function of observed *simultaneously executing tools*, number of persistent agent processes, workspace sizes and idle session policy, not the number of vLLM tokens.

### Stage B: Multiple compute hosts

Add:

- Worker registry/discovery and health-based placement.
- Per-host capacity admission control.
- Backpressure and fair scheduling per tenant.
- Retry/reconciliation for transient host failures.
- Artifact store and session recovery independent of host.
- Central telemetry and capacity dashboards.

At this stage a cluster orchestrator becomes attractive, but it is still not obligatory.

### Stage C: K3s/Kubernetes and sandbox runtimes

Use Kubernetes when deployment, scheduling, health management, quotas and node-level capacity become painful to maintain manually. Place agent pods on a dedicated node pool via labels/affinity and tolerations; use resource limits and network policies. Add `RuntimeClass` for Kata if stronger isolation is needed.

MicroVM sandboxes should be introduced only after validating host compatibility, startup time, network rules, workspace restoration and the threat model. Kubernetes is not a prerequisite for remote container execution.

## 13. Reliability and performance

Separate **agent execution concurrency** from **LLM serving concurrency**. Moving agents to another machine removes their CPU/tool load from FastAPI but does not itself increase the available KV cache, throughput or context capacity on H100/H200.

Watch:

- Active sessions, running jobs, per-host CPU/RAM/disk and sandbox startup latency.
- Queue duration, execution duration, worker utilization and cold-start rate.
- SSE first-event latency and end-to-end token latency.
- Model gateway request count, TTFT and token throughput under agent concurrency.
- Long-running tool memory consumption and per-user quotas.
- Session recovery rate, failed side effects and orphaned workers.
- Event-buffer retention and streaming reconnect success.

Failure policy:

1. Return an explicit error when no permitted worker has capacity; never silently fall back to weaker isolation.
2. Retry **placement/startup failures** when safe, with stable idempotency keys.
3. Do not automatically rerun a partially completed agent turn with external side effects unless the framework/tool is designed for at-least-once execution.
4. Use checkpoint-aware resume for LangGraph where supported.
5. Treat lost Copilot runtime sessions according to actual deployed persistence behavior, not an assumed automatic migration guarantee.
6. Cancel and clean up idle/exceeded-budget runs; verify termination actually happened.

## 14. Implementation plan

### Phase 1: Transport abstraction

- [ ] Inventory all agents, their framework, tools and external state.
- [ ] Introduce `AgentRuntime` and local/remote adapters without changing portal endpoints.
- [ ] Define normalized run/event models and backend-owned placement policy.
- [ ] Add durable run/session metadata and trace IDs.
- [ ] Implement an authenticated worker API with SSE, cancellation, status and health.
- [ ] Run a simple LangGraph agent on the dedicated host and verify gateway/tool access.

**Acceptance:** A single agent runs locally or remotely by server configuration; its conversation, output and audit records appear identically to the portal.

### Phase 2: Copilot and durable sessions

- [ ] Confirm installed SDK and CLI versions and their model-provider configuration.
- [ ] Test remote-only Copilot CLI transport separately from whole-agent worker relocation.
- [ ] Package the Copilot SDK + CLI + allowed tool handlers into a worker image.
- [ ] Enable restrictive `mode="empty"`, scope tools and credentials per session.
- [ ] Validate resume, crash recovery, callback location and tool permission paths.
- [ ] Test portal client-side tool relay from a remote Copilot worker.

**Acceptance:** A Copilot coding session runs on the worker machine without accessing the FastAPI host filesystem or host credentials.

### Phase 3: Isolated workspaces and tools

- [ ] Add dedicated container per coding session with workspace and time/resource quotas.
- [ ] Add per-execution sandbox for generated Python/shell from otherwise local agents.
- [ ] Enforce egress restrictions, image allowlists and artifact permissions.
- [ ] Add idle reclamation, orphan cleanup and failure reconciliation.
- [ ] Add operator visibility into active sessions, containers and resource consumption.

**Acceptance:** Concurrent users cannot read each other's artifacts, credentials, workspaces or session events; terminated sandboxes leave no unmanaged processes.

### Phase 4: Evaluate stronger isolation and orchestration

- [ ] Test gVisor/Kata/Firecracker compatibility and obtain needed KVM permissions.
- [ ] Compare startup latency and CPU/RAM cost against rootless Podman.
- [ ] Measure peak active workers and queueing under real traffic.
- [ ] Add another compute host or K3s only if capacity/operations justify it.

**Acceptance:** Isolation level is selected by risk classification and enforced consistently, even during capacity shortage or host failure.

## 15. Open decisions and recommended defaults

| Decision | Default |
| --- | --- |
| Agent execution location | Local for trusted tasks, dedicated host for coding/untrusted tasks |
| Initial orchestrator | Rootless Podman with an authenticated worker controller |
| Agent lifecycle | Persistent pool for lightweight agents; one container per coding session |
| Dangerous code tools | Separate disposable sandbox, even if main agent is local |
| Remote interface | Versioned internal HTTP + SSE; separate control operations |
| Routing | Backend policy, not user-supplied runtime flags |
| Authentication | Service identity + short-lived session-scoped grants |
| State | External run/session registry; reuse existing framework-specific persistence |
| Inference | Existing gateway and vLLM nodes unchanged |
| Single-port infrastructure | Nginx over approved port `9080` with tested streaming |
| Kubernetes | Only after measurable multi-host/scheduling need |
| MicroVMs | For high-risk workloads when KVM and operations permit |

The main design choice still requiring benchmarking is **whether Copilot coding sessions are best treated as session-long isolated runtimes or whether some should run as trusted persistent agents invoking isolated tools**. Both patterns should remain supported.

## 16. Final recommendation

Build the **Runtime Manager + Remote Worker API** before adding orchestration. Deploy one dedicated Podman host and retain local execution as an explicit trusted mode. Relocate the **whole Copilot SDK application** where true isolation is needed, rather than assuming that a remotely hosted headless CLI isolates SDK callbacks. Keep LangGraph's checkpointing and both frameworks' tool interfaces independent of container placement.

This creates a consistent foundation for per-user coding workspaces, browser/code sandboxes, resource quotas, future K3s/Kata nodes and cross-machine agent execution without requiring a rewrite of the portal or the existing LLM serving stack.
