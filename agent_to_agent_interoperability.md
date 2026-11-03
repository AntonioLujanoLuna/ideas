# Agent-to-Agent Interoperability with A2A (HTTP+JSON First)

**Status:** Design proposal, not deployed or benchmarked  
**Updated:** 2026-11-03  
**Scope:** Multi-user FastAPI agents portal; LangGraph and GitHub Copilot SDK; agents-as-tools; internal/remote workers; MCP; isolated microVM tool sandboxes; corporate identity; persistent tasks; vLLM inference gateway.  
**Protocol target:** A2A v1.0 family (specification v1.0.1 as researched); Python SDK v1.3.0 for the initial evaluation. Pin and retest versions before deployment.

**Related proposals:**
- [Isolated and Distributed Agent Runtime](agent_runtime_isolation_and_remote_workers.md)
- [Portal Agent Tool Execution Architecture](portal_agent_tool_execution_architecture.md)
- [Ultra-light MicroVM Agent Tool Sandboxes PoC](ultralight_microvm_agent_tool_sandboxes_poc.md)
- [Distilled Per-User Memory](agent_portal_distilled_user_memory.md)
- [OpenClaw / Copilot Agent Automation](openclaw_copilot_agents_automation_architecture.md)

---

## 1. Decision and rationale

**Use agents-as-tools as the portal's default orchestration surface. Add A2A as an OPTIONAL standardized boundary for independently deployed agents. Prefer A2A HTTP+JSON (REST-style) for our own new endpoints, with JSON-RPC support only when interoperability requires it.**

A2A is an **agent communication protocol**, not a replacement for:
- LangGraph graphs/subagents, Copilot SDK sessions/subagents, or a workflow scheduler.
- The Agent Runtime Manager, worker placement, Podman containers, or microVM execution.
- MCP, ordinary tool calls, the shared Tool Router, or client-side tools.
- FastAPI's authenticated user-facing API, task/event persistence, or the LLM gateway.

The platform can expose a single logical operation, e.g. **delegate_to_agent**, irrespective of transport. Its trusted adapter chooses a local agent, remote portal-owned agent, or A2A service. The agent/LLM must not provide arbitrary destinations, credentials, execution policy, or tenant identifiers.

Use A2A when the callee is a separately owned/versioned/deployed agent with its own task lifecycle and interface. For tightly coupled internal helpers, ordinary agent-as-tool calls are simpler.

### Decision matrix

| Scenario | Preferred mechanism | Reason |
| --- | --- | --- |
| One LangGraph node invokes another | Native graph/agent-as-tool | Shared control, no network/protocol overhead |
| Portal LangGraph orchestrates a portal-owned Copilot agent | Internal agent adapter / agent-as-tool | We control both SDKs and contract |
| Same agent moves to a dedicated CPU worker | Existing Runtime Manager RPC | Placement and execution are not A2A's job |
| Agent requests RAG search, Jira read, spreadsheet tool | Shared Tool Router / MCP | A tool is not an autonomous agent service |
| Agent runs shell/Python/tests in a microVM | SandboxExecutionProvider | Small guests should not host A2A servers |
| A separately deployed departmental specialist exposes tasks | **A2A** | Standard discovery, task/message semantics |
| Another system wants to use portal agents | **A2A server façade** | Reusable external interoperability contract |

**Do not rewrite currently working agents solely to add A2A.** Implement one adapter and one server façade first; keep native internal dispatch as a baseline for complexity and performance comparisons.

## 2. Existing portal and target architecture

Assumptions drawn from the companion proposals:

- Angular/other clients communicate with FastAPI over existing authenticated routes, streaming via SSE/WebSocket.
- FastAPI runs LangGraph and Copilot SDK agents and manages the identity, session registry and per-user policy.
- The Agent Runtime Manager supports local execution and proposed remote container/microVM configurations.
- The Tool Router governs server tools, internal MCP, client-side operations and the isolated sandbox controller.
- PostgreSQL and existing RAG/artifact/event infrastructure can store durable task correlations and outcomes.
- Agent compute is on CPU/RAM hosts; inference stays behind the existing LLM gateway and H100/H200 vLLM replicas.
- The corporate network may impose one proxied exposed port per machine. A2A URLs can be path-multiplexed through an allowed HTTPS ingress, but do not assume that the GPU gateway's port 9080 should also host A2A.

### Logical design

~~~mermaid
flowchart TB
    USER[Angular / IDE clients] --> API[FastAPI Portal API]
    API --> AUTH[Identity / ACL / policy]
    API --> ORCH[LangGraph or Copilot Orchestrator]
    ORCH --> DELEGATE[Agent-as-Tool / Delegation Service]
    DELEGATE --> LOCAL[LocalAgentAdapter]
    DELEGATE --> REMOTE[RuntimeManagerAdapter]
    DELEGATE --> A2AC[A2AClientAdapter: HTTP+JSON]
    LOCAL --> LA[Local LangGraph / Copilot]
    REMOTE --> RW[Remote portal-owned agent worker]
    A2AC --> A2AS[Independent A2A server]
    API <--> DATA[(Runs / delegation records / events / artifacts)]
    LA --> TOOLS[Shared Tool Router]
    RW --> TOOLS
    A2AS --> OWNTOOLS[Own authorized tools / resources]
    TOOLS --> MCP[MCP / RAG / Jira / client relay]
    TOOLS --> SB[MicroVM sandbox controller]
    LA --> GW[LLM Gateway]
    RW --> GW
    A2AS -. if approved .-> GW
    GW --> GPUs[H100 / H200 vLLM]
~~~

The A2A server may be a thin façade over an existing internal runtime. Serving an A2A request should not imply that the server starts a new process/VM for every turn.

The **control plane** (identity, discovery, policy, scheduling, quotas, approvals) belongs to FastAPI/portal services. The **data plane** includes actual agent execution, A2A requests/streams, tool calls, inference and artifact transfer.

## 3. A2A concepts we need

| Concept | What the specification provides | Integration requirement |
| --- | --- | --- |
| Agent Card | Agent name, description, skills, supported interfaces, security/capability metadata | Curated and validated catalog entries |
| AgentInterface | Endpoint URL, binding (HTTP+JSON, JSONRPC, gRPC), protocol version | Select compatible versions/bindings; never trust unapproved URLs |
| Message | Role, messageId, optional contextId/taskId, and Parts | Translate from internal invocation to A2A, preserving correlation |
| Part | Text, raw bytes or URL data and structured metadata | Size/mime checks, scoped artifact references |
| Task | Durable *logical* work item and server-owned ID | Persist local-to-remote mapping; authorize every operation |
| TaskStatus | Submitted, working, interrupted, or terminal state | Map to portal event types and lifecycle |
| Artifact | Named output assembled from Parts | Persist external artifact metadata; fetch contents only after authorization |
| contextId | Groups related remote interactions | Not equal to a portal session or portable memory snapshot |
| taskId | Identifies one remote task | Not a secret or proof of authorization |

### Task states

- **Nonterminal:** TASK_STATE_SUBMITTED, TASK_STATE_WORKING.
- **Interrupted:** TASK_STATE_INPUT_REQUIRED, TASK_STATE_AUTH_REQUIRED.
- **Terminal:** TASK_STATE_COMPLETED, TASK_STATE_FAILED, TASK_STATE_CANCELED, TASK_STATE_REJECTED.

An interrupted task can accept follow-up input using its taskId (and matching context). Terminal tasks cannot be resumed; a refinement becomes a new task, ordinarily within the same context.

An A2A task is not a resumable OS process or checkpoint format. Runtime crashes, non-idempotent tool side effects and worker recovery still require explicit portal-owned mechanisms.

### Key operations

- **SendMessage:** Submit a request; may yield an immediate Message or a Task. In v1.0, blocking is the default unless configuration.returnImmediately is true.
- **SendStreamingMessage:** Submit and receive incremental SSE events, including task status and artifact updates.
- **GetTask / ListTasks:** Retrieve or list authorized tasks.
- **CancelTask:** Request cancellation, then propagate to the underlying agent and its tools.
- **SubscribeToTask:** Subscribe to an existing active task's updates; do not assume complete replay of all past events.
- **GetExtendedAgentCard:** Optionally provide authenticated discovery metadata.
- **Push notification config operations:** Optional webhook-based asynchronous delivery; not needed for the first PoC.

## 4. HTTP+JSON versus JSON-RPC versus gRPC

All three are A2A bindings with the same conceptual agent/task model. HTTP+JSON and JSON-RPC both send JSON over HTTP(S). The choice changes **routing, envelopes, error representation and integration ergonomics**, not model capability or native execution latency.

| Property | HTTP+JSON (REST-style) | JSON-RPC 2.0 binding | gRPC |
| --- | --- | --- | --- |
| Routing | HTTP method + resource/action path | RPC method in JSON body, typically shared POST endpoint | Named protobuf RPC |
| Example task fetch | GET /tasks/{id} | POST with method GetTask | GetTask RPC |
| Send message | POST /message:send | POST with method SendMessage | SendMessage RPC |
| Errors | HTTP statuses with structured A2A error payload | JSON-RPC error envelope/code | gRPC status |
| Streaming | SSE | SSE | Server streaming |
| FastAPI visibility | REST routes integrate naturally with OpenAPI | Possible with route helper, less resource-oriented | Different ingress/tooling |
| Proxy policies | Straightforward method/path controls | Usually need body-aware method classification for per-operation policies | Requires gRPC-capable ingress |
| Our initial recommendation | **Preferred** | Optional compatibility | Not initially required |

**REST-style is not strictly RESTful:** actions such as :send and :cancel intentionally use POST routes. In particular, POST /tasks/{id}:cancel is not a deletion request. A2A remains stateful at the application/task level, even though it uses HTTP.

### HTTP+JSON operations (relative to each agent's advertised base endpoint)

| Method | Path | Purpose |
| --- | --- | --- |
| POST | /message:send | New/follow-up message |
| POST | /message:stream | New message with SSE |
| GET | /tasks/{id} | Get task |
| GET | /tasks | List/filter tasks |
| POST | /tasks/{id}:cancel | Cancel task |
| POST | /tasks/{id}:subscribe | Stream active task updates |
| GET | /extendedAgentCard | Extended metadata (if available) |

The specification also defines push-notification configuration endpoints; implement only if enabled. The standard well-known Agent Card discovery path is /.well-known/agent-card.json for an agent origin. For multiple agents behind one host, define a **documented per-agent discovery mapping or registry**: do not assume duplicating a root-level well-known route magically publishes multiple distinct cards.

### Illustrative HTTP+JSON wire exchange

The following is **protocol-shaped documentation**, not a running server or a deployed URL.

~~~http
POST /a2a/coding/message:send HTTP/1.1
Host: portal.internal.example
Content-Type: application/a2a+json
Accept: application/a2a+json
A2A-Version: 1.0
Authorization: Bearer <scoped-service-token>

{
  "message": {
    "messageId": "msg-1001",
    "role": "ROLE_USER",
    "parts": [
      {"text": "Review the authentication module for security issues."}
    ]
  },
  "configuration": {
    "returnImmediately": true,
    "acceptedOutputModes": ["text/plain", "application/json"]
  }
}
~~~

A server that accepts durable work may respond with a Task whose state is SUBMITTED or WORKING, and whose id and contextId are generated by the server. The client then calls GET /a2a/coding/tasks/{id}, subscribes, or polls. No client should invent a task ID and assume the server accepted it.

~~~http
GET /a2a/coding/tasks/review-task-123 HTTP/1.1
Host: portal.internal.example
Accept: application/a2a+json
A2A-Version: 1.0
Authorization: Bearer <scoped-service-token>
~~~

Streaming is requested via POST /a2a/coding/message:stream with the same message body (without necessarily setting returnImmediately), and the response uses text/event-stream. An SSE stream can contain the initial Task, TaskStatusUpdateEvent and TaskArtifactUpdateEvent; it is not simply a token stream.

### Equivalent JSON-RPC operation

~~~http
POST /a2a/coding/rpc HTTP/1.1
Host: portal.internal.example
Content-Type: application/json
A2A-Version: 1.0
Authorization: Bearer <scoped-service-token>

{
  "jsonrpc": "2.0",
  "id": "rpc-1001",
  "method": "SendMessage",
  "params": {
    "message": {
      "messageId": "msg-1001",
      "role": "ROLE_USER",
      "parts": [
        {"text": "Review the authentication module for security issues."}
      ]
    }
  }
}
~~~

In v1.0 the JSON-RPC method name is **SendMessage**, not legacy message/send. If both bindings are advertised in supportedInterfaces, implement the same effective capabilities and authorization semantics on both. The client chooses a compatible binding from the Agent Card.

### Why choose HTTP+JSON here?

1. FastAPI/Angular integrations, access logs, OpenAPI documentation and HTTP tracing are familiar.
2. NGINX can route/authorize by method and path rather than inspecting a JSON-RPC body.
3. Debugging a failed GET /tasks/{id} is more direct than inspecting a generic RPC request.
4. A2A Python SDK provides native REST route factory support.

This is an **operational preference**, not a measured throughput advantage. The A2A protocol overhead should be benchmarked separately from model inference and tool execution.

## 5. Discovery, registry and capability selection

**Do not expose unrestricted Agent Card URL fetching to the model.** Catalog entries should be registered and reviewed by portal administrators or a trusted provisioning flow.

Suggested registry schema:

| Field | Meaning |
| --- | --- |
| agent_id, owner_team, display_name | Portal-controlled stable identity |
| card_source_url, card_hash, last_verified_at | Provenance and monitoring |
| interface URL, protocolBinding, protocolVersion | Approved endpoints |
| skills, input/output media types | Selection hints, not authorization |
| trust_class, tenant_scope, data_classification | Deployment/security policy |
| required_auth, allowed_callers, allowed_delegations | Service access constraints |
| limits: concurrency, deadlines, tokens, depth | Resource control |
| health/status | Whether dispatch is currently eligible |

**An Agent Card skill is descriptive, not a strictly typed callable-tool schema.** For deterministic delegation, attach a portal-owned input/output schema to the approved registry entry; validate request and response contracts independently.

Even if an Agent Card advertises a new skill or changes its URL, those changes do not automatically grant permission. Pin/approve endpoint identities and maintain versioned compatibility checks.

Agent Cards may use well-known discovery, internal registry publication, or direct configuration. Prefer **registry first** for corporate use, then allow carefully reviewed cross-team federation.

## 6. Unifying agents-as-tools and A2A

Retain a framework-independent delegation interface, with **transport-independent** semantics:

~~~text
start(agent_id, authenticated_request) -> DelegationHandle
get(handle) -> DelegationState
events(handle, cursor) -> normalized events
respond(handle, message_or_approval) -> DelegationState
cancel(handle) -> terminal_or_cancelling_state
~~~

This is a design contract, **not existing SDK code**.

Implement:
- **LocalAgentAdapter:** Direct invocation of portal-owned LangGraph/Copilot agents.
- **RemoteRuntimeAgentAdapter:** Existing Agent Runtime Manager and worker-control API, with server-owned placement.
- **A2AAgentAdapter:** Resolve trusted Agent Card, negotiate HTTP+JSON 1.0, send/stream/get/cancel/continue, normalize results.

The orchestrating agent sees an approved agent-as-tool interface. The backend decides which adapter handles the delegation. It may also expose selected portal-owned agents through an inbound A2A server façade so other systems can invoke them.

Avoid an unrestricted generic tool taking arbitrary agent URL and headers. Prefer an allowlisted agent_id with validated task arguments and scoped artifact IDs.

### Identifier mapping (durable)

| Internal identifier | Role |
| --- | --- |
| portal_run_id | Original top-level portal execution |
| delegation_id | One child agent invocation |
| parent_delegation_id | Delegation tree |
| internal_agent_id | Catalog entry and permission boundary |
| worker_run_id | Runtime Manager's execution handle |
| remote_task_id | A2A server's taskId |
| remote_context_id | A2A server's contextId |
| agent_session_ref | Framework-specific LangGraph thread / Copilot session |
| trace_id | Cross-agent telemetry correlation |
| artifact_refs | Authorized artifact location/version/digest |

Never conflate the remote Task ID, the portal run ID, the authenticated user session, and a LangGraph thread or Copilot SDK session. One portal run may produce many A2A tasks; a remote context may span several tasks.

## 7. Framework-specific integration

### LangGraph

- Wrap an existing graph in an A2A AgentExecutor without moving the graph's orchestration logic into protocol handlers.
- Map A2A context/session identity to a **server-owned** LangGraph thread ID via an authorized mapping, not untrusted arbitrary IDs.
- Translate graph stream events to A2A status/artifact updates, with bounded buffers and output-size limits.
- For interruption/human input, persist the graph checkpointer and return INPUT_REQUIRED only when its state is genuinely resumable.
- Cancellation must reach the graph and any executing tools. A checkpoint alone does not make external writes automatically reversible.

### GitHub Copilot SDK

- Keep SDK session management, CLI/runtime placement, permission handlers and custom tools in the existing adapter.
- Provide an A2A façade over the **entire** agent, not just the Copilot CLI remote runtime connection.
- Resume/continuation must be mapped to the actual underlying SDK session if supported; do not promise process-level recovery after worker death.
- Only expose approved tools, and do not use blanket approval for sensitive actions. Preserve the portal's client-tool relay where necessary.
- A2A does not confer filesystem or host permissions; code execution still goes through controlled workers or sandboxes.

### MCP and microVM separation

- A2A **agent to agent**: autonomous task delegation and results.
- MCP / shared Tool Router **agent to tool**: bounded tools/resources with schemas and permissions.
- Sandbox controller **agent to isolated execution environment**: shell, code, file operations.
- Runtime Manager **portal to worker**: process placement, lifecycle, quotas, worker state.

Do not host A2A servers in ultra-light shell/Python microVM guests. Keep those guests minimal and isolated.

## 8. Implementing the HTTP+JSON server with the Python SDK

Reference baseline verified against the official SDK migration guide:

- Package: a2a-sdk, with the FastAPI extra.
- Target initial SDK: v1.3.0, with lockfile pin.
- Server components: AgentExecutor, DefaultRequestHandler, task store, AgentCard, and route factories.
- REST route factory: create_rest_routes(request_handler, path_prefix=...).
- Card route factory: create_agent_card_routes(agent_card).
- FastAPI helper: add_a2a_routes_to_fastapi() for OpenAPI-visible routes.
- JSON-RPC is optional via create_jsonrpc_routes().

~~~bash
python -m pip install "a2a-sdk[fastapi]==1.3.0" "uvicorn>=0.30,<1"
~~~

A full application must supply a concrete executor, configured handler, auth and task storage. To avoid pretending an implementation exists, the following function **only mounts already-constructed components** and does not instantiate a fake agent:

~~~python
"""FastAPI A2A HTTP+JSON mounting utility for configured agents."""

from a2a.server.request_handlers import RequestHandler
from a2a.server.routes import (
    add_a2a_routes_to_fastapi,
    create_agent_card_routes,
    create_rest_routes,
)
from a2a.types import AgentCard
from fastapi import FastAPI


def create_a2a_rest_app(
    agent_card: AgentCard,
    request_handler: RequestHandler,
    path_prefix: str = "/a2a/coding",
) -> FastAPI:
    """Mount REST-style A2A routes and the Agent Card.

    Args:
        agent_card: Fully populated card advertising the externally
            reachable HTTP+JSON interface and supported version.
        request_handler: A configured SDK RequestHandler backed by a
            concrete AgentExecutor and task store.
        path_prefix: Canonical ingress prefix for the agent's REST API.

    Returns:
        FastAPI app containing registered A2A routes. Authentication,
        storage and deployment are configured by the application owner.
    """
    app = FastAPI(title="Corporate A2A Agent")

    add_a2a_routes_to_fastapi(
        app,
        agent_card_routes=create_agent_card_routes(agent_card),
        rest_routes=create_rest_routes(
            request_handler,
            path_prefix=path_prefix,
        ),
    )
    return app
~~~

**Integration caveats:**
- Treat this as a module-level integration utility, not a production service or tested end-to-end example. The SDK/route signatures should be pinned and exercised by the PoC.
- If multiple agents share one FastAPI application/NGINX origin, ensure distinct Agent Card discovery endpoints and path/host routing. Test the advertised external URLs through the proxy, not only localhost.
- FastAPI(routes=...) can mount Starlette routes but might not expose them in /docs; the SDK's FastAPI helper addresses that.
- The v1.0 SDK removed earlier A2AFastApiApplication/A2ARESTFastApiApplication classes; do not copy pre-1.0 tutorials.
- An executor must consistently produce either a standalone Message **or** an initial Task followed by task updates; mixing these causes protocol errors.
- A known late-October 2026 SDK issue concerns REST tenant routes with path_prefix. Avoid relying on that combination until its fix is released and tested; use explicit auth-scoped tenant routing instead.

## 9. Security model (mandatory for corporate use)

### Identity and auth

1. **User identity:** FastAPI authenticates human users via the current corporate identity provider.
2. **Calling service:** An A2A caller authenticates to the serving agent (e.g., TLS + OAuth2 service identity or mTLS).
3. **Delegated authority:** If the request acts for a user, the backend issues a narrow short-lived audience-scoped capability or performs a trusted token exchange. Never forward arbitrary user bearer tokens or give remote agents the portal's broad service credentials.
4. **Authorization:** Independently enforce agent, skill, tool, workspace, tenant, artifact and action permissions at the serving side.
5. **Approval:** Backend-mediated, bound to exact action/arguments/resources; A2A INPUT_REQUIRED/AUTH_REQUIRED alone is not authorization.

TaskId/contextId are identifiers, **never** access credentials. Authorize GetTask, ListTasks, CancelTask, SubscribeToTask, artifacts and any follow-up message on every call.

### Agent Cards and SSRF

- Approve remote hostnames/ports/certificates, resolve and restrict redirects, and block arbitrary metadata-provided URLs.
- Validate outbound egress against DNS rebinding, loopback, private network, metadata endpoints and unapproved hosts.
- Separate internal Agent Cards/extended metadata from public discovery.
- Do not treat card descriptions, skills or remote tool outputs as system instructions.

### Delegation and data security

- Limit delegation depth, fan-out, total model tokens, per-task runtime, CPU work, artifact bytes and concurrent agent tasks.
- Track caller/user/tenant and transitive invocation chain; reject unauthorized cycles or recursive delegation.
- Use scoped workspace/artifact references and validate every dereference; minimize prompt and conversation data sent to remote agents.
- Apply data-classification policy before cross-team or external delegation.
- Keep large code diffs, binaries and documents in approved artifact storage; use hashes, short-lived grants and audit trails.
- Log security decisions and durations, not raw secrets or unredacted private outputs.

### Push notifications

Initially **disable** A2A push notifications. Prefer SSE and GetTask. Before enabling webhooks, enforce allowlisted callback hosts, HTTPS, signed/authenticated delivery, limits, deduplication, validation at configuration and dispatch, and independently enforced outbound network restrictions.

## 10. Execution lifecycle, persistence and reliability

A2A standardizes externally visible task states; it does not provide exactly-once execution, cross-host worker reconciliation or guaranteed full SSE history replay.

Suggested database tables (conceptual):

- agent_registry: approved Agent Cards, interfaces, trust and policies.
- agent_delegations: parent/child run relationships, remote task/context IDs, deadlines, status.
- agent_delegation_events: normalized ordered portal events with durable sequence/cursor.
- agent_artifacts: references, digest, MIME type, permissions, expiry.
- agent_execution_leases: executing worker, heartbeat and reconciliation state.
- agent_side_effects: idempotency keys, approval IDs, committed external effects.

### Recommended workflow

1. Authenticate user and parent portal run; resolve allowed target agent from the registry.
2. Allocate delegation_id, idempotency key and a bounded resource budget.
3. Dispatch using the selected adapter; store resulting remote taskId/contextId and trace correlation.
4. Stream or poll A2A updates and normalize them into **portal-owned durable events**.
5. Forward only safe, authorized output to the parent agent and Angular UI.
6. If INPUT_REQUIRED or AUTH_REQUIRED, persist pending state and ask the correct user through FastAPI.
7. On continuation, check identity/approvals and send a follow-up message using the same live remote task.
8. On completion, fetch/validate artifacts, finalize local state and release reservations.
9. On timeout, cancel the remote task and underlying tools where possible; reconcile orphaned execution.
10. On disconnect/restart, recover from durable task state and GetTask, not from an assumed complete SSE replay.

**Retries:** The specification's messageId is not an exactly-once guarantee. Track application-level idempotency, deduplicate messages, and use a transaction/outbox pattern for sensitive writes (Jira, Git push, deployments). Treat uncertain network failures as potentially executed before retrying.

**Cancellation:** A successful A2A cancel response must be verified against the actual worker/tool lifecycle. Side effects already performed may not be reversible.

**Approval semantics:** Represent approval as a signed portal decision against a particular operation. Never convert free-form agent text into blanket permission.

## 11. API and network deployment

Keep existing user-facing routes separate:

~~~text
/api/agents
/api/sessions
/api/runs
/api/tools
/api/workspaces
~~~

Expose independent agent services, if approved, via path-based HTTPS ingress:

~~~text
/a2a/research/message:send
/a2a/research/message:stream
/a2a/research/tasks/{id}
/a2a/research/tasks/{id}:cancel

/a2a/coding/message:send
/a2a/coding/message:stream
/a2a/coding/tasks/{id}
/a2a/coding/tasks/{id}:cancel
~~~

The agent base URL and discovery card path must be tested as a pair. A central internal registry can publish the exact authenticated card locations, avoiding ambiguous shared /.well-known routes.

NGINX considerations:
- Terminate TLS and authenticate at a trusted boundary; forward verified identity to the upstream, never trust arbitrary caller-supplied identity headers.
- Preserve A2A-Version and relevant Accept/Content-Type headers.
- Disable buffering on SSE endpoints, configure long-lived read timeouts and client cancellation behavior.
- Protect request body sizes, response size, header forwarding and rate limits.
- Route and audit per agent/path; avoid exposing worker controller, sandbox APIs, or raw model GPU ports directly.
- Use internal service DNS or mTLS between gateway and agent services when approved by corporate infrastructure.
- Path-based auth policies must check task ownership at the serving application, not merely the route.

**No change is required to vLLM, KV cache, or llm-d to support A2A.** However, multi-agent fan-out will increase aggregate inference demand; integrate it with existing gateway quotas and tracing.

## 12. Performance and capacity evaluation

Compare **native agent-as-tool** and **A2A HTTP+JSON** for equivalent work and deployment location. Also measure JSON-RPC only if interoperability drives its adoption.

Record:
- Discovery/Agent Card validation latency and caching behavior.
- Transport overhead: submit-to-task-ack p50/p95/p99, GetTask round-trip, SSE event lag.
- End-to-end agent turn latency split into queue, A2A transport, prefill/TTFT, decode, tool calls and artifact transfer.
- Concurrency 1, 10, 50 and realistic developer workloads, with portal concurrency ceilings.
- Idle/active worker CPU/RAM, connection counts, SSE disconnects/reconnects.
- Event store throughput, backpressure, task polling cost, artifact size distribution.
- Cancellation and recovery correctness.
- Incremental token consumption and KV cache hit behavior from the underlying gateway.

Set empirical rollout thresholds after baseline measurement; do not assert that A2A improves model throughput. A remote multi-agent workflow may **increase** inference requests, TTFT pressure and aggregate token demand.

## 13. Proof of concept

**PoC objective:** One LangGraph portal orchestrator delegates work to one independently exposed agent. The second agent is initially a trivial deterministic service, then a real Copilot SDK or LangGraph agent. Preserve your original direct wrapper as the comparison baseline.

### Phase 0. Protocol readiness

- Pin the SDK at a tested v1.3.0 release and inspect current release notes/known issues.
- Run the official SDK example that exposes HTTP+JSON, then retrieve the card and invoke message:send and message:stream.
- Run the A2A Technology Compatibility Kit (TCK) using its documented command set and JSON/HTTP transport configuration.
- Confirm the A2A-Version: 1.0 request header and HTTP+JSON media type behaviour through proxy and SDK.

### Phase 1. Deterministic service

- Deploy a lightweight A2A server with a single deterministic skill and an in-memory task store **only for development**.
- Implement SendMessage, GetTask, SendStreamingMessage, a meaningful artifact, one INPUT_REQUIRED flow and cancellation semantics.
- Validate that the same logical operation works using the portal's internal adapter and the A2A client adapter.

### Phase 2. Real framework interoperability

- Wrap a read-only LangGraph research agent with an AgentExecutor.
- Let a Copilot SDK agent invoke it through the portal's approved delegate_to_agent tool, or reverse direction.
- Compare direct agent-as-tool versus A2A for correctness, latency, code complexity and auditability.
- Keep the underlying agent's framework checkpointer/session ownership unchanged.

### Phase 3. Durable multi-user execution

- Add authenticated catalog entries and scoped A2A service identity.
- Persist delegation mapping and normalized events.
- Verify task retrieval/authorization after FastAPI restart and worker restart (only guarantee supported recovery).
- Run multi-user negative tests: cross-tenant task access, malicious Agent Cards, SSRF, fake approval, canceled tools and excessive delegation.

### Phase 4. Optional federation

- Expose one portal-owned specialist agent as an inbound A2A service for another team.
- Support REST as the preferred binding; advertise JSON-RPC only if the consuming team requires it.
- Evaluate a second independent agent implementation to prove actual interoperability, not just same-SDK loopback.

### Acceptance criteria

- An approved caller can discover the agent and complete a delegated task through A2A HTTP+JSON.
- A caller with insufficient rights cannot read/cancel another user's task or artifacts.
- Streaming, interruption/continuation and cancellation map to visible portal events.
- A2A agent results and artifacts are validated, traceable and recoverable according to documented guarantees.
- No tool sandbox or worker host receives broader network, identity or filesystem privileges because A2A is enabled.
- Latency, connection and token costs are measured against the agent-as-tool baseline.
- The SDK's official conformance checks pass for the advertised binding/capabilities, with remaining limitations documented.

## 14. Risks and open questions

1. **Do we actually need agent federation yet?** If all agents are portal-owned and tightly coupled, A2A may add needless complexity.
2. **Which remote tasks deserve persistence?** LangGraph checkpoints and Copilot SDK session persistence differ; define explicit guarantees.
3. **How should structured specialist input contracts be versioned?** Agent Card skills are descriptive; use our own schema registry for deterministic calls.
4. **Should A2A be ingress-only, egress-only, or both?** Start outbound; expose inbound only after security review.
5. **Which identity delegation pattern does corporate SSO permit?** Do not assume generic bearer forwarding is allowed.
6. **Are remote agents allowed to use the central model gateway?** Enforce separate credentials, budgets and network policy.
7. **Should the portal choose one agent or a multi-agent plan?** Keep centralized workflow control and avoid unconstrained automatic fan-out.
8. **Can REST path prefixes and tenant routing be implemented reliably with the pinned SDK?** Validate against current open issues.
9. **Should external agents retain task history?** Establish retention, data classification, deletion and audit requirements.

## 15. Recommended implementation sequence

1. Keep the existing agent-as-tool interface and document a stable internal DelegationHandle contract.
2. Add an A2A HTTP+JSON **client adapter** behind it; call one deterministic trusted service.
3. Add FastAPI A2A **server façade** to one existing LangGraph agent.
4. Build durable task mapping, normalized events and artifact references.
5. Verify auth/tenant isolation, interrupts, cancellations, idempotency and recovery.
6. Benchmark direct versus A2A delegation before making deployment decisions.
7. Introduce JSON-RPC compatibility and broader registry-based discovery only when an actual integrator needs them.

**Bottom line:** Treat A2A as a versioned network interoperability protocol. It makes independently owned agents addressable without sharing framework internals. It is *not* required for every local helper or microVM tool, and it should not displace the portal's identity, agent orchestration or runtime management.

## 16. Authoritative references

- A2A 1.0 specification: https://a2a-protocol.org/latest/specification/
- A2A concepts and A2A/MCP positioning: https://a2a-protocol.org/latest/topics/what-is-a2a/
- Official Python SDK and installation: https://github.com/a2aproject/a2a-python
- SDK v0.3 to v1.0 migration guide (route factories, FastAPI, behavior): https://github.com/a2aproject/a2a-python/blob/main/docs/migrations/v1_0/README.md
- Python SDK releases (v1.3.0 on 2026-10-27): https://github.com/a2aproject/a2a-python/releases
- A2A Technology Compatibility Kit: https://github.com/a2aproject/a2a-tck
- Official SDK code samples: https://github.com/a2aproject/a2a-python/tree/main/samples
- Tracked REST tenant path-prefix issue as of 2026-11-03: https://github.com/a2aproject/a2a-python/issues/1376

**Validation status:** The protocol design, relevant official SDK APIs, and existing portal design documents were reviewed. No A2A service was deployed, no SDK example was executed, and no performance benchmark has been run as part of this proposal.
