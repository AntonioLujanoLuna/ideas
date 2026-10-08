# Adaptive LLM Guardian with Risk Memory

**Status:** Design proposal · **Updated:** 2026-10-08  
**Scope:** Corporate gateway for developer IDEs, LangGraph/Copilot SDK agents, MCP tools and RAG.  
**Goal:** Address the [OWASP LLM Top 10 (2026)](https://github.com/GenAI-Security-Project/GenAI-LLM-Top10/tree/main/2026/final) with measurable safeguards and minimal routine inference overhead.

## 1. Idea summary

Build a **self-hosted security middleware** in front of the existing gateway's vLLM routing. A compact classifier flags prompt injection and similar threats; deterministic controls enforce identities, permissions, resource limits and sensitive-data boundaries. Suspicious or uncertain cases can query a dedicated **risk memory** in Elasticsearch containing *verified attacks and benign lookalikes*. Reviewed incidents improve detection and eventually train a smaller company-specific classifier.

**Four architectural rules:**

1. **Do not replace detection with vector similarity.** New attacks need not resemble known ones, while security documentation may closely resemble malicious instructions.
2. **Keep retrieval off the normal path.** Use incremental scanning and an exact-content *feature* cache; query the vector index only when useful.
3. **Never delegate authorization to a model.** Tool execution, cross-tenant retrieval, network egress and quotas require hard policy checks.
4. **Learn only from verified evidence.** Automated promotion of attacker-submitted prompts poisons the risk database and training loop.

**Initial recommendation:** Benchmark [Prompt Armor](https://github.com/prompt-armor/prompt-armor) as a possible foundation versus a simpler ONNX-based detector. Use [PIGuard](https://github.com/leolee99/PIGuard)'s overdefense evaluation methods and [Vigil](https://github.com/deadbits/vigil-llm)'s hybrid/vector scanning as design references. Target **<30 ms p95 added latency for routine short requests** as an experimental SLO, not a claimed result.

## 2. Proposed architecture

```text
Claude Code / Copilot / Cursor / portal agents
                   |
                   v
           Existing API gateway
     auth / user scope / limits / provenance
                   |
                   v
         Incremental input inspection
           /                    \
     Deterministic          Small classifier
     policy checks          (CPU / ONNX)
           \                    /
            \                  /
             v                v
                  Policy engine
               /       |       \
            allow   uncertain  block
              |        |         |
              |        v         +--> audit
              |    Risk memory
              |    (Elasticsearch)
              |        |
              +--------+
                   |
                   v
            llm-d -> vLLM
                   |
                   v
       Selective response / egress DLP

Separate boundaries: agent tool router / local executor
  -> scoped authorization -> validated arguments
  -> sensitive-action approval / sandbox

Async only: event -> security review -> verified labels
           -> risk index -> evaluation / fine-tuning
```

This design fits the current H100 + 2×H200 Qwen deployment without allocating another large decoder to a GPU. Put the guardian near the gateway; llm-d still performs load/prefix-cache-aware model routing. Existing Elasticsearch infrastructure can host a **separately permissioned** risk index.

### Request handling

- Assign provenance and trust metadata at a server-controlled boundary: `user`, `repository_file`, `retrieved_chunk`, `tool_result`, `tool_call`, `model_output`. Do not trust client-supplied role labels.
- Inspect only **new or modified segments** in long conversations. Chunk long content with overlap; never silently treat a truncated scan as safe.
- Run inexpensive deterministic checks and a compact classifier; cache *features* keyed by content hash, provenance, detector version and preprocessing. **Always re-evaluate current authorization.**
- Allow, block, or escalate using a versioned policy. Consult Elasticsearch only for borderline/suspicious cases.
- For high-impact agent actions, enforce policy **where the tool executes**, including client-side tools invisible to the model gateway.
- Inspect selected responses **before releasing** sensitive content. A scan after streaming cannot retract leaked bytes.

## 3. OWASP 2026 control coverage

A security classifier is only part of OWASP coverage. The categories below correspond to the [2026 final definitions](https://github.com/GenAI-Security-Project/GenAI-LLM-Top10/tree/main/2026/final).

| Category | Principal safeguard |
| --- | --- |
| **LLM01 Prompt Injection** | Input/tool-result classifier, trust-boundary labels, isolation of retrieved instructions |
| **LLM02 Sensitive Information Disclosure** | Secret/PII DLP, retrieval ACLs, egress checks, context minimization |
| **LLM03 Excessive Agency** | Per-user tool scopes, validated arguments, approvals, sandboxing |
| **LLM04 Supply Chain** | Dependency/model provenance, scanning, pinned verified artifacts |
| **LLM05 Data and Model Poisoning** | Ingestion validation, source integrity, index provenance, review |
| **LLM06 Unbounded Consumption** | Token/concurrency/rate/time budgets enforced before GPU inference |
| **LLM07 Misinformation** | Optional grounded verification, cited evidence and human oversight |
| **LLM08 Hidden Context Exposure** | Prompt minimization, tenant separation, secret/context-output restrictions |
| **LLM09 Vector and Embedding Weaknesses** | Pre-retrieval ACL filters, vector-index isolation and poisoning tests |
| **LLM10 Improper Output Handling** | Output schemas, escaping, command validation and sandboxing |

The guardian detects textual signals most directly for **LLM01** and some disclosure/agency scenarios; other categories depend mainly on hard application controls. Track implementation and tests per category, not an invented overall "OWASP safety score".

## 4. Existing projects worth evaluating

| Project | How it works / what to reuse | License and limitations |
| --- | --- | --- |
| **[Prompt Armor](https://github.com/prompt-armor/prompt-armor)** | Parallel regex, DeBERTa ONNX, contrastive vector similarity, structural checks and anomaly scoring, followed by score fusion; closest match to this design. Candidate to **extend instead of rebuilding**. | **Apache 2.0** for repository; project-reported latency/accuracy need independent validation. Audit bundled models, dependencies, benchmark leakage, and operational maturity. |
| **[Vigil](https://github.com/deadbits/vigil-llm)** | Modular prompt/response scanner with YARA, transformers, vector similarity, optional updating, canary and other detectors. Useful prior art for security signatures and risk memory. | **Apache 2.0**; upstream explicitly calls it **experimental/alpha**, so treat as a reference rather than an unreviewed production dependency. Not the unrelated Vigil AI-SOC product. |
| **[PIGuard](https://github.com/leolee99/PIGuard)** | Prompt-injection model and *NotInject* benign-trigger-word dataset designed to reduce **overdefense**, particularly valuable for developers discussing exploits and security concepts. | Repository **MIT**; verify model-weight, training-data and dependency terms separately. Benchmark short-text accuracy/latency and unknown attack families. |

**Evaluation order:** Start with Prompt Armor and a minimal single-model ONNX baseline. Test PIGuard alongside the baseline for reduced false positives. Borrow useful Vigil techniques selectively. None provides the full OWASP authorization/egress protection needed for agents.

Other model baselines: [Prompt Guard 2 22M](https://huggingface.co/meta-llama/Llama-Prompt-Guard-2-22M) or [86M](https://huggingface.co/meta-llama/Llama-Prompt-Guard-2-86M) (Llama 4 Community License, review for corporate use), and [ProtectAI DeBERTa v3](https://huggingface.co/protectai/deberta-v3-base-prompt-injection-v2) (Apache 2.0, narrower scope). Commercial use of Liquid d1-3B may require a separate agreement under the LFM license: do not make it the default.

## 5. Risk memory and adaptive learning

Use dedicated Elasticsearch partitions:

- `confirmed_attacks`: reviewed attack examples, family, technique, source, outcome.
- `benign_hard_negatives`: legitimate examples that resemble attacks, with the authorization context explaining why.
- `pending_incidents`: unverified alerts. **Never** use these as trusted labels.

Store the minimum useful metadata: case ID, tenant scope, source/trust type, OWASP category, content digest, embeddings with model version, detector/policy versions, review status, outcome and retention deadline. Prefer redacted examples and pointers to restricted evidence. Both raw text **and embeddings** require access controls and retention.

### Retrieval scoring

For a candidate segment `x`, compare its nearest verified malicious and benign neighbors:

```text
attack_sim(x) = max cosine(e(x), e(verified_attack))
benign_sim(x) = max cosine(e(x), e(verified_benign))
margin(x)     = attack_sim(x) - benign_sim(x)
```

Combine both similarities, `margin`, classifier outputs, provenance, tool capabilities and risk category in a **calibrated decision function**. Use BM25 or signatures alongside embeddings where useful. A cosine threshold is **not** a probability of malicious intent and must never automatically whitelist content.

Optimize per-category thresholds on **held-out attack families and benign developer requests**, not paraphrase-contaminated random splits. Evaluate classifier-only, classifier+memory and a fine-tuned compact model. Promote cases to trusted partitions only after independent review; retrain periodically with immutable test sets, canary rollout and rollback.

## 6. Latency, failure policy and evaluation

**Routine short-request target:** additional gateway p95 **<30 ms**. This is a proposed benchmark target, not a guarantee. Monitor p50/p95/p99 separately for deterministic checks, CPU inference, ANN escalation and total overhead. Long new documents require a **different SLO** and bounded inspection resources.

To control overhead:

- Run warmed, CPU-hosted ONNX inference near the gateway first. Compare GPU only if measured CPU throughput fails.
- Reuse classifier *features* for repeated exact segments; never reuse a previous user's permission decision.
- Keep ANN retrieval **conditional**. Deduplicate or cluster repeated reviewed attacks.
- Isolate worker pools, bound queue length and fail safely under guardian outages. High-risk tool actions should fail closed; lower-risk read-only requests may follow an explicitly documented degraded-mode policy with audit.
- Avoid blocking security researchers for merely quoting attacks. Inspect **intent, provenance and requested operation**.

Evaluate with genuine coding tasks, GitLab READMEs, RAG/tool-result injection, Jira/MCP misuse, exfiltration attempts, Spanish/English prompts, obfuscations, long context and benign security explanations. Key metrics: unseen-family recall, false blocks per 1,000 benign developer requests, escalation rate, precision/calibration, p95/p99 latency, and unauthorized actions actually prevented.

## 7. Implementation plan

| Phase | Deliverable | Exit criterion |
| --- | --- | --- |
| **0. Threat model** | Inventory gateway visibility, source trust, agent tool execution, licenses and representative labeled cases | Owners and hard controls mapped to OWASP |
| **1. Shadow guardian** | Deterministic checks + CPU classifier + incremental chunking + versioned events | Measured latency and false-positive/recall baseline |
| **2. Risk memory** | Isolated Elasticsearch indexes, reviewed attacks and benign negatives, conditional retrieval | Demonstrated accuracy benefit for acceptable retrieval overhead |
| **3. Agent enforcement** | Hooks in LangGraph/Copilot SDK tool dispatch and local executor, per-user authorization, DLP | Unsafe actions blocked at execution/egress boundaries |
| **4. Adaptive student** | Reviewed training corpus, compact-model tuning, recalibration, canary and rollback | Improved security/latency tradeoff on independent data |

Start with **Phase 1**, using Prompt Armor versus a simple ONNX baseline in shadow mode. Do not commit to building a custom vector-heavy engine before these measurements.

## 8. Links to existing infrastructure proposals

- [llm-d cache-aware vLLM routing](./llm_d_cache_aware_vllm_routing.md)
- [Path-multiplexed vLLM replicas](./llm_d_path_multiplexed_vllm_replicas.md)
- [Portal agent tool execution](./portal_agent_tool_execution_architecture.md)
- [OpenClaw / Copilot agent automation](./openclaw_copilot_agents_automation_architecture.md)

**Bottom line:** **Detect every new relevant trust-boundary input cheaply, retrieve reviewed precedents when uncertain, enforce real permissions at action boundaries, and improve a compact guardian only with verified feedback.**
