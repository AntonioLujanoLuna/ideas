# Agents Portal: Self-Hosted PII Detection, Pseudonymization and Document Redaction Service

**Status:** Architecture proposal and evaluation plan, not implemented or benchmarked  
**Updated:** 2026-11-12  
**Scope:** Existing host-based Agents Portal, LangGraph and GitHub Copilot SDK agents, incoming user text, agent/tool messages, RAG ingestion/retrieval, and uploaded TXT/JSON/HTML/DOCX/PDF/image documents.  
**Primary constraints:** Fully self-hosted processing; no third-party PII detection or document-processing APIs; fast common path; reliable protection; Spanish data (DNI/NIE, names, addresses, IBAN) and multilingual content; 25+ page PDFs, both searchable and scanned.  
**Security principle:** PII protection is an enforced gateway/security boundary, **not** an optional tool the agent decides whether to invoke.

**Related designs:**
- [Portal Agent Tool Execution Architecture](portal_agent_tool_execution_architecture.md)
- [Agent Runtime Isolation and Remote Workers](agent_runtime_isolation_and_remote_workers.md)
- [Agent Portal: Presentation Generation and Editing with Self-Hosted MicroVM Tools](agent_portal_presentation_generation_editing_microvm.md)
- [Excel RAG: Structure-Aware Elasticsearch Search](excel_rag_structure_aware_elasticsearch_search.md)
- [Agent Portal k6 Load and Workflow Testing](agent_portal_k6_load_and_workflow_testing.md)

## 1. Executive recommendation

Build a small internal **PII Gateway** exposing synchronous text analysis/redaction and asynchronous document processing. Start with:

1. **Microsoft Presidio** for recognition/anonymization orchestration and pluggable entity recognizers.
2. **Validated deterministic recognizers** for DNI/NIE/NIF, IBAN, payment-card numbers, email, phone numbers, IP addresses, tax IDs, configurable employee/customer IDs and organization-specific formats.
3. **GLiNER2-PII** as the first local multilingual contextual detector to benchmark for names, addresses and less-structured PII. Compare with spaCy/Stanza NER, Presidio's NLP recognizers and OpenAI Privacy Filter before production selection.
4. **PyMuPDF** for fast native PDF text extraction, page geometry and physical PDF redaction, subject to AGPL/commercial licensing review. Consider a permissively licensed alternative if distribution/licensing is incompatible.
5. **PaddleOCR** and/or **Tesseract** for scanned pages and embedded image regions, with OCR invoked only where text extraction is insufficient.
6. **Existing Docling infrastructure**, if available, as an optional fallback for complicated tables/layouts rather than the default parser for every PDF.
7. **Isolated document-processing workers**, strict verification, fail-closed policies and metrics. Avoid using the portal's production H100/H200 vLLM endpoints for routine PII detection.

**Do not send each page of a 25-page PDF as an image to an LLM.** Native PDFs can be parsed as text, and scanned PDFs should be processed with local OCR plus small PII models. An LLM may be a *policy-approved fallback* for ambiguous fragments, never the primary detector or the sole verification mechanism.

There are two different deliverables:

- **Sanitized content for agents:** extract text or structure, identify PII, replace sensitive spans, return safe text/chunks. This is usually the fastest path and does **not** require building a modified PDF.
- **Safe document for download/sharing:** remove PII from the actual binary document, sanitize hidden channels and verify the resulting artifact. This is harder and has different reliability requirements.

## 2. Goals, assumptions and non-goals

### Goals

- Protect user uploads, messages, tool results, retrieved context, RAG chunks, and agent-visible document content.
- Support **masking**, **consistent pseudonymization**, and **irreversible document redaction** as distinct operations.
- Handle PDF native text, scanned images, mixed text/image pages, Word documents, and common textual formats.
- Preserve useful non-PII context, reading order, provenance and document/page locations where possible.
- Provide deterministic, independently testable security policies: different entity types, tenants, workflows, destinations, and clearance levels.
- Default to local CPU inference or a modest dedicated inference worker; scale OCR independently.
- Return explicit partial/uncertain statuses; never silently leak source content on extraction/verification failure.
- Retain only what is needed for audit and authorized restoration, without recording raw sensitive values in ordinary logs.

### Non-goals for initial release

- Perfect recall or a claim that ML/NER detects every form of personal information.
- Pixel-perfect editing of all Word/PDF features.
- Automatic processing of all languages, handwriting, barcodes, faces, signatures and photographs with equal accuracy.
- Full semantic reasoning over private facts with general-purpose LLMs on the default path.
- A new agents framework, a mandatory MCP server, or per-request microVMs for short text calls.
- Allowing an agent to override the PII policy or request the unredacted original by changing its prompt.

### Initial policy scope to agree with security/legal

**PII is not the same as confidential information.** Decide whether the policy also covers corporate secrets, credentials, contracts, protected health data, financial details, faces/signatures, QR/barcodes, etc. Identify data that must be *blocked*, not merely replaced. GDPR pseudonymization is not anonymization; pseudonymized data may still be personal data.

## 3. Reference architecture

\`\`\`mermaid
flowchart TB
    UI[Portal UI / upload API] --> ING[Ingestion boundary]
    TOOLS[Tool responses / external documents] --> ING
    RAG[Retrieved records / chunks] --> ING
    ING --> AUTH[Identity, tenant, policy, file safety checks]
    AUTH --> ROUTE{Input type}
    ROUTE -->|Text / JSON / tool result| TEXT[Text normalization + structure-aware parsing]
    ROUTE -->|PDF / DOCX / image| DOC[Document parser and page/region classifier]
    DOC -->|Text layer| TEXT
    DOC -->|Scan or embedded image| OCR[Isolated OCR workers]
    OCR --> TEXT
    TEXT --> RULES[Presidio + deterministic recognizers and checksums]
    TEXT --> NER[Local contextual PII model]
    RULES --> FUSE[Span reconciliation + entity consistency + policy]
    NER --> FUSE
    FUSE -->|Ambiguous only, approved| FALLBACK[Optional local LLM or human review]
    FALLBACK --> FUSE
    FUSE -->|Agent context| MASK[Text masking / scoped pseudonyms]
    FUSE -->|Export artifact| REDACT[Physical redaction / reconstruction]
    REDACT --> VERIFY[Binary and visual output verification]
    MASK --> OUT[Authorized sanitized content]
    VERIFY -->|Pass| OUT
    VERIFY -->|Fail or uncertain| BLOCK[Quarantine / review / reject]
    FUSE -->|Extraction incomplete| BLOCK
    OUT --> AGENT[LangGraph / Copilot SDK / RAG store / download]
\`\`\`

**Integration placement:**
- Enforce before user input enters an LLM request and before documents are indexed in a shared RAG collection.
- Sanitize tool responses and external sources before they enter agent context.
- Treat already indexed unredacted content as a migration problem: do not assume applying a new ingress filter retroactively removes it.
- If sanitizing outbound data as well, do not pass unsafe tool-call parameters, URLs or requests to external providers.
- Preserve a policy-gated path for explicitly authorized operations requiring raw data (e.g. form submission). The general agent does not receive unrestricted mappings or decryption rights.

The PII service is a **native, internal HTTP API** called by portal middleware. Exposing an optional, self-hosted MCP adapter to agents is possible but **not** the security boundary; agents must not be able to bypass middleware. No externally hosted MCP.

## 4. Detection model: rules + NER, not one model alone

### 4.1 Deterministic recognizers

Create a recognizer registry with versioned patterns, multilingual contexts and validators:

| Entity | Technique | Failure considerations |
| --- | --- | --- |
| Spanish DNI/NIF/NIE | Spain-specific patterns + control-letter validation | OCR confuses O/0, I/1, B/8; malformed but plausible strings should be flagged |
| IBAN, including ES | ISO 13616 formatting + mod-97 checksum | OCR whitespace/hyphen insertion and line breaks |
| Credit/debit card | Candidate patterns + Luhn + contextual clues | False positives: numeric sequences with valid Luhn |
| Email, phone, IP | Patterns + country normalization and context | False positives and phone formatting variants |
| Passport, account/employee/customer IDs | Jurisdiction-specific rules and context | No globally reliable checksum for every passport format |
| URLs and secrets | Separate configurable recognizers | Credential detection is not equivalent to PII detection |
| Local names, addresses, dates of birth | Contextual NER + rules | Ambiguity and indirect reference resolution |

Prefer recall-oriented candidate generation, then use validators to assign confidence. **Do not discard all checksum-invalid values**: degraded OCR may corrupt one character of a genuine DNI/IBAN.

Presidio provides a practical analyzer/recognizer/anonymizer framework; configure its language-specific NLP engine and custom Spanish domain recognizers explicitly. Do not assume out-of-box Spanish performance is adequate.

### 4.2 Contextual recognizers

Benchmark:
- **GLiNER2-PII**: first-choice small multilingual contextual model, with entity categories configured to our policy; check official artifacts, model revision, runtime and license before deployment.
- **OpenAI Privacy Filter**: alternative open-weight local PII detector for contextual detection; review its entity taxonomy, runtime requirements and output semantics.
- **spaCy/Stanza models and Presidio NLP recognizers**: baseline, especially for Spanish.
- **Optional specialized LLM**: only for ambiguous spans/pages under strict resource budgets and an explicit data-handling policy.

Avoid calling a general LLM on every input. An LLM cannot certify that a document contains no PII.

### 4.3 Ensemble/fusion

Normalize all outputs to one span schema and reconcile overlapping results:
- Validated structured identifiers generally take precedence over ambiguous NER spans.
- Keep the union of high-confidence detections for safety, but define overlap and conflict handling explicitly so names inside addresses are not replaced twice.
- Evaluate model-only candidates by policy-specific thresholds, context and entity type.
- Search for **document-wide repeated identifiers** and safe, contextually supported aliases (e.g. full name later shortened). Do not blindly redact every use of a common first name without context.
- Distinguish **unknown** from **no PII detected**. Missing/failed extraction is never a clean scan.

Example:

\`\`\`text
Original:
María García López, DNI 12345678Z, presentó una solicitud.

Agent-facing:
[PERSON_001], DNI [ES_DNI_001], presentó una solicitud.
\`\`\`

## 5. Document pipeline

### 5.1 Text, JSON, HTML, tool results

- Parse structurally first. For JSON, anonymize values without changing keys/schema or serialization validity unless policy explicitly masks selected keys.
- For HTML, parse text nodes and relevant attributes; remove scripts, unsafe embedded content and metadata when exporting a safe artifact.
- Preserve source offset mapping and Unicode normalization information. Text offsets should specify a single unit (e.g. Unicode code points) and be converted carefully when clients use UTF-16 offsets.
- Use overlapping text windows for model context but reconcile outputs to stable document-global offsets.
- For tables and structured output, preserve cell boundaries as metadata.

### 5.2 Native (searchable) PDF

Fast path:
1. Validate file and PDF parser limits; inspect encryption and page count.
2. Extract text and positions using PyMuPDF (page, block, line, word/character boxes).
3. Detect PII on **reading-order-normalized text**, retaining a mapping back to glyph positions.
4. For agent ingestion, return sanitized text/chunks with page provenance.
5. For document export, map spans to page geometry, apply **actual redaction**, regenerate/clean the PDF and verify. Black visual overlays without content removal are prohibited.

Caveats:
- Multiple columns, tables, ligatures, rotated text, overlapping glyphs and hidden text can make character-to-box mapping nontrivial.
- Some PDFs have an OCR text layer over a scanned image. Redacting the text layer alone leaves the scanned pixels intact.
- Embedded images, annotations, forms/AcroForm/XFA, attachments, links, metadata, JavaScript and incremental revisions can retain information.
- Use a full rewrite/sanitization process and inspect residual objects. Test PyMuPDF's redaction handling for both text and image regions.
- Check PyMuPDF licensing; AGPL/commercial obligations may affect this architecture.

### 5.3 Scanned PDF / images

1. Determine whether text is available **per page and per region**. A textual footer does not make the scanned body safe.
2. Rasterize at a tested OCR resolution and limit pixel count.
3. Run local OCR with words/lines/polygons and confidence. PaddleOCR first, Tesseract as CPU comparison.
4. Run the same PII engine on normalized OCR text with geometry mapping.
5. Union adjacent boxes for multiword names, split IDs and hyphenated/line-wrapped values. Add padding where useful.
6. For safe export, redact the original image pixels or reconstruct from sanitized raster pages.
7. If OCR is uncertain or misses regions, **escalate or fail closed**, not silently accept.

For a 25-page scan, run pages in bounded parallel OCR batches. Do not send all pages as images to an LLM. Optical detection of handwriting, ID card photographs, face/signature content, QR/barcodes and hidden personal details requires separate models or explicit unsupported-content handling.

### 5.4 Mixed PDFs and complex layouts

Classify **regions**, not only whole files. Some pages contain both digitally drawn text and scanned image blocks. Deduplicate OCR/native overlapping detections while ensuring both pixel and text representations are protected. Keep Docling available when simple parsing loses reading order/tables, but benchmark its overhead before making it mandatory.

### 5.5 DOCX / Word

DOCX is an OOXML ZIP package and can hold sensitive data beyond visible paragraph text:
- Runs split across formatting boundaries, nested tables, headers/footers, comments, footnotes/endnotes, tracked changes, hidden text, field codes and hyperlinks.
- Text boxes/shapes, document properties, embedded images/OLE objects, alternative text, custom XML, macros and other relationships.

For **agent context**, extract supported visible and relevant hidden text, mask it before exposure. For **safe files**, choose between:
- **Initial safer choice:** generate a sanitized PDF or normalized text/Markdown from vetted content. This changes document fidelity.
- **Later editable DOCX:** edit every relevant OOXML part, strip unsupported/hidden components and reconstruct the package. Build fidelity and leakage regression tests before enabling.

Do not claim \`python-docx\` alone performs comprehensive Word sanitization. DOC/DOCM, password-protected files and embedded objects can initially be rejected, converted in a tightly isolated worker after policy approval, or routed for review.

## 6. Redaction, pseudonymization and restoration

Use three distinct modes:

| Mode | Example | Reversible? | Typical destination |
| --- | --- | --- | --- |
| Mask | \`****5678\` / \`[IBAN]\` | No mapping needed | Logs, some previews |
| Stable pseudonym | \`[PERSON_001]\`, \`[IBAN_001]\` | Only with protected mapping | Agent conversations and approved workflows |
| Physical document redaction | Text/image removed from released artifact | No | Downloads, exports and external sharing |

**Default agent behavior:** prefer stable per-session pseudonyms. Tokens should be non-sensitive opaque IDs, not hashes of low-entropy personal values. A name can map to the same ID across a document or authorized session without exposing the original.

**Restoration:** keep a separate encrypted, access-controlled mapping vault scoped by tenant, principal, conversation/workflow and TTL. Restoration requires a server-side policy check for a specific authorized operation. Do not send the mapping, originals or vault keys to the agent. Avoid leaking through errors, traces, telemetry, cache keys or filenames.

**Important:** Replacing a name with a placeholder does not guarantee anonymity if remaining context uniquely identifies the person.

### 6.1 Physical PDF output policies

1. **Standard:** apply PDF content removal/redaction; scrub attachments, forms, annotations and metadata; serialize to a new sanitized artifact; then verify.
2. **High assurance:** render fully sanitized page images to a *new* PDF, optionally with a **freshly generated safe OCR layer**. This sacrifices editability, indexing fidelity, accessibility and size; it does not compensate for undetected PII in pixels.
3. **Unable to establish safety:** quarantine/reject, or request review. Do not export with a misleading "redacted" status.

## 7. Proposed HTTP API

Suggested native REST endpoints:

\`\`\`text
POST /v1/pii/analyze
POST /v1/pii/redact/text
POST /v1/pii/redact/document
GET  /v1/pii/jobs/{job_id}
GET  /v1/pii/jobs/{job_id}/artifact
DELETE /v1/pii/jobs/{job_id}
\`\`\`

All endpoints must authenticate caller/service identity, authorize tenant/workflow policy and impose size, time, CPU/memory and document-page limits. Large file inputs should use approved internal object-storage references or multipart uploads, not unbounded JSON/base64. Do not expose raw values in API errors.

Example text request:

\`\`\`json
{
  "text": "María García López, DNI 12345678Z.",
  "language_hints": ["es"],
  "policy_id": "agent-context-default",
  "mode": "pseudonymize",
  "scope_id": "conversation-scoped-opaque-id"
}
\`\`\`

Example (illustrative) response:

\`\`\`json
{
  "status": "completed",
  "sanitized_text": "[PERSON_001], DNI [ES_DNI_001].",
  "findings": [
    {"type": "PERSON", "start": 0, "end": 18, "source": "ner", "confidence": 0.94, "replacement": "[PERSON_001]"},
    {"type": "ES_DNI", "start": 24, "end": 33, "source": "validated_rule", "confidence": 0.99, "replacement": "[ES_DNI_001]"}
  ],
  "policy_version": "example-v1",
  "mapping_reference": "opaque-server-side-handle"
}
\`\`\`

The confidence values are illustrative, **not calibrated scores or measured model performance**. In production, do not reveal original entity values in findings by default. For document mode, findings add page and geometry and the response may return an asynchronous \`job_id\`. Include \`failed\`, \`partial\`, \`needs_review\` and \`verified\` states; \`partial\` must never be treated as safe for agent exposure under a strict policy.

### Internal data model

- \`document_id\`, \`tenant_id\`, \`policy_version\`, \`input_format\`
- \`page\`, \`element_id\`, \`text_range\`, \`coordinate_system\`, \`bounding_boxes\`
- \`entity_type\`, \`source\` (rule/NER/OCR/manual), \`confidence\`, \`validation_result\`
- \`replacement\`, \`scope_id\`, \`mapping_handle\`
- \`extraction_coverage\`, \`verification_status\`, \`failure_reasons\`
- Request/job timing, stage versioning, model/rule hashes and telemetry **without raw PII**

Avoid persisting raw matches in the normal analytical datastore.

## 8. Integration with the existing portal

### Ingress and chat

- In the FastAPI/backend request path, sanitize user text before assembling LangGraph/Copilot SDK LLM messages.
- Maintain a canonical sanitized conversation view. Raw input, if business requirements require retention, stays outside the standard agent state.
- Prevent alternate client/upload routes from bypassing the gateway.
- Agent tools should receive safe context by default. Where tools truly need original data, require narrow, audited server-side restoration.

### Document upload and RAG

- Scan and sanitize **before** chunking, embedding or indexing into the general RAG store. Vector embeddings and search snippets can leak personal data if built from originals.
- Preserve source page and element metadata for citations, without retaining raw sensitive strings in metadata.
- Retrieval-time enforcement remains necessary for legacy indexes, access-controlled originals and tool-specific datasets. Do not rely only on ingestion filtering.
- Plan migration/reindexing for old unsanitized vectors and document caches.

### Tool responses, traces and outbound traffic

- Sanitize potentially untrusted external tool responses **before** their content becomes an LLM observation or a user-visible artifact.
- Apply output policy to documents the agent creates or modifies.
- Use structured logging with explicit allowlisted fields; default to redacted traces and disable raw prompt/response capture where required.
- Agent-mediated web searches, API calls and file exports should pass through egress policy so names, IDs and bank details do not leak via query strings, headers, prompts or body content.
- Protect machine-generated intermediate artifacts, temp files and crash dumps.

### Deployment

The portal's orchestration currently runs on the host. Put the PII Gateway in an independent self-hosted service/process with isolated parser/OCR workers. **Do not require MCP** and **do not put an LLM in the critical path**.

Document parsers process untrusted binaries: use restrictive isolation (microVMs where the portal's approved execution architecture requires it, or another approved sandbox with similar boundaries), file quotas, no public network access, resource caps and patched libraries. Short text-only inference can be a persistent low-overhead service. Reuse the platform's microVM/tool sandbox design when practical rather than inventing a second execution system.

The goal is to avoid consuming the existing H100/H200 vLLM replicas for routine PII. Profile GLiNER2 and OCR on available CPU, then evaluate a small separate GPU if throughput/latency warrants it.

## 9. Reliability model and explicit fail-closed semantics

No single detector can guarantee zero misses. Treat protection as layered risk reduction plus verification:

1. **Extraction coverage first:** classify every page, image, layer, embedded object and unsupported component. Missing pages/unknown regions cause \`needs_review\`, not \`no_pii\`.
2. **Rules + contextual NER:** ensemble and retain provenance for every entity.
3. **Document-level consistency:** find repeated names, accounts, identifiers and safe aliases after initial recognition.
4. **Independent second pass:** reparse produced text/artifacts and scan for residual identifiers with different rules and, where possible, a separate detector. Verify redacted pixels on relevant pages and inspect embedded objects.
5. **Output artifact audit:** no retained source text underneath overlays; no hidden OCR layer leaking originals; no metadata, comments, annotations, attachments or OOXML revisions containing PII.
6. **Human escalation:** unsupported forms, uncertain OCR, low-quality scans, high-risk categories or failed verification go to authorized reviewers only. Maintain review access logs.
7. **Versioned policies and explicit approval:** thresholds and fail-open/fail-closed choices must be audited, not buried in agent prompts.

Suggested policy levels:

| Policy | Behavior |
| --- | --- |
| \`agent-context-default\` | Pseudonymize configured PII; fail closed on incomplete extraction |
| \`public-export-strict\` | Strong physical redaction, metadata stripping, independent verification, review when uncertain |
| \`structured-analytics\` | Minimize or irreversibly aggregate fields rather than per-record reversible pseudonyms |
| \`authorized-sensitive-operation\` | Restricted, audited access to specific values needed by a trusted action |

**Privacy review required:** legal basis, storage/retention, records of processing, transfer constraints and data subject rights under GDPR. This design is engineering guidance, not a legal certification.

## 10. Performance and measurement

Do not assign fixed latency numbers before representative benchmarks. Separate these stages:

- Request parsing/auth/policy
- Text extraction and geometry
- OCR (per page, region and resolution)
- Deterministic recognizers
- ML inference, batching and model warmup
- Span merge/document consistency
- Redaction/reconstruction
- Verification and artifact storage

**Expected behavior:**
- Small text: low-latency synchronous path with resident recognizers and NER model.
- Searchable PDF: direct extraction plus detection, usually without any page-image inference.
- Scanned PDF: OCR dominates, so use bounded parallel pages and asynchronous jobs.
- Mixed PDF: region-selective OCR, with attention to overlapping text layers.
- DOCX: parsing and content enumeration are often cheap; ensuring safe editable output is the more complex step.

Design concurrency using bounded queues, per-tenant admission control, OCR CPU/GPU pools, request budgets and cancellation. Batch NER over short windows, but never combine text from different tenants into a cross-tenant trace/cache. Consider model warmup and pinned revisions. Avoid duplicate processing with content-addressed jobs only when caching is explicitly permitted and tenant-isolated.

### Benchmark corpus

Create a **consented/synthetic** evaluation suite that contains:
- Spanish DNI/NIE, IBAN, foreign IDs, names, addresses, DOBs, phones, emails, account numbers and ambiguous numeric strings.
- English/Spanish mixed documents, legal/HR/finance contexts, invoices and letters.
- Native PDFs, scanned PDFs, OCR-overlay PDFs, mixed image/text pages and long 25-page documents.
- Low DPI, rotated, warped, noisy and multi-column scans; tables and split-line identifiers.
- DOCX comments, tracked changes, headers, footers, hyperlinks, text boxes, metadata, embedded images, and unsupported objects.
- Adversarial cases with overlapping boxes, white-on-white text, invisible layers, embedded attachments and pixels that still contain the original identifier.
- Tests that assess both **recall** and **overmasking**, including non-person references that resemble PII.

Collect:
- Entity recall/precision/F1 by type, language, format, scan quality and model.
- **Complete-document leak rate**: fraction of documents with one or more residual protected entities. This is often more important than average token-level F1.
- OCR character/word error rate, extraction coverage and mapping correctness.
- p50/p95/p99 end-to-end latency by stage, pages/sec, memory, CPU/GPU utilization and queue wait.
- Binary leakage testing: recoverability from text extraction, PDF internals, OCR layers, OOXML, metadata and attachments.
- Operator-review rate, false-positive interruption rate and unit cost/resource per page.

Do not publish the system as "100% PII safe" merely because one benchmark is high. Define production acceptance thresholds with the security owner; stricter policies should reject uncertain documents rather than return unchecked content.

### k6 integration

Use the repository's [Agents Portal k6 Load and Workflow Testing](agent_portal_k6_load_and_workflow_testing.md) plan for HTTP and workflow load: text bursts, concurrent uploads, mixed native/scanned documents, request cancellation, queue saturation, retries and authorized agent flows. k6 is for throughput/latency, **not** semantic redaction correctness; pair it with a document gold-set evaluator and binary artifact leak tests.

## 11. Implementation phases

### Phase 0: Scope and golden corpus

- Approve categories, severity, output modes and retention policy.
- Enumerate all ingress/egress paths in the portal.
- Build ground-truth data, including Spanish administrative documents and adversarial PDF/DOCX artifacts.
- Decide licensing constraints and permitted compute/isolation approach.

**Exit criterion:** agreed policy, evaluation harness and source-to-output threat model.

### Phase 1: Text service and fast path

- Implement \`/analyze\` and \`/redact/text\` with Presidio, validators, GLiNER2 candidate and stable pseudonyms.
- Benchmark GLiNER2 against alternatives on actual languages/entities.
- Add native searchable-PDF extraction for agent context and basic DOCX-to-safe-text.
- Put the API between backend and both agent frameworks, instrumented with safe metrics.

**Exit criterion:** representative messages and native documents sanitized, with no route around the gateway and measured latency/recall.

### Phase 2: Scans and secure PDF exports

- Add per-page/per-region scan detection and local OCR worker queue.
- Restore word/polygon geometry and reconcile OCR/native text.
- Add actual PDF redaction, metadata/object scrubbing, high-assurance raster export and independent verification.
- Add asynchronous document jobs and review/quarantine statuses.

**Exit criterion:** scanned/mixed PDFs processed end-to-end without recoverable redacted text on the attack corpus; unsupported files fail closed.

### Phase 3: Hardening and expanded formats

- Support more OOXML parts and optional editable DOCX export only after comprehensive leak/fidelity tests.
- Add entity consistency, language specialization, operator review workflow, versioned policies and tenant-specific validators.
- Integrate RAG reindexing, tool-response scanning, egress checks, tracing restrictions and audit reporting.
- Calibrate thresholds and capacity with sustained k6 + semantic eval load.

**Exit criterion:** security signoff, documented residual risks, service SLOs, rollback plan and production runbook.

## 12. Key decisions to resolve

1. **Threat model:** Is the requirement to keep PII from LLMs, from end users, from external integrations, from logs, or all of them? These require different enforcement boundaries.
2. **Mode:** Is per-conversation reversible pseudonymization needed, or irreversible redaction sufficient for most flows?
3. **Output formats:** Is safe *agent-readable text* sufficient at first, or must users download a redacted PDF/DOCX?
4. **Review:** Are user-facing "document needs review" states acceptable, and who can access originals?
5. **Coverage:** Must the service detect handwritten data, signatures, photos, ID cards, QR/barcodes, hidden text and attachments?
6. **Hardware:** Can CPU achieve acceptable OCR/NER performance, or should we provision a small dedicated GPU?
7. **Licensing:** Is PyMuPDF's AGPL/commercial licensing acceptable, or do we need to substitute another PDF stack?
8. **Data lifecycle:** How long may original uploads and reversible mappings exist, and where will those keys be held?

## 13. Useful official references

- Presidio: https://github.com/microsoft/presidio
- Presidio anonymizer: https://microsoft.github.io/presidio/anonymizer/
- GLiNER / GLiNER2 ecosystem: https://github.com/urchade/GLiNER
- PyMuPDF redaction documentation: https://pymupdf.readthedocs.io/en/latest/page.html#Page.apply_redactions
- PyMuPDF licensing: https://pymupdf.readthedocs.io/en/latest/about.html
- PaddleOCR: https://github.com/PaddlePaddle/PaddleOCR
- Tesseract OCR: https://github.com/tesseract-ocr/tesseract
- OCRmyPDF: https://ocrmypdf.readthedocs.io/
- Docling: https://github.com/docling-project/docling
- python-docx: https://python-docx.readthedocs.io/
- EU GDPR: https://eur-lex.europa.eu/eli/reg/2016/679/oj

**Bottom line:** The best starting architecture is a mandatory, self-hosted **Presidio + validated recognizers + local NER** gateway, plus **native document extraction and selective OCR**, backed by safe artifact generation and independent verification. Treat OCR, document reconstruction and LLM-based fallback as conditional workloads rather than the default processing path.
