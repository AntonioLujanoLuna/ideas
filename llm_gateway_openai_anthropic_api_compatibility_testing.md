# LLM Gateway: OpenAI and Anthropic API Compatibility and Conformance Testing

**Status:** Proposal and implementation plan; not yet executed against the gateway  
**Date:** 2026-11-13  
**Scope:** OpenAI-compatible and Anthropic-compatible `/v1` gateway APIs, LiteLLM client compatibility, official SDK interoperability, contract validation, streaming, tool use, and integration testing.  
**Target deployment:** Self-hosted LLM gateway / NGINX / vLLM on H100 and H200 nodes.  
**Related proposals:** [Agent Portal k6 load and workflow testing](agent_portal_k6_load_and_workflow_testing.md), [llm-d cache-aware routing](llm_d_cache_aware_vllm_routing.md), [path-multiplexed replicas](llm_d_path_multiplexed_vllm_replicas.md).

## 1. Executive recommendation

**Yes, use LiteLLM to test the gateway, but do not treat passing LiteLLM calls as proof of full OpenAI or Anthropic API compliance.** Build three complementary layers:

1. **Consumer interoperability:** LiteLLM Python SDK plus *official* OpenAI and Anthropic SDKs calling the gateway. Include representative clients that the portal actually uses.
2. **Protocol and API contract:** Raw HTTP/SSE tests, provider-specific schemas, required headers, validation errors, tool-calling and structured outputs. Compare against a **pinned** upstream API specification, not the latest unpinned version.
3. **End-to-end gateway behavior:** Authentication, routing, backend capability handling, retries, timeouts, request cancellation, usage accounting and concurrency, with both mock upstreams and real vLLM replicas.

These are not substitutes for one another. LiteLLM often normalizes provider differences and can translate between APIs. In particular, success when LiteLLM maps a Responses API request to Chat Completions is **not** evidence that the gateway has implemented native `POST /v1/responses`. A green test must prove which HTTP route, headers and payload the gateway actually received.

**Desired deliverable:** A CI-generated compatibility matrix of `PASS / FAIL / UNSUPPORTED / UNTESTED` at the *endpoint × feature × client × backend* level. Declare precisely which supported subset of each API the gateway implements. Do not claim exhaustive conformance to every commercial-platform feature unless it has been implemented and tested.

## 2. Scope, topology, and constraints

Current design context:

- Gateway in front of self-hosted vLLM replicas, with NGINX as reverse proxy and a single externally exposed port (9080 in the current deployment design).
- H100 and H200 GPU workers; multiple independent replicas, potentially selected through llm-d in a later routing iteration.
- Agentic clients with long context (target 262k), streaming, tool calls and reasoning-related parameters.
- The inference server and model may lack capabilities that exist in OpenAI's or Anthropic's hosted platform.

Keep **wire-protocol conformance** separate from **model capabilities** and **upstream platform features**. For example, a refusal to generate audio because the deployed model does not support audio is not automatically a gateway bug; returning a misleading successful audio response or an invalid provider-shaped error is a bug.

Do not require any remotely hosted MCP servers or hosted test infrastructure. Client SDKs, pytest, contract tools and mock providers can run on organization-controlled hosts. Normal remote package/documentation retrieval during development is distinct from requiring external serving infrastructure.

### 2.1 Multiple provider interfaces on one hostname

There is a fundamental route collision: both OpenAI and Anthropic use `GET /v1/models`, but have different payload contracts. A single gateway cannot return both shapes unconditionally.

Choose an explicit dispatch policy, for example:

- **Preferred:** Separate, clearly documented base URLs, such as `https://gateway.example/openai/v1` and `https://gateway.example/anthropic`, reverse-proxied to provider-specific adapters.
- **Alternative:** The same route with deterministic discrimination based on `anthropic-version` and the relevant Anthropic authentication header, versus OpenAI Bearer authorization. Make ambiguity an explicit error rather than silently guessing.
- **Verification:** Call the model-list endpoint with **both official SDKs** and validate each provider's precise schema, pagination, model IDs and status codes.

These URLs are architectural examples, not assertions about the gateway's existing routes. Anthropic's Python SDK `base_url` is the API root and appends `/v1/messages`; OpenAI's Python SDK `base_url` normally points to the `/v1` root. Test the exact path used by each SDK.

## 3. Endpoint inventory and support levels

Treat this table as a **proposed coverage inventory**, not an assertion that each operation exists or must be exposed. Mark unsupported entries explicitly and document fallback behavior.

| Interface | Method / path | Priority | Notes |
|---|---|---|---|
| OpenAI | `GET /v1/models` | P0 | List response, model IDs, provider conflict |
| OpenAI | `GET /v1/models/{model}` | P1 | Correct model lookup and 404 behavior |
| OpenAI | `POST /v1/chat/completions` | P0 | Non-stream, stream, tools, JSON mode, usage |
| OpenAI | `POST /v1/responses` | P0 | **Native** Responses shape and SSE events; no implicit proxy translation claim |
| OpenAI | `GET /v1/responses/{response_id}` | Conditional | Only when storing/retrieving responses is supported |
| OpenAI | `POST /v1/responses/{response_id}/cancel` | Conditional | Relevant for background responses, not an assumed mandatory operation |
| OpenAI | `DELETE /v1/responses/{response_id}` | Conditional | Only when response persistence is implemented |
| OpenAI | `POST /v1/completions` | P2 | Legacy text-completion clients |
| OpenAI | `POST /v1/embeddings` | Conditional | Only if an embedding model is deployed |
| OpenAI | Other files, batches, images, audio, etc. | Out of initial scope | Add only if product requirements justify them |
| Anthropic | `POST /v1/messages` | P0 | Native Anthropic request/response semantics |
| Anthropic | `POST /v1/messages/count_tokens` | P1 | Token-counting semantics, not naive string-length heuristics |
| Anthropic | `GET /v1/models` | P1 | Anthropic schema and cursor pagination |
| Anthropic | `GET /v1/models/{model_id}` | P1 | Model metadata and missing model behavior |
| Anthropic | `/v1/messages/batches` operations | Conditional | Submit, retrieve, list, cancel, results, only if supported |

**P0** means needed to claim basic agentic-client compatibility; **P1** means highly useful; **P2** is optional/legacy. Confirm all paths against the exact upstream specification revision selected for implementation. Neither API is static.

A comprehensive test inventory must be generated from the declared support manifest: never equate "all routes that happened to be tested" with "all routes exposed by the gateway."

### Suggested capability manifest

Maintain a version-controlled YAML manifest so tests do not need to infer features from live model responses:

```yaml
schema_version: 1
interfaces:
  openai:
    endpoints:
      - "GET /v1/models"
      - "GET /v1/models/{model}"
      - "POST /v1/chat/completions"
      - "POST /v1/responses"
    features:
      chat_streaming: supported
      responses_streaming: supported
      tool_calls: supported
      structured_output: conditional
      images: conditional
      background_responses: unsupported
  anthropic:
    endpoints:
      - "GET /v1/models"
      - "POST /v1/messages"
      - "POST /v1/messages/count_tokens"
    features:
      messages_streaming: supported
      tool_use: supported
      extended_thinking: conditional
      images: conditional
backends:
  modelA:
    provider_model_id: "modelA"
    tool_calls: conditional
    vision: conditional
```

**Illustrative only:** Replace every value with measured and implemented support. A manifest is the source of truth for test selection, not an excuse to skip required behavior silently. Include gateway version, model family, vLLM version and spec revision in generated reports.

## 4. Three complementary test layers

### 4.1 LiteLLM as a consumer

Use the Python LiteLLM client to execute representative calls:

- Basic chat, system prompts, multi-turn history and token limits.
- Streaming with an actual iteration over the response.
- Tool schemas, parallel calls, tool-result continuations and unusual arguments.
- Error propagation and usage fields.
- Different model IDs routed through the gateway.

Use the explicit `openai/` provider prefix and `api_base` when testing OpenAI-compatible calls. Do **not** confuse LiteLLM's OpenAI-compatible proxy *server* with its SDK client: running an extra LiteLLM proxy in front of your gateway can hide discrepancies behind another translation layer.

**Required negative control:** Capture raw outgoing LiteLLM requests using a mock HTTP server or trace logs to establish which API endpoint and wire format it really used. Any LiteLLM-specific normalization or substitution must be recorded in the test report.

### 4.2 Official SDK interoperability

Run the official OpenAI and Anthropic Python SDKs directly against the gateway. Test typical operations using native methods:

- `OpenAI(...).models.list()`, `chat.completions.create(...)`, `responses.create(...)`.
- `Anthropic(...).models.list()`, `messages.create(...)`, `messages.count_tokens(...)`.
- SDK streaming helpers, typed object parsing, proper errors, retries disabled in deterministic tests.
- SDK method behavior, not only successful response text.

Pin SDK versions in CI. The official SDKs are **consumer compatibility checks**, not proof of the entire HTTP contract: SDKs may be permissive about some missing fields, and their supported method surface may lag behind or differ from the specification revision.

### 4.3 Raw HTTP, SSE and schema conformance

Use `httpx` and pytest for exact request/response assertions, with Schemathesis or a comparable OpenAPI tool for positive/negative schema-driven cases.

- Assert status code, content type, mandatory headers, exact JSON object shape and value types.
- Reject malformed JSON, absent required arguments, incompatible types, unknown models, invalid tool schemas, illegal parameter combinations and excessive token limits appropriately.
- Check OpenAI and Anthropic error bodies separately, including authentication, throttling and upstream failure mapping.
- Explicitly test request content semantics (e.g. multimodal blocks, roles, tool results), not merely schema validation.
- Test the same route through actual ingress and, where possible, an isolated provider adapter to distinguish NGINX/routing failures from implementation errors.

Property-based testing should use an **allowlisted supported endpoint subset**. Upstream OpenAPI definitions may contain endpoints that intentionally do not exist on the gateway, and the provider docs may not supply one complete usable machine-readable schema for all Anthropic behaviors. Build and maintain custom test schemas where necessary.

## 5. Critical behavioral contract tests

### 5.1 Streaming is not merely "returns SSE"

**OpenAI Chat Completions**

- HTTP 200 with `text/event-stream` when streaming.
- Each non-terminal `data:` payload parses as a chat completion chunk; IDs/model/choices/finish reasons consistent.
- Delta role/content/tool-call fragments arrive with the correct association and ordering.
- Stream ends with the appropriate `[DONE]` terminator for Chat Completions.
- Usage in the terminal chunk is handled correctly if requested and supported.

**OpenAI Responses**

- Provider-specific typed events, including creation, output-item/content deltas and a terminal completion or failure event.
- Do not substitute the Chat Completions stream format or assert that Responses must terminate with Chat's `[DONE]` sentinel.
- Correct output item IDs, function-call arguments, partial JSON handling, reasoning-related items (if supported), and terminal usage/status.

**Anthropic Messages**

- Parse event names and JSON data together. Expected progression can include `message_start`, `content_block_start`, `content_block_delta`, `content_block_stop`, `message_delta`, `message_stop`.
- Validate indices, block IDs, text deltas, `input_json_delta` for tool inputs, `stop_reason`, and final usage.
- Account for `ping` events and documented stream errors; do not assume every event is content.
- A stream transport ending without a valid terminal event should fail the test.

For every protocol: test 1-token output, large output, tool output, partial delivery, slow consumer, early client disconnect, midstream backend error and malformed upstream SSE. Verify cleanup and accounting after cancellation.

### 5.2 Tools and agentic loops

Use a deterministic fixture involving at least one synthetic tool:

1. User requests a task that requires the tool.
2. Model emits provider-specific tool invocation(s); validate ID, name, arguments, format and stop/finish reason.
3. Client returns tool result in the appropriate provider-native message format.
4. Model continues and produces final content with the correct conversation state.

Include zero, one and multiple tool calls; sequential versus parallel tool use; duplicate call identifiers; JSON fragments split across stream chunks; a tool error; and a tool-result message arriving in an invalid position. Report whether behavior relies on model ability or gateway implementation.

### 5.3 Other feature suites

| Suite | Required checks |
|---|---|
| Structured outputs | Valid JSON, JSON schema constraints, unsupported-schema errors, streaming fragments |
| Reasoning / thinking | Parameter parsing, reasoning tokens or content blocks if exposed, no leakage of internal fields, proper usage |
| Multimodal | Supported image representations, image+text mix, unsupported input handling, size limits |
| Token counting | Actual serving tokenizer/template, system prompts, tools, multimodal input accounting and documented approximations |
| Errors | 400, 401, 403, 404, 413/422 where relevant, 429, 5xx, provider-shaped error body and trace IDs |
| Authorization | Bearer versus `x-api-key`, tenant/model permissions, reject missing/invalid keys, no secret logs |
| Reliability | Timeout, health transition, retry safety, over-capacity response, disconnect cleanup |
| Routing | Requested model maps to allowed pool; no silent cross-model substitution; fair backend selection |
| Usage | Input, output, cached and reasoning tokens only where meaningful; no double count on retries |
| Observability | Request ID, model ID, backend, latency, token count, terminal status and error category |

For 262k-context workloads, test boundaries in **tokens**, not just characters, and run near-limit tests in a dedicated resource budget rather than on every pull request.

## 6. Executable smoke-test starting point

This intentionally small suite checks that both official SDKs and LiteLLM can call the gateway. It is **not full conformance**. Place it in a test project as `test_gateway_compat.py`.

Install packages:

```bash
uv add --dev pytest litellm openai anthropic httpx
```

Environment variables:

- `GW_OPENAI_BASE_URL`: OpenAI API base ending in `/v1` (or the configured provider-specific equivalent).
- `GW_ANTHROPIC_BASE_URL`: Anthropic API root, **without** `/v1`.
- `GW_API_KEY`: Gateway test credential. Use a non-production account.
- `GW_MODEL`: A model ID available via both interfaces; use separate fixture parametrization if exposed IDs differ.

```python
import os

import httpx
import litellm
from anthropic import Anthropic
from openai import OpenAI


OPENAI_BASE = os.environ["GW_OPENAI_BASE_URL"].rstrip("/")
ANTHROPIC_BASE = os.environ["GW_ANTHROPIC_BASE_URL"].rstrip("/")
API_KEY = os.environ["GW_API_KEY"]
MODEL = os.environ["GW_MODEL"]


def test_litellm_chat() -> None:
    """Call the gateway with LiteLLM's OpenAI-compatible client."""
    response = litellm.completion(
        model=f"openai/{MODEL}",
        api_base=OPENAI_BASE,
        api_key=API_KEY,
        messages=[{"role": "user", "content": "Say hello."}],
        max_tokens=64,
    )
    assert response.choices
    assert response.choices[0].message.role == "assistant"


def test_openai_chat() -> None:
    """Confirm basic OpenAI SDK Chat Completions interoperability."""
    with OpenAI(base_url=OPENAI_BASE, api_key=API_KEY, max_retries=0) as client:
        result = client.chat.completions.create(
            model=MODEL,
            messages=[{"role": "user", "content": "Say hello."}],
            max_tokens=64,
        )
    assert result.object == "chat.completion"
    assert result.choices
    assert result.usage is not None


def test_openai_responses() -> None:
    """Check the gateway's native Responses API, not LiteLLM translation."""
    with OpenAI(base_url=OPENAI_BASE, api_key=API_KEY, max_retries=0) as client:
        result = client.responses.create(
            model=MODEL,
            input="Say hello.",
            max_output_tokens=64,
        )
    assert result.object == "response"
    assert result.id
    assert result.status == "completed"


def test_anthropic_messages() -> None:
    """Confirm Anthropic SDK Messages API interoperability."""
    with Anthropic(base_url=ANTHROPIC_BASE, api_key=API_KEY, max_retries=0) as client:
        result = client.messages.create(
            model=MODEL,
            max_tokens=64,
            messages=[{"role": "user", "content": "Say hello."}],
        )
    assert result.type == "message"
    assert result.role == "assistant"
    assert result.content
    assert result.usage.input_tokens >= 0


def test_openai_chat_stream() -> None:
    """Inspect raw Chat Completions SSE and terminal marker."""
    with httpx.Client(timeout=90) as client:
        with client.stream(
            "POST",
            f"{OPENAI_BASE}/chat/completions",
            headers={"Authorization": f"Bearer {API_KEY}"},
            json={
                "model": MODEL,
                "messages": [{"role": "user", "content": "Say hello."}],
                "max_tokens": 64,
                "stream": True,
            },
        ) as response:
            assert response.status_code == 200
            assert "text/event-stream" in response.headers.get("content-type", "")
            chunks = [
                line[5:].strip()
                for line in response.iter_lines()
                if line.startswith("data:")
            ]
    assert chunks
    assert chunks[-1] == "[DONE]"


def test_anthropic_stream() -> None:
    """Inspect the minimal native Anthropic SSE event sequence."""
    with httpx.Client(timeout=90) as client:
        with client.stream(
            "POST",
            f"{ANTHROPIC_BASE}/v1/messages",
            headers={
                "x-api-key": API_KEY,
                "anthropic-version": "2023-06-01",
            },
            json={
                "model": MODEL,
                "max_tokens": 64,
                "messages": [{"role": "user", "content": "Say hello."}],
                "stream": True,
            },
        ) as response:
            assert response.status_code == 200
            assert "text/event-stream" in response.headers.get("content-type", "")
            events = [
                line[6:].strip()
                for line in response.iter_lines()
                if line.startswith("event:")
            ]
    assert "message_start" in events
    assert "message_stop" in events
```

Run:

```bash
uv run pytest -v test_gateway_compat.py
```

**Limitations of the smoke suite:** It makes real inference calls, uses a single shared model identifier, expects the gateway to support all tested operations, only samples event *presence* rather than parsing SSE payloads, does not test provider-specific model-list routes, and cannot distinguish gateway faults from upstream model failures. Replace these assumptions with fixtures, support-manifest skips and complete protocol assertions in the main suite.

## 7. Contract testing tools

### 7.1 Provider specifications

- OpenAI's machine-readable API description: https://github.com/openai/openai-openapi
- OpenAI API reference: https://platform.openai.com/docs/api-reference
- Anthropic API documentation: https://docs.anthropic.com/en/api/overview
- Anthropic Messages and streaming documentation: https://docs.anthropic.com/en/api/messages

**Pin the provider specification revision and date.** Record which endpoints are in scope and maintain localized provider-specific contract schemas wherever upstream source materials are incomplete or incompatible with automated schema validation. The upstream spec covers a larger product than an inference-only vLLM gateway.

### 7.2 Schemathesis and pytest

- Schemathesis: https://github.com/schemathesis/schemathesis
- pytest: https://docs.pytest.org/
- httpx: https://www.python-httpx.org/
- LiteLLM: https://docs.litellm.ai/

Use Schemathesis for structured fuzzing and negative input generation against a curated OpenAPI definition of **this gateway's intended supported surface**. Use handwritten pytest tests for SSE event sequences, dynamic tool calls, cancellation, model routing, response-state operations and headers because those behaviors are not captured by JSON schemas alone.

Never send unbounded property-based fuzz traffic or 262k-token cases into shared production GPUs. Use a fake deterministic inference upstream for the majority of API contract tests.

## 8. Test environment and CI architecture

```text
                  CI (pytest + Schemathesis + report generator)
                                     |
                    +----------------+----------------+
                    |                                 |
             Official SDKs                        LiteLLM SDK
             OpenAI/Anthropic                         |
                    +----------------+----------------+
                                     |
                         raw HTTP/SSE checks
                                     |
                        Gateway / NGINX ingress
                                     |
                          Auth / provider adapters
                                     |
                   +-----------------+------------------+
                   |                                    |
           Mock deterministic backend               Real vLLM
           fixed response / SSE / errors        H100 and H200 pools
                   |                                    |
             schema and fault tests           integration / feature tests
                   +-----------------+------------------+
                                     |
                        Compatibility report + CI gates
```

**CI groups:**

1. **Per-PR, fast and deterministic (mandatory):** fake upstream; route/schema checks, auth, invalid parameters, error translation, common SSE event forms, forced disconnects, no GPU required.
2. **Merge or scheduled integration:** actual gateway and isolated vLLM model(s); official SDKs, LiteLLM, tool loops, typical long-running streaming; stricter time budgets.
3. **Scheduled compatibility and stress:** larger prompt/token-context boundaries, concurrent requests, load balancing, backend failures, race conditions. Cross-reference the [k6 plan](agent_portal_k6_load_and_workflow_testing.md) for load/performance metrics; do not conflate protocol compliance with latency/load benchmarking.

Capture gateway and upstream request IDs in test artifacts but redact tokens, credentials and sensitive prompt contents. Where feasible, trace request and response payload hashes to investigate translation mismatches.

## 9. CI report and acceptance criteria

Generate both a human-readable Markdown report and a machine-readable JSON report.

Example matrix:

| Interface | Operation | Feature | Client | Backend | Result |
|---|---|---|---|---|---|
| OpenAI | `/v1/chat/completions` | basic non-stream | OpenAI SDK | mock | PASS |
| OpenAI | `/v1/responses` | native stream | raw HTTP | mock | UNTESTED |
| Anthropic | `/v1/messages` | streaming tools | Anthropic SDK | vLLM | UNTESTED |
| Anthropic | `/v1/messages/count_tokens` | prompt with tools | raw HTTP | tokenizer | UNSUPPORTED |

These are **example rows, not actual test results**.

Proposed merge gates:

- All declared **P0 implemented operations** have positive and negative tests and pass against the deterministic mock backend.
- All declared streaming endpoints pass event schema, ordering, terminal status and error-event tests.
- Both official SDKs can deserialize the operations claimed to be supported; LiteLLM compatibility independently verified.
- No unintended cross-model routing, credential bypass, prompt leakage or silent fallback from one API to another.
- Unsupported features return appropriate, documented errors rather than misleading successes.
- Track P1 and conditional-feature coverage explicitly, with a date/owner for missing coverage.
- Version pinning: gateway release SHA, client dependencies, provider contract revision and vLLM version captured.
- Every regression gets a reproducible fixture, and previously passing P0 cases block releases when they fail.

Use separate labels in reports: `FAIL` indicates a claimed capability is broken, `UNSUPPORTED` means the capability is intentionally absent, and `UNTESTED` means compatibility has **not** been established.

## 10. Proposed implementation sequence

### Phase A: Surface inventory and deterministic smoke suite

- [ ] Inventory actual routes, provider dispatch logic and supported model IDs from the gateway code.
- [ ] Decide and document how OpenAI and Anthropic model-list routes are disambiguated.
- [ ] Create a versioned feature-support manifest.
- [ ] Add the six initial pytest smoke tests, plus official SDK model listings and model retrievals.
- [ ] Record emitted paths and payloads for LiteLLM requests. Verify there is no hidden translation false positive.

### Phase B: Native protocol correctness

- [ ] Implement or validate independent provider-specific serializers and error translators.
- [ ] Add strict JSON/schema and property-based request validation.
- [ ] Build an SSE parser/assertion library with protocol-specific terminal-event checks.
- [ ] Add streamed and non-streamed tool-call round trips, structured output and reasoning fixtures.
- [ ] Check `/v1/messages/count_tokens` against the actual serving tokenizer/template.
- [ ] Add adversarial auth, malformed input, timeout and cancellation tests.

### Phase C: Integration and quality gates

- [ ] Run supported endpoint and feature matrix against real vLLM backend(s).
- [ ] Compare direct-to-backend and through-gateway behavior and isolate gateway overhead.
- [ ] Exercise model selection with H100/H200 replicas and llm-d path routing if deployed.
- [ ] Add GitHub Actions (or existing CI runner) jobs with artifacts and P0 release gates.
- [ ] Run scheduled long-context/concurrency/resilience suites, coordinated with k6 load testing.
- [ ] Publish and maintain an explicit compatibility statement for portal users.

## 11. Decision summary

Use LiteLLM to ensure your gateway works with a common multiprovider client ecosystem. **Use official SDKs for native interoperability. Use raw HTTP/SSE, pinned API contracts and property-based tests for actual conformance.** Test ingress, provider adaptation and backend execution separately. Do not let a permissive SDK or translation layer conceal missing routes, invalid provider-shaped responses or unsupported model behavior.

This plan can be implemented incrementally without changing the production gateway architecture, requiring Kubernetes, or deploying remote MCP servers.
