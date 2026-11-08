# Agents Portal: Self-Hosted Performance and Workflow Testing with Grafana k6

**Status:** Design and PoC plan, not implemented or benchmarked  
**Updated:** 2026-11-08  
**Scope:** LLM gateway, vLLM endpoints, LangGraph agents, GitHub Copilot SDK agents, agent tool execution, subagents, streaming, stateful conversations, and the Agents Portal UI.  
**Hosting constraint:** Load generators, metric storage, dashboards, tracing services, test fixtures, and optional MCP integrations must be deployed on infrastructure controlled by the organization. No Grafana Cloud, hosted load generation, remote MCP service, or external tracing SaaS is required.

**Related proposals:** [llm-d cache-aware routing](llm_d_cache_aware_vllm_routing.md), [path-multiplexed vLLM replicas](llm_d_path_multiplexed_vllm_replicas.md), [portal agent tool execution](portal_agent_tool_execution_architecture.md), [agent runtime isolation](agent_runtime_isolation_and_remote_workers.md), [ultra-light microVM PoC](ultralight_microvm_agent_tool_sandboxes_poc.md), [A2A interoperability](agent_to_agent_interoperability.md), [browser agent architecture](browser_agent_architecture.md).

## 1. Recommendation and mental model

**Adopt k6 as the primary load generator**, not as the entire agent-evaluation system. Complement it with:

- **Streaming-aware measurement:** `xk6-sse` (community extension) or a purpose-built streaming client for SSE/TTFT/ITL.
- **Backend observability:** OpenTelemetry spans and Prometheus metrics for gateway, LangGraph nodes, tools, subagents, microVM workers, and vLLM.
- **Inference microbenchmarks:** vLLM's own serving benchmark and representative token distributions to validate raw backend capacity independently of the portal.
- **Functional evaluation:** deterministic test fixtures, artifact assertions, trace checks, and optional local LLM-based judges for nondeterministic tasks. An HTTP 200 or plausible final answer is insufficient evidence of agent correctness.
- **Browser realism:** a small number of `k6/browser` Chromium sessions under concurrent API-level load, rather than hundreds of browser instances.

A **single agent run is a transaction**: one user-facing request can lead to many graph-node executions, sequential/parallel model calls, tools, subagents, checkpoint writes, and streamed events. Measure both the outer user-visible transaction and every expensive internal stage.

### Intended architecture

```text
k6 load generator (separate CPU machine; self-hosted)
  |
  +-- HTTP/SSE/WebSocket/Chromium --> Agents Portal API and frontend
  |                                      |
  |                                      +-- LangGraph / Copilot SDK runtime
  |                                      |     |
  |                                      |     +-- tools / MCP / subagents / sandbox
  |                                      |     +-- checkpointer / state / queues
  |                                      |
  |                                      +-- LLM gateway (NGINX, port 9080)
  |                                                |
  |                                                +-- H100  -> model A
  |                                                +-- H200  -> model B
  |                                                +-- H200  -> model C
  |
  +-- Direct-to-gateway and direct-to-vLLM baseline tests

OpenTelemetry from backend, gateway, tools and agent runtime
    -> self-hosted collector -> Prometheus-compatible metrics / optional traces
k6 -> Prometheus remote write -> self-hosted Prometheus -> Grafana
```

The H100/H200 arrangement, three independently serving replicas, path routing, 262k target context and KV/cache-aware routing are the system's design context, **not benchmarked capacity assertions**. Do not presume that 200–300 registered developers imply 200–300 simultaneous active generations.

## 2. What can be tested

| Layer | Example transaction | Measurements | Correctness condition |
| --- | --- | --- | --- |
| Gateway-only | POST a mocked completion via gateway | gateway overhead, RPS, p95/p99, failures | correct route and response |
| Inference-only | Call each vLLM replica directly | TTFT, inter-token latency, output tokens/s, saturation | complete, valid response |
| Gateway + vLLM | Call `/modelA`, `/modelB`, `/modelC` routes and pooled gateway | routing distribution, queueing, prefix affinity, prefill/decode | correct model/route and output |
| Single LangGraph run | one graph invocation | run duration, model calls, node time | specified final state |
| Stateful conversation | create thread, send multiple messages | latency per turn, persistence/checkpoint overhead | retained conversation state |
| Tool-using agent | read a file, fetch a document, run a command | tool selection/calls, tool latency, retries | expected tool trace **and** resulting artifact |
| Coding agent | inspect repository, patch, execute tests | workflow duration, sandbox lifecycle, I/O | tests pass and expected diff |
| Supervisor / subagents | delegate multiple tasks | fan-out, parallelism, amplification, wait times | verified child results |
| Browser / portal | sign in, start chat, stream, cancel, resume | page latency, UI responsiveness, dropped stream | visible and persisted user result |
| Long-running workflow | interrupt, user approval, resume | idle state and resumption latency | no duplicate side effects |
| Resilience | failures/timeouts/disconnect/restart | recovery, error propagation, cleanup | expected safe failure/retry |

Keep gateway, orchestration, tool, persistence, and full E2E tests in **separate suites**. Otherwise a slow model masks a fast gateway, and a bad tool masks a healthy model.

## 3. What k6 does, and what it does not

k6 executes scripted operations using virtual users (VUs) and emits performance metrics. A VU can perform an entire agent session across several requests and state transitions.

- **One request:** a script with one VU and one iteration.
- **A conversation:** an iteration creates a thread and makes several sequential run calls.
- **An agentic workflow:** an iteration starts a graph, waits or polls until terminal status, and validates a fixture.
- **Concurrent agents:** multiple VUs execute independent workflows; all runs must use unique thread/workspace identifiers.
- **Arrival-rate load:** initiate a controlled number of *new workflows per second*, regardless of prior completion, provided adequate VUs exist.
- **Continuous active sessions:** constant/ramping VUs repeatedly run workflows; this is a **closed workload** and total request rate decreases if agents slow down.

k6 cannot inspect internal LangGraph nodes just by observing an HTTP response. Add tracing and counters inside the backend. Plain `k6/http` buffers HTTP responses; use an SSE-aware client to measure first meaningful streamed tokens and stream gaps. Avoid equating a first SSE metadata event with a first generated token.

A request that returns `202 Accepted` or a `run_id` is only **submitted**. Do not record workflow completion until the backend reports a terminal status and the expected output/artifacts are verified. Handle cancellations, interrupts and timeouts explicitly.

## 4. API targets: distinguish real deployments

### 4.1 Your own Agents Portal backend

The portal already runs agents in the backend. The **preferred E2E target** is its real public/internal API, using its actual authentication, message/session creation, run submission, streaming and cancellation mechanisms.

**Do not assume** that using the LangGraph Python library automatically exposes LangGraph Agent Server endpoints. If the portal has custom HTTP routes, write a thin k6 adapter to those exact routes. If it internally invokes graphs via `graph.ainvoke()` or `graph.astream()`, the API layer determines how k6 can access them.

Before implementing the adapter, inventory:

1. authentication headers/cookies, CSRF and rate-limit behavior;
2. conversation/thread creation and correlation IDs;
3. synchronous vs asynchronous run initiation;
4. run completion/status lookup and terminal state representation;
5. SSE or WebSocket event schemas, heartbeat and end-of-stream conventions;
6. cancellation, interrupt, approval/resume, retry, and disconnect semantics;
7. artifact and file-execution APIs.

### 4.2 If a LangGraph-compatible Agent Server is deployed

Common Agent Protocol endpoints include:

| Purpose | Example endpoint |
| --- | --- |
| Create thread | `POST /threads` |
| Stateless wait for completion | `POST /runs/wait` |
| Stateful wait for completion | `POST /threads/{thread_id}/runs/wait` |
| Stateless stream | `POST /runs/stream` |
| Stateful stream | `POST /threads/{thread_id}/runs/stream` |
| Delete test thread | `DELETE /threads/{thread_id}` |

A typical messages-based graph accepts:

```json
{
  "assistant_id": "agent",
  "input": {
    "messages": [
      { "role": "user", "content": "Reply with exactly K6_OK." }
    ]
  }
}
```

`assistant_id` can refer to an assistant configuration or graph name as supported by the deployed Agent Server. A different graph may require a different input and final-state schema. The examples below **assume** a compatible server with a `messages` state. Verify actual routes and semantics on your deployment before running load.

### 4.3 Gateway and model endpoints

Use the configured, actual model aliases and gateway paths. Conceptually:

- direct `http://replica-host:port/v1/chat/completions`;
- gateway `https://gateway-host:9080/modelA/v1/chat/completions`, similarly for model B/C;
- common/pooled routing endpoint **only if** implemented.

Avoid interpreting path multiplexing alone as dynamic load balancing. The documented gateway may expose per-model routes; a cache-aware router is a distinct, separately testable layer.

## 5. Workload matrix and progression

Begin with one deterministic request. Progress to representative usage before stress.

| Workload | Initial single-run test | Load dimension | Particularly important measurement |
| --- | --- | --- | --- |
| Plain chat | short prompt, 128–512 output tokens | 1/5/10/20/30/50+ requests | TTFT, TPOT, output tokens/s |
| Medium coding chat | 4k–16k input tokens | repeated conversations | cache affinity, context growth |
| Long-context chat | 32k/64k/128k/262k **where supported** | low concurrency first | KV pressure, queueing, OOM, truncation |
| Multi-turn | 5–20 turns per thread | active user sessions | checkpoints, reuse, p95 per turn |
| Agent with tools | 5/20/50 calls in a workflow | concurrent workflows | tool latency, retries, amplification |
| Coding agent | inspect/edit/test a disposable repo | full-session throughput | sandbox startup, command time, correctness |
| Subagents | supervisor launches 2–5 children | nested parallel runs | peak in-flight inference |
| Streaming | SSE chunks and final result | event rate and concurrent streams | TTFT, gaps, cancellations |
| UI | login, submit prompt, render output | tens of browsers, not hundreds | browser experience under backend load |

These are **test configurations**, not predictions of achievable capacity.

Run each benchmark with (a) mocked LLM/tools where appropriate, then (b) real inference/tools, and (c) through the full portal. Keep prompt datasets, model versions, cache policy, max tokens, generation settings, and warm-up procedures consistent between comparable runs.

## 6. Complete k6 example: single HTTP request

The following can be saved as `http-k6.js`. Set the absolute URL and serialized JSON body for the target endpoint. It is suitable for an OpenAI-compatible completion endpoint with `stream:false` or a simple Portal POST route.

```javascript
/**
 * Execute one HTTP JSON POST and assert the expected response status.
 *
 * Required: TARGET_URL, JSON_BODY.
 * Optional: EXPECTED_STATUS (default 200), API_TOKEN, AUTH_HEADER.
 *
 * Example:
 *   k6 run -e TARGET_URL="$TARGET_URL" -e JSON_BODY="$JSON_BODY" http-k6.js
 */
import http from 'k6/http';
import { check } from 'k6';

const targetUrl = __ENV.TARGET_URL;
const jsonBody = __ENV.JSON_BODY;
const expectedStatus = Number(__ENV.EXPECTED_STATUS || 200);

if (!targetUrl || !jsonBody) {
  throw new Error('Set TARGET_URL and JSON_BODY.');
}
JSON.parse(jsonBody);

export const options = {
  vus: 1,
  iterations: 1,
  thresholds: {
    checks: ['rate==1'],
    http_req_failed: ['rate==0'],
  },
};

/** POST JSON once, recording response time and status. */
export default function () {
  const headers = {
    'Content-Type': 'application/json',
    'X-Load-Test': 'k6',
  };
  if (__ENV.API_TOKEN) {
    const name = __ENV.AUTH_HEADER || 'Authorization';
    headers[name] = name.toLowerCase() === 'authorization'
      ? 'Bearer ' + __ENV.API_TOKEN
      : __ENV.API_TOKEN;
  }
  const response = http.post(targetUrl, jsonBody, {
    headers,
    timeout: '10m',
    tags: { name: 'single_http_post', operation: 'single_http_post' },
  });
  check(response, {
    'expected HTTP status': (r) => r.status === expectedStatus,
  });
  console.log('status=' + response.status +
    ' latency_ms=' + response.timings.duration);
}
```

Example invocation (substitute the actual route and model identifier):

```bash
export TARGET_URL="https://gateway.example.internal:9080/modelA/v1/chat/completions"
export JSON_BODY='{"model":"MODEL_ALIAS","messages":[{"role":"user","content":"Reply K6_OK"}],"max_tokens":32,"stream":false}'
k6 run -e TARGET_URL="$TARGET_URL" -e JSON_BODY="$JSON_BODY" http-k6.js
```

This measures **full response completion**, not TTFT. On self-hosted TLS endpoints, configure trusted internal CAs rather than disabling certificate verification.

## 7. Complete k6 example: LangGraph runs, threads, tools and concurrency

Save as `langgraph-k6.js`. The script uses actual Agent Server endpoints; adapt `postJson`, `createThread` and `runAgent` for the Portal's own API if no Agent Server is exposed.

Each workflow is marked successful only after a terminal response is parsed and the assertion passes. The tool test uses a hidden fixture expected value, although validating tool **usage** still requires traces or artifact checks.

```javascript
/**
 * k6 workflows for a messages-based LangGraph-compatible Agent Server.
 *
 * Required: BASE_URL, ASSISTANT_ID.
 * For agentic/load: AGENT_TASK, AGENT_EXPECT.
 * Optional: API_TOKEN, AUTH_HEADER, CLEANUP=true, VUS=5, DURATION=2m.
 *
 * Select one:
 *   k6 run --scenario single langgraph-k6.js
 *   k6 run --scenario conversation langgraph-k6.js
 *   k6 run --scenario agentic langgraph-k6.js
 *   k6 run --scenario load langgraph-k6.js
 */
import http from 'k6/http';
import { check } from 'k6';
import { Rate, Trend } from 'k6/metrics';
import exec from 'k6/execution';

const BASE_URL = (__ENV.BASE_URL || '').replace(/\/$/, '');
const ASSISTANT_ID = __ENV.ASSISTANT_ID;
const TASK = __ENV.AGENT_TASK;
const EXPECTED = __ENV.AGENT_EXPECT;
const CLEANUP = __ENV.CLEANUP === 'true';

if (!BASE_URL || !ASSISTANT_ID) {
  throw new Error('Set BASE_URL and ASSISTANT_ID.');
}
const headers = {
  'Content-Type': 'application/json',
  'X-Load-Test': 'k6',
};
if (__ENV.API_TOKEN) {
  const name = __ENV.AUTH_HEADER || 'Authorization';
  headers[name] = name.toLowerCase() === 'authorization'
    ? 'Bearer ' + __ENV.API_TOKEN
    : __ENV.API_TOKEN;
}

const workflowDuration = new Trend('agent_workflow_duration_ms', true);
const workflowSuccess = new Rate('agent_workflow_success');

export const options = {
  scenarios: {
    single: {
      executor: 'per-vu-iterations',
      vus: 1,
      iterations: 1,
      exec: 'single',
      maxDuration: '10m',
    },
    conversation: {
      executor: 'per-vu-iterations',
      vus: 1,
      iterations: 1,
      exec: 'conversation',
      maxDuration: '10m',
    },
    agentic: {
      executor: 'per-vu-iterations',
      vus: 1,
      iterations: 1,
      exec: 'agentic',
      maxDuration: '15m',
    },
    load: {
      executor: 'constant-vus',
      vus: Number(__ENV.VUS || 5),
      duration: __ENV.DURATION || '2m',
      gracefulStop: '10m',
      exec: 'agentic',
    },
  },
  thresholds: {
    agent_workflow_success: ['rate>0.99'],
    http_req_failed: ['rate<0.01'],
  },
};

/**
 * POST JSON and return a parsed response; throw on invalid status or JSON.
 * @param {string} path - Path relative to the base URL.
 * @param {object} payload - JSON request.
 * @param {string} operation - Low-cardinality metric label.
 * @returns {object} Parsed response.
 */
function postJson(path, payload, operation) {
  const response = http.post(BASE_URL + path, JSON.stringify(payload), {
    headers,
    timeout: '10m',
    tags: { name: operation, operation },
  });
  check(response, {
    'HTTP 2xx': (r) => r.status >= 200 && r.status < 300,
  }, { operation });
  if (response.status < 200 || response.status >= 300) {
    throw new Error(operation + ' HTTP ' + response.status);
  }
  try {
    return response.json();
  } catch (e) {
    throw new Error(operation + ' invalid JSON: ' + String(e));
  }
}

/** Create a new per-workflow thread; return its identifier. */
function createThread() {
  const thread = postJson('/threads', {}, 'create_thread');
  if (!thread || typeof thread.thread_id !== 'string') {
    throw new Error('Missing thread_id');
  }
  return thread.thread_id;
}

/**
 * Execute to completion using the synchronous wait API.
 * @param {string|null} threadId - Existing thread; null for stateless.
 * @param {string} prompt - New user message.
 * @returns {object} Final graph state.
 */
function runAgent(threadId, prompt) {
  const path = threadId
    ? '/threads/' + encodeURIComponent(threadId) + '/runs/wait'
    : '/runs/wait';
  return postJson(path, {
    assistant_id: ASSISTANT_ID,
    input: { messages: [{ role: 'user', content: prompt }] },
  }, 'run_agent');
}

/**
 * Extract the last assistant message from a messages-based final state.
 * Change this method for other graph output shapes.
 * @param {object} state - Graph final state.
 * @returns {string} Assistant text or empty string.
 */
function lastAssistantText(state) {
  const messages = state && Array.isArray(state.messages)
    ? state.messages : [];
  for (let i = messages.length - 1; i >= 0; i--) {
    const m = messages[i];
    if (!m || (m.type !== 'ai' && m.role !== 'assistant')) continue;
    if (typeof m.content === 'string') return m.content.trim();
    if (Array.isArray(m.content)) {
      return m.content
        .filter((p) => p && p.type === 'text' && typeof p.text === 'string')
        .map((p) => p.text).join('\n').trim();
    }
  }
  return '';
}

/**
 * Delete a disposable test thread after measuring the user transaction.
 * @param {string|null} threadId - Thread to clean up.
 */
function cleanupThread(threadId) {
  if (!CLEANUP || !threadId) return;
  const response = http.del(
    BASE_URL + '/threads/' + encodeURIComponent(threadId), null,
    { headers, timeout: '30s', tags: { name: 'delete_thread' } },
  );
  check(response, {
    'cleanup HTTP 2xx': (r) => r.status >= 200 && r.status < 300,
  });
}

/**
 * Measure and record a whole transaction, including correctness.
 * @param {string} kind - Scenario label.
 * @param {function():void} action - Transaction body.
 */
function measureWorkflow(kind, action) {
  const started = Date.now();
  let success = false;
  try {
    action();
    success = true;
  } catch (e) {
    console.error(kind + ' failed: ' + String(e));
  } finally {
    workflowDuration.add(Date.now() - started, { kind });
    workflowSuccess.add(success, { kind });
  }
}

/** Test a single stateless run with an exact expected response. */
export function single() {
  measureWorkflow('single', () => {
    const answer = lastAssistantText(
      runAgent(null, 'Reply with exactly K6_OK and no other text.'));
    if (answer !== 'K6_OK') {
      throw new Error('Unexpected answer: ' + JSON.stringify(answer));
    }
  });
}

/** Verify a two-turn conversation has retained thread-local state. */
export function conversation() {
  let threadId = null;
  try {
    measureWorkflow('conversation', () => {
      threadId = createThread();
      const reference = 'K6REF_' + exec.vu.idInTest + '_' +
        exec.vu.iterationInScenario + '_' + Date.now();
      const ack = lastAssistantText(runAgent(
        threadId, 'Remember this reference: ' + reference + '. Reply ACK.'));
      if (!ack) throw new Error('Empty first turn');
      const answer = lastAssistantText(runAgent(
        threadId, 'What reference did I tell you? Reply with only that value.'));
      if (answer !== reference) {
        throw new Error('State not preserved: ' + JSON.stringify(answer));
      }
    });
  } finally {
    cleanupThread(threadId);
  }
}

/** Execute a tool-capable graph task and validate hidden expected output. */
export function agentic() {
  let threadId = null;
  try {
    measureWorkflow('agentic', () => {
      if (!TASK || !EXPECTED) {
        throw new Error('Set AGENT_TASK and AGENT_EXPECT');
      }
      threadId = createThread();
      const answer = lastAssistantText(runAgent(threadId, TASK));
      if (!answer.includes(EXPECTED)) {
        throw new Error('Fixture assertion failed: ' + JSON.stringify(answer));
      }
    });
  } finally {
    cleanupThread(threadId);
  }
}
```

Run individually (requires a reachable compatible server and a real configured agent):

```bash
k6 run --scenario single \
  -e BASE_URL="$AGENT_SERVER_URL" -e ASSISTANT_ID="$ASSISTANT_ID" \
  langgraph-k6.js

k6 run --scenario conversation \
  -e BASE_URL="$AGENT_SERVER_URL" -e ASSISTANT_ID="$ASSISTANT_ID" \
  -e CLEANUP=true langgraph-k6.js
```

For a tool test, seed a **disposable** workspace with a `k6-fixture.txt` file containing `K6_FIXTURE_OK`, without placing that string in the agent prompt. Instruct the agent to use the filesystem tool to read that file. Run:

```bash
k6 run --scenario agentic \
  -e BASE_URL="$AGENT_SERVER_URL" \
  -e ASSISTANT_ID="$ASSISTANT_ID" \
  -e AGENT_TASK="Read k6-fixture.txt with the filesystem tool and reply with its contents." \
  -e AGENT_EXPECT="K6_FIXTURE_OK" \
  -e CLEANUP=true \
  langgraph-k6.js
```

And sustained concurrency:

```bash
k6 run --scenario load \
  -e BASE_URL="$AGENT_SERVER_URL" \
  -e ASSISTANT_ID="$ASSISTANT_ID" \
  -e AGENT_TASK="$AGENT_TASK" -e AGENT_EXPECT="$AGENT_EXPECT" \
  -e VUS=20 -e DURATION=5m -e CLEANUP=true \
  langgraph-k6.js
```

**Caveats:**
- `--scenario` is supported in current k6, but pin an actual tested release before rollout.
- The script uses synchronous `/runs/wait` and a ten-minute HTTP timeout; long or background runs may need submit + poll + explicit terminal-state handling.
- The fixture string being present in a final answer does **not** prove the tool was invoked. Inspect spans/events and the resulting file state.
- A completed agent run can be internally wrong despite returning HTTP 2xx. Separate transport, run, and semantic correctness.
- Per-run thread creation/deletion is included for isolation. Decide whether their overhead belongs to the user transaction and measure it separately.
- With `constant-vus`, a VU starts the next task when its previous task finishes. This is not a fixed requests-per-second load.
- In synthetic multi-turn tests, repeated use of the same prefix can artificially improve cache hit rate. Mix datasets and explicit cold/warm cache cases.

## 8. Streaming benchmarks (SSE and WebSockets)

For gateway/vLLM and streaming agent endpoints, benchmark **actual stream consumption**, not a buffered HTTP request.

### Metrics and event semantics

- **TTFB:** first HTTP response bytes (not necessarily a token).
- **Time to first SSE event:** may be metadata, heartbeat, tool status, or a token.
- **TTFT:** first **meaningful model-generated content token**, measured at the client. Define how empty content/reasoning chunks are handled.
- **ITL:** inter-arrival time of generated content tokens/chunks. A chunk is **not always a token**, so do not label raw chunk gaps TPOT without token accounting.
- **TPOT:** generation time divided by number of generated tokens, using consistent tokenization/usage accounting.
- **End-to-end completion latency:** time until terminal success/failure and final content are received.
- **Stream interruption/cancellation:** abruptly disconnected clients and graceful cancellations, with backend resource-release time.

Candidate: [phymbert/xk6-sse](https://github.com/phymbert/xk6-sse). Its published examples include POST-based OpenAI-compatible chat-completion streams and first-event timing. It is **community-maintained**, not equivalent to an officially guaranteed protocol implementation. Pin its version, test it against the actual k6 release, and audit its behavior for POST body, event handlers, timeouts, errors, automatic extension resolution and connection cleanup.

For LangGraph agent streams, parse the selected stream mode (`messages` / `updates` / `values`, as configured), correlate run ID, node/tool events and completion, and distinguish graph progress events from LLM token events. If the Portal streams over WebSockets, use the k6 WebSocket API and validate application-level completion/correlation.

A high-level stream test should:

1. open the stream with unique run ID;
2. record request start and first response/event timestamps;
3. ignore non-content metadata for TTFT;
4. count content chunks and verify usage if emitted;
5. record terminal result, errors, tool events and cancellation;
6. close/abort the stream, and verify server-side run and GPU activity eventually stop.

Do not calculate token throughput by assuming one SSE event equals one token.

## 9. Model realistic traffic correctly

### Closed workload: active executions

Use `constant-vus` or `ramping-vus` for 1, 5, 10, 20, 30, 50, 75, 100, 150, 200, 300 VUs as exploratory levels. A VU that waits synchronously for a graph run typically corresponds to one active workflow. A portal user might instead spend most of their session reading results, so add think time between turns where appropriate.

### Open workload: arrival of new tasks

Use `constant-arrival-rate` (or `ramping-arrival-rate`) to launch a target number of new workflows per second independent of service latency:

```javascript
export const options = {
  scenarios: {
    new_agent_tasks: {
      executor: 'constant-arrival-rate',
      rate: 2,
      timeUnit: '1s',
      duration: '5m',
      preAllocatedVUs: 100,
      maxVUs: 300,
      exec: 'agentic',
    },
  },
};
```

This is a **scenario configuration fragment** that can replace the scenarios object in the complete LangGraph script. Ensure the generator has enough VUs; report `dropped_iterations`, or the offered load was not achieved. Do not add a sleep at the end of each arrival-rate iteration.

**Capacity planning:** measure actual active generations, not only frontend sessions. Useful derived metrics:

- `LLM requests per agent run` = total downstream inference calls / completed agent workflows.
- `Tool calls per run`, `subagent fan-out`, `average active generations per run`.
- `Offered task rate` vs `completed task rate`.
- `Successful workflows per minute` and associated end-to-end p50/p95/p99.
- `System throughput` in generated tokens/second per GPU and aggregate.
- `Queueing vs processing` at the gateway, agent worker and vLLM level.

A supervisor launching several children may multiply peak model concurrency without multiplying user sessions.

## 10. Instrumentation: measure where time is spent

### Proposed OpenTelemetry span hierarchy

```text
portal.request
  -> auth/session
  -> agent.run (run_id, agent_kind, assistant_name)
     -> checkpoint.read
     -> langgraph.node (node_name)
        -> llm.request (route/model/replica)
           -> gateway.queue
           -> vllm.generate
        -> tool.invoke (tool_name, executor_location)
           -> microvm.allocate / microvm.execute / artifact.write
     -> subagent.invoke (child agent/run ID)
     -> checkpoint.write
  -> response.stream / response.complete
```

Carry trace context across HTTP/gRPC, asynchronous workers and tool execution wherever feasible. Use trace/run IDs in logs and exemplars; **do not** attach unique user/thread/run IDs as Prometheus labels, because they create unbounded metric cardinality. Limit content logging, redact credentials, and do not publish user prompt or file content in telemetry.

### Suggested metrics (to implement, not guaranteed built-ins)

| Metric | Type | Dimensions (bounded) |
| --- | --- | --- |
| `agent_run_duration_seconds` | Histogram | agent kind, status |
| `agent_active_runs` | Gauge | agent kind |
| `agent_queue_wait_seconds` | Histogram | queue |
| `agent_run_outcomes_total` | Counter | status, failure class |
| `agent_node_duration_seconds` | Histogram | node name, graph |
| `agent_tool_calls_total` | Counter | tool, status |
| `agent_tool_duration_seconds` | Histogram | tool, executor |
| `agent_tool_errors_total` | Counter | tool, error class |
| `agent_llm_calls_total` | Counter | model route, status |
| `agent_subagent_calls_total` | Counter | child agent kind |
| `agent_stream_first_content_seconds` | Histogram | agent kind |
| `agent_checkpoint_duration_seconds` | Histogram | operation |
| `sandbox_startup_seconds` | Histogram | worker class |
| `sandbox_execution_seconds` | Histogram | worker class |
| `agent_cancellations_total` | Counter | reason |

Correlate them with NGINX/request metrics, existing vLLM exported metrics (queue, running/waiting requests, KV cache, request and token counters as supported by pinned version), node CPU/RAM, and NVIDIA GPU utilization/VRAM/power. For routing, compare per-replica distribution and prefix-cache behavior, including the previously observed uneven utilization, without assuming new router policies are already deployed.

Keep three latency components distinct: client/perceived latency, time queued before execution, and actual inference/tool processing time. An answer that takes 90 seconds requires backend spans to explain why.

## 11. Testing correctness of agentic flows

Do not rely on free-form LLM responses alone. Build a **scenario corpus** with stable fixtures and automated oracles.

| Scenario | Setup | Oracle |
| --- | --- | --- |
| Exact simple response | deterministic prompt | exact text or schema |
| Conversation retention | unique reference in first turn | second turn returns correct value |
| Filesystem read | seeded file inaccessible through prompt | relevant tool span and exact content |
| File write | temporary workspace | hash/contents of output file |
| Code change | pinned miniature repository | expected diff + passing tests |
| Search and retrieval | controlled local web fixture or stub | source provenance and retrieved item |
| Subagents | registered child tools with counters | expected invocations and outputs |
| Failure/retry | tool throws once | bounded retry, single side effect |
| User approval | pause before dangerous operation | no side effect before approval |
| Cancellation | cancel active run/tool | timely stop and cleanup |
| Memory and permission boundary | two isolated users/threads | no cross-user leakage |
| Resume/recovery | restart worker after checkpoint | no duplicated non-idempotent action |

Use both **mocked** deterministic tool/model backends to test orchestration and **real** vLLM inference to measure deployment performance. Maintain separate suites for semantic quality, trace structure, latency and resilience. Nondeterministic LLMs require repeated sampling and pass-rate distributions rather than treating one answer as definitive.

For coding agents, validate actual repository changes, not just their descriptions of changes. Test command execution only in an isolated disposable workspace. For idempotency, simulate request retries around shell and file writes.

## 12. Sandbox and deployment safety

The currently documented agent execution is backend-hosted; microVM isolation is a planned architecture, not assumed available today. Therefore:

- First run destructive tool tests against **disposable, strictly permissioned test resources**, never arbitrary host directories or production secrets.
- Once microVMs are available, assign a fresh filesystem/workspace per test run and a bounded CPU/RAM/process/time budget. Snapshot or discard after each test.
- Use synthetic user accounts, sanitized prompts and locally seeded fixture repositories.
- Deny real external writes, emails, pull-request creation and non-test deployments unless specifically targeted in a safe isolated test.
- For web tools, route through your own services and corporate proxy as specified in [portal tool architecture](portal_agent_tool_execution_architecture.md); avoid unintentionally load-testing public sites or search engines.
- Pin test target environments and service versions. Keep soak/stress testing outside normal production hours and define a stop mechanism if availability degrades.
- Put k6 load generation on a separate CPU host, not on the H100/H200 inference nodes, to prevent the generator from contaminating results.
- Use dedicated test rate-limit credentials with scoped permissions. Keep tokens in environment/secrets stores, never committed scripts.
- Validate that the generator itself has enough CPU, memory, file descriptors, ephemeral ports and network throughput.

## 13. Self-hosted metrics and dashboards

A practical first stack is **k6 + Prometheus + Grafana**, with backend instrumentation exported through the existing OpenTelemetry stack. Grafana Cloud is unnecessary. Prometheus remote write ingestion must be enabled on the chosen receiver.

Example:

```bash
K6_PROMETHEUS_RW_SERVER_URL="http://prometheus.internal:9090/api/v1/write" \
K6_PROMETHEUS_RW_TREND_STATS="p(50),p(95),p(99),min,max" \
k6 run -o experimental-prometheus-rw \
  --scenario load \
  -e BASE_URL="$AGENT_SERVER_URL" \
  -e ASSISTANT_ID="$ASSISTANT_ID" \
  -e AGENT_TASK="$AGENT_TASK" -e AGENT_EXPECT="$AGENT_EXPECT" \
  -e VUS=20 -e DURATION=5m \
  --tag testid=agent-load-baseline \
  langgraph-k6.js
```

As of the documentation reviewed, `experimental-prometheus-rw` remains an **experimental** k6 output. Pin versions and test exporter compatibility. Grafana dashboards should plot at least:

1. workflow starts/completions, success and p50/p95/p99 duration;
2. gateway RPS/latency/errors, per-route traffic and per-replica queues;
3. TTFT, token throughput, stream errors and inter-token gaps;
4. active graph runs, queues, node/tool/subagent timing;
5. vLLM running/waiting requests, KV-cache and GPU statistics;
6. saturation curves, dropped iterations, cancellations and recovery times.

Test tags should identify the benchmark dataset, deployment revision, model build, workload kind and route without high-cardinality IDs.

## 14. Controlled experiment plan

### Phase 0: instrument and validate one request

- Create disposable account/workspace and a fixed fixture dataset.
- Observe one request end-to-end, with matching run, trace, tool and gateway IDs.
- Confirm the difference between HTTP completion, agent completion and semantic success.
- Check that metrics do not store sensitive prompt content.

### Phase 1: gateway isolation

- Replace inference with a fast mock OpenAI-compatible backend.
- Compare direct mock latency against gateway latency.
- Test authentication, paths, errors, timeouts, reconnection, routing, and a fixed RPS range.
- Record gateway-induced latency and resources separately from model latency.

### Phase 2: real inference (direct vs routed)

- Benchmark each vLLM replica directly and through the gateway with identical prompts.
- Test short/medium/long contexts and output lengths; warmed and cold prefixes.
- Compare cache-aware placement against simpler routing when both are deployed.
- Examine H100/H200 imbalance, queueing, TTFT and aggregate throughput.

### Phase 3: real LangGraph

- Run `single`, `conversation` and `agentic` from the included example.
- Validate tools with traces and artifacts; include checkpoints and subagent execution.
- Progress from 1 to 5, 10, 20, 30, 50, 75 and 100 concurrent runs, extending further only if meaningful and safe.

### Phase 4: full Portal sessions and resilience

- Exercise authentication, UI chat creation, streaming, cancel, resume and approval.
- Add `k6/browser` for a small number of real UI sessions under protocol-level load.
- Inject network disconnects, tool timeouts, worker restarts and backend errors.
- Run longer steady-state soak tests to catch memory leaks, orphaned runs, stuck streams and leaked workspaces.

### Phase 5: performance regression gates

- Store fixed workload definitions alongside representative fixture hashes.
- Collect baseline distributions after warm-up across repeated runs.
- Fail CI on a statistically meaningful regression in p95 workflow time, error rate, or task success; avoid arbitrary thresholds before empirical baseline.
- Compare baseline vs candidate versions of gateway routing, model quantization, prefix-cache policy and orchestration.

A sensible first comparison: **single request directly to vLLM -> same request through gateway -> single LangGraph run -> tool agent -> 20 concurrent agent runs**. This locates bottlenecks incrementally without conflating the entire stack.

## 15. Acceptance criteria for a proof of concept

- [ ] All test infrastructure and observability are self-hosted; no remote MCP or cloud load-testing service.
- [ ] Both a generic gateway request and a real LangGraph execution run reproducibly.
- [ ] Multi-turn state is verified using independent, uniquely identified threads.
- [ ] At least one tool-enabled agent workflow is validated with a hidden fixture and actual trace/artifact checks.
- [ ] Agent runs are measured end-to-end, rather than marking submission as completion.
- [ ] TTFT and streaming behavior are measured using a streaming-capable client, separately from buffered HTTP response duration.
- [ ] Backend metrics/spans permit identification of graph, tool, gateway and vLLM bottlenecks.
- [ ] Separate mocked and real-inference baselines exist.
- [ ] Workloads can run at 1, 5 and 20 concurrent active workflows with zero shared mutable test state.
- [ ] Cancellation, timeout and error paths release resources and do not produce duplicate side effects.
- [ ] Grafana panels correlate k6 results with backend queue length, model performance and GPU metrics.
- [ ] Soak and stress runs are bounded, repeatable, safe for test infrastructure, and leave no uncollected workspaces.

## 16. Sources and further reading

- Grafana k6 scenarios: https://grafana.com/docs/k6/latest/using-k6/scenarios/
- Scenario selection (`--scenario`): https://grafana.com/docs/k6/latest/using-k6/scenarios/
- k6 executors: https://grafana.com/docs/k6/latest/using-k6/scenarios/executors/
- Constant arrival rate: https://grafana.com/docs/k6/latest/using-k6/scenarios/executors/constant-arrival-rate/
- k6 browser: https://grafana.com/docs/k6/latest/using-k6-browser/
- Prometheus remote write: https://grafana.com/docs/k6/latest/results-output/real-time/prometheus-remote-write/
- Community SSE extension: https://github.com/phymbert/xk6-sse
- LangGraph/LangSmith Agent Server conceptual documentation: https://docs.langchain.com/langsmith/data-plane
- LangGraph assistants and API usage: https://docs.langchain.com/langsmith/configuration-cloud

**Implementation note:** This Markdown records a proposal and illustrative, fully written scripts. The scripts have not been executed against the actual portal/Agent Server in this repository. Update request/response adapters for your deployed API, and do not treat proposed metric names, microVM features or routing policies as already implemented.
