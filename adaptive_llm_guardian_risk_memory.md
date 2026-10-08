# Adaptive LLM Guardian with Risk Memory

**Status:** Design proposal · **Updated:** 2026-10-08  
**Scope:** Corporate gateway for developer IDEs, LangGraph/Copilot SDK agents, MCP tools and RAG.  
**Goal:** Address the [OWASP LLM Top 10 (2026)](https://github.com/GenAI-Security-Project/GenAI-LLM-Top10/tree/main/2026/final) with measurable safeguards and minimal routine inference overhead.

## 1. Idea summary

Build a **self-hosted security layer** ahead of vLLM routing. Evaluate **[CerbIA](https://github.com/InditexTech/cerbia)** as the configurable scanning and verdict pipeline, pairing inexpensive rules with optional local ML. The gateway and tool executors independently enforce permissions, limits and egress restrictions. Ambiguous events consult an Elasticsearch **risk memory** of *verified attacks and benign lookalikes*. Reviewed incidents help calibrate detectors and eventually train a compact company-specific model.

**Four architectural rules:**

1. **Do not replace detection with vector similarity.** New attacks need not resemble known ones, while security documentation may closely resemble malicious instructions.
2. **Keep retrieval off the normal path.** Use incremental scanning and an exact-content *feature* cache; query the vector index only when useful.
3. **Never delegate authorization to a model.** Tool execution, cross-tenant retrieval, network egress and quotas require hard policy checks.
4. **Learn only from verified evidence.** Automated promotion of attacker-submitted prompts poisons the risk database and training loop.

**Initial recommendation:** Prototype **CerbIA as the orchestration layer**, and benchmark its lightweight rules-only and optional ProtectAI detection configurations against **[Prompt Armor](https://github.com/prompt-armor/prompt-armor)** and a minimal ONNX baseline. Use [PIGuard](https://github.com/leolee99/PIGuard) for overdefense tests and [Vigil](https://github.com/deadbits/vigil-llm) for historical hybrid/vector-scanning patterns. Target **<30 ms p95 added latency on short requests** as an experimental SLO, not a published CerbIA result.

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
                    |
            CerbIA scan pipeline
       deobfuscation / fast rules /
       optional local ML classifier
                    |
     Gateway policy / verdict evaluator
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

This fits the current H100 + 2×H200 Qwen deployment without allocating another large decoder to a GPU. Run CerbIA in a gateway-local Python 3.12+ process or a dedicated internal service; its documented `SecurityGate.scan()` API supports programmatic integration. CerbIA returns scan findings/verdicts, while **the gateway remains the final policy authority**. llm-d still handles cache-aware model routing. Keep the Elasticsearch risk index **separately permissioned**. CerbIA does *not* document built-in vector risk memory or agent-execution authorization: those remain our components.

### Request handling

- Assign provenance and trust metadata at a server-controlled boundary: `user`, `repository_file`, `retrieved_chunk`, `tool_result`, `tool_call`, `model_output`. Do not trust client-supplied role labels.
- Inspect only **new or modified segments** in long conversations. Chunk long content with overlap; never silently treat a truncated scan as safe.
- Use CerbIA's configurable loaders/preprocessors/scanners and optional local ProtectAI classifier. Start with selective fast checks (e.g. instruction patterns, secrets, URLs, invisible text). Cache *features* keyed by content hash, provenance, scanner versions and preprocessing. **Always re-evaluate current authorization.**
- Configure CerbIA scanner actions, score aggregation, thresholds and scanner-error behavior explicitly; a safe verdict is **not** authorization. The gateway implements `allow / escalate / block`, consulting Elasticsearch only for borderline/suspicious cases.
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
| **[CerbIA](https://github.com/InditexTech/cerbia)** | **First framework candidate:** configurable gates with loaders, deobfuscation/preprocessors, security scanners, score aggregation and verdicts; optional local ProtectAI and Presidio integrations. Wrap at the GW, with external risk memory and policy enforcement. | **Apache 2.0** repo, Python 3.12+; newly released, no validated enterprise latency benchmark or built-in risk memory. Audit scanner configuration, scoring, failure modes, and optional model licenses. |
| **[Prompt Armor](https://github.com/prompt-armor/prompt-armor)** | **Detection benchmark/alternative:** parallel regex, DeBERTa ONNX, contrastive similarity, structural and anomaly scoring. Consider as CerbIA-integrated detector only if measured benefits justify complexity. | **Apache 2.0** for repository; reported latency/accuracy require independent validation. Audit model/dependency licenses, benchmark leakage and maturity. |
| **[Vigil](https://github.com/deadbits/vigil-llm)** | Modular prompt/response scanner with YARA, transformers, vector similarity, optional updating, canary and other detectors. Useful prior art for security signatures and risk memory. | **Apache 2.0**; upstream explicitly calls it **experimental/alpha**, so treat as a reference rather than an unreviewed production dependency. Not the unrelated Vigil AI-SOC product. |
| **[PIGuard](https://github.com/leolee99/PIGuard)** | Prompt-injection model and *NotInject* benign-trigger-word dataset designed to reduce **overdefense**, particularly valuable for developers discussing exploits and security concepts. | Repository **MIT**; verify model-weight, training-data and dependency terms separately. Benchmark short-text accuracy/latency and unknown attack families. |

**Evaluation order:** (1) CerbIA rules-only, (2) CerbIA + optional ProtectAI, (3) Prompt Armor standalone and/or a minimal ONNX detector. Compare the security/latency tradeoff on the same held-out traffic before integrating Prompt Armor into CerbIA. Use PIGuard for false-positive testing and Vigil as reference material. **None substitutes for tool authorization, ACLs or egress controls.**

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

- Start with warmed CerbIA fast scanners in a gateway-local Python service; enable optional CPU-hosted ML/ONNX only if it improves held-out recall sufficiently. Compare GPU only if measured CPU throughput fails.
- Reuse classifier *features* for repeated exact segments; never reuse a previous user's permission decision.
- Keep ANN retrieval **conditional**. Deduplicate or cluster repeated reviewed attacks.
- Isolate worker pools, bound queue length and test CerbIA scanner errors, configuration-dependent `WARN` versus `BLOCK` handling, and aggregation behavior. High-risk tool actions should fail closed; lower-risk read-only requests may follow an explicitly documented degraded-mode policy with audit.
- Avoid blocking security researchers for merely quoting attacks. Inspect **intent, provenance and requested operation**.

Evaluate with genuine coding tasks, GitLab READMEs, RAG/tool-result injection, Jira/MCP misuse, exfiltration attempts, Spanish/English prompts, obfuscations, long context and benign security explanations. Key metrics: unseen-family recall, false blocks per 1,000 benign developer requests, escalation rate, precision/calibration, p95/p99 latency, and unauthorized actions actually prevented.

## 7. Implementation plan

| Phase | Deliverable | Exit criterion |
| --- | --- | --- |
| **0. Threat model** | Inventory gateway visibility, source trust, agent tool execution, licenses and representative labeled cases | Owners and hard controls mapped to OWASP |
| **1. Shadow guardian** | CerbIA rules-only / CerbIA+ML / Prompt Armor or ONNX baseline; incremental chunks and versioned events | Comparative latency, error handling, false-positive and recall baseline |
| **2. Risk memory** | Isolated Elasticsearch indexes, reviewed attacks and benign negatives, conditional retrieval | Demonstrated accuracy benefit for acceptable retrieval overhead |
| **3. Agent enforcement** | Hooks in LangGraph/Copilot SDK tool dispatch and local executor, per-user authorization, DLP | Unsafe actions blocked at execution/egress boundaries |
| **4. Adaptive student** | Reviewed training corpus, compact-model tuning, recalibration, canary and rollback | Improved security/latency tradeoff on independent data |

Start with **Phase 1**: adopt CerbIA only if its framework overhead and detection quality justify it against simpler alternatives. Do not commit to a custom vector-heavy engine before the benchmarks. CerbIA is young: review production readiness and pin/verify the version.

## 8. References and related infrastructure

- [CerbIA repository](https://github.com/InditexTech/cerbia) and [architecture / scanner documentation](https://inditextech.github.io/cerbia/latest/main/components/scanners/)
- [CerbIA configurable gate behavior](https://inditextech.github.io/cerbia/latest/main/gate/)

- [llm-d cache-aware vLLM routing](./llm_d_cache_aware_vllm_routing.md)
- [Path-multiplexed vLLM replicas](./llm_d_path_multiplexed_vllm_replicas.md)
- [Portal agent tool execution](./portal_agent_tool_execution_architecture.md)
- [OpenClaw / Copilot agent automation](./openclaw_copilot_agents_automation_architecture.md)

**Bottom line:** **Detect every new relevant trust-boundary input cheaply, retrieve reviewed precedents when uncertain, enforce real permissions at action boundaries, and improve a compact guardian only with verified feedback.**
