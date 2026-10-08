# Adaptive LLM Guardian: Low-Latency OWASP Defense with Risk Memory

**Status:** Architecture and implementation proposal  
**Date:** 2026-10-08  
**Scope:** Corporate LLM gateway for developer IDEs, LangGraph and Copilot SDK agents, RAG, MCP/server tools, and client-executed tools.  
**Primary objective:** Add meaningful, measurable defenses against the [OWASP GenAI LLM Top 10 2026](https://github.com/GenAI-Security-Project/GenAI-LLM-Top10/tree/main/2026/final) without materially increasing normal LLM request latency.  
**Guiding principle:** Models detect suspicious behavior; deterministic policy controls enforce security. A vector database is *risk memory*, not a substitute for either.

## 1. Context and decision

Existing deployment:

- IDEs (Claude Code, GitHub Copilot, Cursor) and internal agents call a central API gateway.
- The gateway routes to locally deployed Qwen/vLLM replicas, currently one H100 and two H200 GPUs, using Nginx and planned cache-aware llm-d routing.
- LangGraph and Copilot SDK agents can use internal RAG, Jira/GitLab/MCP services and potentially client-side filesystem, code-execution, and other tools.
- Elasticsearch is already available for retrieval; deployment is on-premises, with enterprise networking/proxy constraints.
- The LLM can receive very long contexts. Rescanning the complete conversation on every request is unacceptable.

**Decision:** Implement a small, self-hosted, multi-layer security service immediately upstream of model routing. Run cheap deterministic checks and a compact prompt-injection detector on *new relevant content*. On uncertain cases, retrieve verified examples from a dedicated risk index. Review and label incidents asynchronously. Periodically calibrate or fine-tune a compact model using verified hard examples.

Do **not** switch to "vector-only detection" once the index becomes large. A never-seen attack can be far from known vectors, while authorized security research can be semantically almost identical to malicious instructions.

## 2. What the guardian can and cannot secure

A classifier cannot "cover OWASP" alone. The 2026 release has ten categories, many involving authorization, isolation, software integrity and operational policy rather than suspicious prompt text.

| OWASP 2026 category | Guardian/model contribution | Required enforcement beyond classification |
| --- | --- | --- |
| LLM01 Prompt Injection | Score new user messages, retrieved documents and tool results for instruction attempts and trust-boundary crossings | Treat retrieved/tool content as data; protect higher-priority instructions and tool authorization |
| LLM02 Sensitive Information Disclosure | Detect likely exfiltration requests; optionally inspect egress text for secrets/PII | Data classification, ACLs, context minimization, secret redaction, DLP and egress controls |
| LLM03 Excessive Agency | Identify suspicious tool intentions and anomalous requested actions | Per-user/service identity, allowlisted tools, argument checks, scopes, approval for sensitive operations |
| LLM04 Supply Chain | Little direct value from request classification | Pinned/checksummed model and package artifacts, provenance, signed releases, dependency scanning |
| LLM05 Data and Model Poisoning | Flag suspect ingested documents or anomalous content | Provenance, permissions, ingestion validation, immutable audit trail and index quality checks |
| LLM06 Unbounded Consumption | Recognize some abuse patterns | Hard token, concurrency, spend, time and request budgets; queueing/rate limits before inference |
| LLM07 Misinformation | Optional response-versus-evidence review | Source attribution, task-specific verification, user-facing uncertainty, human review where necessary |
| LLM08 Hidden Context Exposure | Identify attempts to extract system/developer/private context | Never inject unauthorized context; separate tenants; minimize prompts; outgoing data controls |
| LLM09 Vector and Embedding Weaknesses | Detect some suspicious retrieved content | Retrieval-time ACL filtering, index isolation, provenance, deletion, poisoning tests |
| LLM10 Improper Output Handling | Detect some dangerous output patterns | Typed schemas, escaping, safe rendering, command validation and execution sandboxing |

**OWASP is a threat-model checklist, not a certification granted by a model score.** Maintain evidence of which controls mitigate each category.

## 3. High-level architecture

```text
Developer IDEs / portal agents / MCP + RAG tool outputs
                         |
                         v
             Corporate API gateway
             - identity / tenant / quota
             - message provenance labels
             - incremental content tracking
                         |
          +--------------+--------------+
          |                             |
          v                             v
 Deterministic policy checks    Compact injection classifier
 - RBAC/ABAC, token limits      - CPU / ONNX candidate
 - sensitive patterns           - scan new chunks only
 - tool permissions             - calibrated score
          |                             |
          +--------------+--------------+
                         |
                         v
                  Policy evaluator
               /         |         \
            allow      uncertain     block
              |           |             |
              |           v             v
              |     Risk-memory lookup  Audit/response
              |     + optional deeper
              |     investigation
              |           |
              +-----------+
                         |
                         v
             llm-d / Nginx -> vLLM
                         |
               streamed outputs
                         |
          output DLP + schema/tool controls
                         |
                         v
                Client or agent

Off critical path:
Events -> verified case queue -> security review / labeling
       -> isolated Elasticsearch risk index
       -> calibration, hard negatives, model evaluation
       -> versioned model/policy promotion
```

The **security middleware sits before llm-d routing** so protection is model-pool independent. llm-d remains responsible for scheduling and cache-aware routing, not authorization. Keep its endpoint health and metrics isolated from the guardian.

**Agent-side enforcement is mandatory:** The gateway cannot see, much less block, a filesystem write, shell command or client-side tool call executed without passing through it. Add execution hooks to LangGraph/Copilot SDK tool routers and the client tool executor. A prompt classifier is not a sandbox.

## 4. Fast path: incremental, bounded classification

### 4.1 Ingress and provenance

Extract/derive for each content segment:

- `source_type`: `user_message`, `system_instruction`, `retrieved_document`, `repository_file`, `tool_result`, `tool_request`, `model_output`, etc.
- `trust_level` determined by the *server-side integration*, never solely by client-supplied role labels.
- `tenant_id`, authenticated principal, tool identity, intended sink/operation and capabilities, where available.
- `content_digest`, model/checkpoint identifier, detector version and policy version.
- Content range, offsets, token count, language and relevant parent request/session identifiers.

Reject spoofed/unauthorized privileged roles at the gateway. Treat arbitrary message bodies, repository files and third-party tool results as untrusted even if their text says "system".

Only classify *new or modified* relevant segments where their identity and provenance can be established securely. Repeated segments can reuse **cached detector features**, but **not** cached authorization decisions; privileges and policy may have changed.

### 4.2 Bounded inspection

- For short content, scan directly.
- For longer input, segment by natural boundaries plus overlapping token windows, then aggregate risk using a conservative policy. A 512-token model must not silently truncate away the attack.
- Apply strict maximum inspection work per request and use staged inspection for very large inputs. Never interpret partial scanning as proof of safety.
- For extremely large new untrusted documents, perform full asynchronous ingestion scanning before they enter a trusted retrieval index, plus cheap checks when fetched.
- Do not resend or rescan complete 32k/128k/262k prompt histories merely to classify a new user instruction.
- Preserve high-signal metadata, including *who supplied content*, *where it came from*, and *what tool action it tries to cause*.

**Important tradeoff:** A large number of new chunks can exceed a 30 ms target. Report this as a separate long-input SLO, not as a routine short-request sample.

### 4.3 Low-latency model candidates

Start with licensing review and reproducible benchmarks, not vendor accuracy claims.

| Candidate | License consideration | Intended role |
| --- | --- | --- |
| [Meta Prompt Guard 2 22M](https://huggingface.co/meta-llama/Llama-Prompt-Guard-2-22M) | Llama 4 Community License, **not** Apache 2.0; internal commercial usage subject to its terms | Small fast injection/jailbreak baseline |
| [Meta Prompt Guard 2 86M](https://huggingface.co/meta-llama/Llama-Prompt-Guard-2-86M) | Same license family; verify specifics | Stronger-classifier candidate, measure CPU cost |
| [ProtectAI DeBERTa v3 prompt injection v2](https://huggingface.co/protectai/deberta-v3-base-prompt-injection-v2) | Apache 2.0 as shown on model card | Permissive-licensed baseline; English/injection focus and known limits |
| A small internally fine-tuned encoder | Validate *base weights*, datasets, redistribution and training artifacts | Eventual specialized low-cost deployment |

Do not default to Liquid d1-3B for this corporate guardian: the LFM Open License's commercial-use terms may require a separate agreement for a large commercial organization. Kev/Laya can be evaluated separately, but neither should be presumed a robust security detector merely from general decision benchmarks.

**Baseline:** CPU-hosted ONNX Runtime service, with warmed model/tokenizer, pinned weights and bounded thread pools. Benchmark against a co-located small GPU service only if CPU p95/p99 latency or throughput proves inadequate; do not reduce the Qwen vLLM KV-cache budget without measuring the cost.

### 4.4 Policy outcomes

- **Allow:** no hard-policy violation; classifier risk is low within an evaluated risk class and trusted-enough context.
- **Escalate:** unusual or uncertain text, untrusted tool output pushing instructions, sensitive intended operation, score disagreement or new technique. Invoke vector retrieval and, if necessary, a slower classifier/human approval.
- **Block:** deterministic policy violation, well-supported high-confidence attack, unauthorized tool operation, hard quota exceeded, or non-overridable data policy.
- **Observe-only:** initial deployment and cases without sufficient calibration. Continue collection without disrupting developers.

Do not use one universal threshold, and do not map uncalibrated softmax scores straight to "probability of compromise".

## 5. Risk memory: the Elasticsearch security index

Reuse existing Elasticsearch deployment if isolation, access controls, latency and capacity requirements allow it, but use a **dedicated restricted index, namespace, service account and retention policy**. This index must never become a general user-facing RAG corpus.

Recommended logical partitions:

| Partition | Content | Trust requirements |
| --- | --- | --- |
| `guardian-confirmed-attacks` | Verified attack families, variants, security-relevant context and observed outcome | Security-reviewed; immutable provenance/verification record |
| `guardian-benign-hard-negatives` | Legitimate developer prompts that resemble attacks | Security-reviewed; include reason/authorized context |
| `guardian-pending-events` | Model flags, retrieved-neighbor evidence and uncertain observations | **Never** automatically treated as ground truth |

Fields:

```text
case_id, tenant_scope, category, technique, source_type, trust_level,
normalized_features, embedding, embedding_model_version,
detector_version, policy_version, confidence, review_status,
attack_family_id, outcome, first_seen, last_seen, retention_deadline,
evidence_pointer, minimal_redacted_excerpt
```

Prefer minimized, redacted evidence and a reference to an access-controlled original log over retaining raw proprietary prompts, code or secrets in the vector index. Embeddings can still leak sensitive information and must be access-controlled. Enforce tenant boundaries before similarity search, not after retrieving results.

### 5.1 Retrieval signal, not security verdict

For incoming content `x`, retrieve nearest **verified** malicious and benign neighbors.

```text
s_attack(x) = max cosine(embedding(x), embedding(a)) over validated attacks a
s_benign(x) = max cosine(embedding(x), embedding(b)) over validated benign cases b
margin(x)   = s_attack(x) - s_benign(x)
```

Use `s_attack`, `s_benign`, `margin`, source provenance, intent/tool context, nearest attack-family diversity and classifier score as features in a **calibrated fusion policy**.

They are not calibrated likelihoods of compromise. A high similarity to an attack could mean "the developer is asking us to explain that attack"; low similarity may simply mean a new exploit.

Potential improvements:

- Hybrid lexical (BM25/signatures) + dense retrieval for obfuscated strings, exact indicators and paraphrases.
- Hard benign neighbors to suppress false positives for normal vulnerability analysis and code-review traffic.
- Per-category or per-source threshold calibration.
- Case deduplication and family centroids, not thousands of duplicate vectors.
- Scheduled pruning, retention and embedding-model migration plans.

**Critical rule:** No vector match can *on its own* prove a request safe or override deterministic authorization/egress controls. Retrieval should normally happen only for uncertain/escalated events.

### 5.2 Cache versus vector search

There are two different optimizations:

1. **Exact-segment feature cache:** Cache classifier results for identical content with the same provenance, content preprocessing, detector/checkpoint and policy version. Recheck the current user's authority and intended action each time. Useful for repeated system templates, retrieval chunks and repeated conversation history.
2. **Vector risk memory:** Fuzzy precedent retrieval for uncertain or suspicious cases. Useful to explain, categorize and evaluate known attack families; not a blanket safe-answer cache.

Avoid semantic "allow caches". Similar text can cross different trust boundaries and require different policy decisions. An attacker must not be able to seed vectors or poison the reviewed partitions by submitting crafted prompts.

## 6. Agents, tools and streaming responses

### Agent-specific boundaries

A secure design inspects:

1. **Before model inference:** new user text and external/retrieved/tool-result segments.
2. **Before executing tools:** tool identity, per-user identity, operation, arguments, destination and capability/risk class, enforced at the execution location.
3. **Before sending sensitive data out:** network destinations, uploaded files, secrets, credential material, internal RAG results and privileged context.
4. **During model output:** schema/format validation and selective content scanning. If an output must not disclose secrets, buffer/check before releasing relevant content; inspecting a chunk *after* streaming it to the caller is too late.
5. **At ingestion:** poisoning/provenance checks before documents and code snippets enter trusted indexes.

Tool approval cannot be substituted with "classifier says safe". For example, a legitimate user may request a destructive command that is still prohibited by the application role. Conversely, developers may legitimately *describe* dangerous commands without asking an agent to execute them.

Client-side Copilot/Claude Code actions require local hooks, enterprise endpoint controls or capability-restricted tool executors. The gateway alone cannot promise comprehensive visibility of local execution.

### Failures and backpressure

- **Fail closed** for explicit policy checks and privileged actions when required identity/authorization cannot be established.
- For a transient *model-detector* outage, select an explicit policy by risk tier: e.g. deterministic controls remain mandatory, lower-risk requests may proceed with audit, and high-impact actions must block or require human approval.
- Timebox guardian calls; isolate queues and circuit breakers from vLLM and llm-d.
- Independently budget inspection CPU, Elasticsearch ANN work and review traffic to prevent the guardian itself becoming a denial-of-service target.

## 7. Optimization and evaluation

### 7.1 Initial SLOs (engineering targets, not measured claims)

| Traffic class / metric | Proposed initial target |
| --- | --- |
| Short routine request, added gateway latency | p95 < 30 ms, p99 monitored |
| Deterministic middleware checks | p95 < 5 ms |
| Short-window model classification | p95 < 25 ms, subject to actual CPU and model |
| Normal-request vector retrieval | None; should not be on the default path |
| Large new tool results / long documents | Separate SLO and explicit resource budget |
| Security effectiveness | Optimize recall for *unseen* attack families subject to a low legitimate-developer false-positive rate |

Instrument *added guardian latency* independently of vLLM TTFT, output throughput and queue time. Compare baseline GW -> vLLM to GW -> guardian -> vLLM at matched load and token distributions.

### 7.2 Dataset and test design

Include labeled cases for:

- Realistic IDE tasks, vulnerability write-ups and benign security research.
- Direct prompt injection, tool-result injection and malicious repository/README content.
- Legitimate and unauthorized shell commands, filesystem writes, network calls and MCP operations.
- Attempts to exfiltrate secrets/system instructions, RAG leakage and cross-tenant retrieval.
- Multilingual and obfuscated payloads, adversarial paraphrases and long-context placement.
- Both trusted-user instructions and the same strings embedded in *untrusted* content.
- Legitimate "false-positive trap" prompts and code comments that mention known attack terms.

Use **attack-family-separated and time-separated** train/validation/test splits. Deduplicate paraphrases and near-identical samples before splitting. Measure performance by source type, language, request size and downstream capability.

Primary metrics:

- Attack recall / false-negative rate on previously unseen families.
- False-positive rate per 1,000 *legitimate developer requests* and false blocking of routine code analysis.
- Detection precision, PR-AUC, probability calibration (Brier/ECE where applicable).
- Added p50/p95/p99 latency, classifier time, ANN escalation rate, cost and CPU utilization.
- Privileged tool actions blocked by deterministic policy, attempted data egress prevented.
- Reviewer volume, labeling agreement, drift, false-positive reversal rate and risk-index growth.

Threshold selection is a **risk-constrained optimization** rather than maximizing generic F1:

```text
Minimize   C_FN * missed_attacks + C_FP * false_blocks + C_L * added_latency
Subject to mandatory authorization, DLP and resource enforcement rules.
```

Choose distinct thresholds by source/risk tier when enough validation data exists, freeze them before the held-out test, and revalidate after every model, policy or embedding change.

## 8. Adaptation without poisoning the guardian

Use an asynchronous, controlled feedback loop:

1. Collect structured security events and sufficient minimized evidence.
2. Deduplicate by technique/family and queue uncertain/high-impact cases.
3. Human or trusted security process **verifies** ground truth and downstream outcome.
4. Promote verified attacks into `guardian-confirmed-attacks` and legitimate lookalikes into `guardian-benign-hard-negatives`.
5. Evaluate a risk-memory-augmented decision rule in shadow mode.
6. Periodically fine-tune a small classifier using verified, provenance-preserving cases; consider contrastive representation training for the security embedding model.
7. Calibrate and compare against the frozen baseline on unseen techniques and benign developer data.
8. Release with versioned artifacts, feature flags, canary rollout and rollback.

Never automatically promote a model flag or vector match into trusted training labels. Rate-limit attacker-supplied samples, retain immutable review history, prevent cross-tenant case leakage, and maintain independent evaluation sets.

## 9. Suggested implementation plan

### Phase 0: Baseline and threat model

- [ ] Confirm gateway request shape, trust/provenance metadata, agent tool-execution surfaces and authorized client roles.
- [ ] Baseline current TTFT, concurrency and GW overhead.
- [ ] Establish OWASP 2026 matrix of controls, owners and tests.
- [ ] Obtain approval for model-weight, dataset and dependency licenses. Prefer permissive licensing when feasible.

**Exit:** Agreed security-policy boundaries and representative labeled evaluation corpus.

### Phase 1: Minimal guardian in shadow mode

- [ ] Deploy a small CPU/ONNX classifier behind an internal-only service endpoint.
- [ ] Implement deterministic authentication, scoped authorization, quotas and simple DLP/signature checks.
- [ ] Add new-content extraction, provenance, token chunking and exact feature cache.
- [ ] Return structured risk scores and policy decisions; only enforce existing deterministic policy initially.
- [ ] Record p50/p95/p99 and false-positive reports without changing developer workflows.

**Exit:** Measured short-path overhead and initial category-specific recall/FPR.

### Phase 2: Risk memory and targeted escalation

- [ ] Create isolated Elasticsearch security indices with ACLs and retention.
- [ ] Add reviewed attack/benign cases, hybrid retrieval and nearest-neighbor explanations.
- [ ] Retrieve only for uncertain or risk-escalated traffic.
- [ ] Calibrate per-category decisions and demonstrate incremental benefit over classifier-only.
- [ ] Add incident review/adjudication and threat-intelligence updates.

**Exit:** Reduced false positives or increased unseen-attack recall at acceptable ANN escalation cost.

### Phase 3: Agent/action enforcement

- [ ] Instrument LangGraph and Copilot SDK tool routers and client tool executor.
- [ ] Enforce per-user identity, least privilege, destination/argument allowlists, approvals and sandboxing.
- [ ] Inspect untrusted tool results and RAG ingress; add targeted pre-egress checks.
- [ ] Run end-to-end exfiltration, filesystem, MCP, Jira and GitLab abuse simulations.

**Exit:** Controls block unsafe actions *at execution boundaries*, even when text detection misses.

### Phase 4: Adaptive compact student model

- [ ] Build a human-verified attack-family/hard-negative dataset.
- [ ] Fine-tune or distill a compact encoder and optionally task-specific embeddings.
- [ ] Compare student-only, baseline-only, baseline+memory and student+memory.
- [ ] Canary deploy with versioned thresholds, audit and rollback.
- [ ] Decide whether vector retrieval can become rarer, not whether it should fully replace detection.

**Exit:** Demonstrated improvement in the latency/security Pareto frontier on independent tests.

## 10. Open design choices

1. **Where to host:** gateway-local CPU service (preferred first) versus an isolated GPU service.
2. **Model license:** Llama Prompt Guard 2 with legal review versus Apache-licensed ProtectAI/other encoder.
3. **Long contexts:** exact segmentation strategy, sliding-window overlap, allowed partial scanning and separate limits.
4. **Risk-memory privacy:** whether even redacted prompt excerpts/embeddings may be retained; per-tenant or shared reviewed patterns.
5. **Tool policy:** what can be enforced at the gateway, portal backend and local client executor; which actions require approval.
6. **Streaming:** which output classes need buffer-before-release, and how this changes TTFT and perceived latency.
7. **Incident review:** ownership, response workflow, retention, policy-versioning and SLAs.
8. **Cost of escalation:** CPU/ANN budgets, timeout behavior and fallbacks under load.

## 11. References and related designs

- [OWASP GenAI LLM Top 10 2026, canonical source](https://github.com/GenAI-Security-Project/GenAI-LLM-Top10/tree/main/2026/final)
- [OWASP GenAI Security Project](https://genai.owasp.org/)
- [Llama Prompt Guard 2, 22M](https://huggingface.co/meta-llama/Llama-Prompt-Guard-2-22M)
- [Llama Prompt Guard 2, 86M](https://huggingface.co/meta-llama/Llama-Prompt-Guard-2-86M)
- [Prompt Guard 2 license](https://huggingface.co/meta-llama/Llama-Prompt-Guard-2-22M/blob/main/LICENSE)
- [ProtectAI DeBERTa prompt-injection classifier](https://huggingface.co/protectai/deberta-v3-base-prompt-injection-v2)
- [Meta LlamaFirewall](https://github.com/meta-llama/PurpleLlama/tree/main/LlamaFirewall), useful as an architectural reference; verify component availability/license.
- [llm-d cache-aware routing](./llm_d_cache_aware_vllm_routing.md)
- [Path-multiplexed vLLM replicas](./llm_d_path_multiplexed_vllm_replicas.md)
- [Portal agent tool execution](./portal_agent_tool_execution_architecture.md)
- [OpenClaw/Copilot agent automation](./openclaw_copilot_agents_automation_architecture.md)

---

**One-sentence design conclusion:** Deploy a small, always-on guardian that learns from **verified** attacks and legitimate lookalikes; use exact caching to avoid repeated inference, vector retrieval to investigate ambiguity, and deterministic permissions to prevent unsafe actions regardless of model score.
