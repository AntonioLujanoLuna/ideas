# Agent Portal: Presentation Generation and Editing with Self-Hosted MicroVM Tools

**Status:** Design proposal and evaluation plan, not implemented or benchmarked  
**Updated:** 2026-11-06  
**Scope:** Existing host-based FastAPI Agents Portal with LangGraph and GitHub Copilot SDK agents; user-requested creation, modification, preview, and export of slides and PowerPoint decks.  
**Decision constraints:** No container execution infrastructure currently available; agent-controlled software must execute in microVMs rather than on the backend host. Any MCP server must run on infrastructure operated by us; third-party remote MCP endpoints are prohibited.

**Related designs:**
- [Portal Agent Tool Execution Architecture](portal_agent_tool_execution_architecture.md)
- [Ultra-light MicroVM Agent Tool Sandboxes PoC](ultralight_microvm_agent_tool_sandboxes_poc.md)
- [Isolated and Distributed Agent Runtime](agent_runtime_isolation_and_remote_workers.md)
- [Browser Agent Architecture](browser_agent_architecture.md)

## 1. Executive decision

**Keep agent orchestration on the existing host. Run presentation tool execution in isolated, organization-controlled microVM workers, using the portal's existing framework-independent tool router and sandbox execution contract.**

Do not move LangGraph, the Copilot SDK, the LLM gateway, or conversation state into every guest. Do not build a separate microVM platform for slides. Extend the general-purpose sandbox infrastructure to support presentation workloads.

Support two complementary authoring paths:

1. **Web-native slide decks**: Slidev first; Reveal.js or Marp where appropriate. Markdown and assets remain the editable source. Output may include HTML, PDF, PNG, or PPTX. Slidev's editable PPTX conversion needs fidelity testing.
2. **PowerPoint-native decks**: Office Kit for inspecting/editing existing PPTX documents, PptxGenJS as a generation-only fallback. Evaluate OpenPresentation (OPF) for a structured, machine-editable canonical deck format.

Expose stable high-level tools for ordinary requests, and enable a tightly scoped command/file interface for more complex agent-directed authoring. Both can execute in microVMs. **A microVM is not technically required by an SDK or MCP protocol, but it is the isolation mechanism selected for this portal because neither containers nor arbitrary tool processes on the backend host are acceptable execution backends.**

**Important correction to existing PoC:** [ultralight_microvm_agent_tool_sandboxes_poc.md](ultralight_microvm_agent_tool_sandboxes_poc.md) starts with Podman + krun. That is a valid future comparison but **not an immediately usable implementation path under the present no-container-infrastructure constraint**. For an initial container-independent PoC, assess direct Firecracker + KVM (or Cloud Hypervisor), subject to host access, nested virtualization, security approval, and the engineering work described below. Do not present a direct Firecracker PoC as already deployed.

## 2. Goals and non-goals

### Goals

- Generate useful, visually coherent slide decks from prompts, sources, datasets, and approved templates.
- Import and modify existing PPTX files while preserving unrelated content as far as the selected library permits.
- Support targeted operations: replace a title, update a chart, reorder slides, insert a slide, adjust theme, or replace images.
- Render previews and validate final PPTX/PDF/HTML artifacts before returning them.
- Reopen a saved deck for continued editing, with source files and revisions preserved.
- Reuse the existing tool router, authorization, artifact storage, tracing, user/session isolation, and microVM execution contract.
- Keep all tool execution, MCP servers, dependencies, fonts, renderers, and conversion software within infrastructure controlled by the organization.

### Non-goals for the initial release

- Pixel-perfect import/edit/export of every third-party PowerPoint feature.
- Full browser-based collaborative PowerPoint editing from day one.
- Creating a new agent SDK, orchestration framework, or cluster scheduler.
- Exposing guest MCP services directly to users or to the public internet.
- Giving microVMs unrestricted corporate network access, package registry access, or host filesystem mounts.

## 3. Technology shortlist

| Technology | Primary role | Existing PPTX input | Agent integration | Main caveat |
| --- | --- | --- | --- | --- |
| [Slidev](https://sli.dev/) | Markdown/Vue web-native slides | Not native editing | CLI, official skills, built-in stdio/HTTP MCP | Editable PPTX export cannot faithfully represent all web components |
| [Reveal.js](https://revealjs.com/) | HTML-based interactive presentations | Not native editing | Files and CLI / custom wrappers | No general native PowerPoint import/edit path |
| [Marp](https://marp.app/) | Simple Markdown-to-slide workflow | Not native editing | CLI / Markdown skill | Less expressive layout; editable PPTX conversion has limitations |
| [PptxGenJS](https://github.com/gitbrent/PptxGenJS) | New editable PPTX generation | No | Node SDK | Generation only; cannot import and change arbitrary existing decks |
| [Office Kit PPTX](https://github.com/office-kit/pptx) | Read/modify/write OOXML PPTX | Yes | TypeScript SDK, TSX authoring skill, preview packages | Pre-1.0; require round-trip/fidelity regression testing |
| [OpenPresentation / OPF](https://www.openpresentation.org/) | Structured JSON-based authoring/editing | Import supported, verify fidelity | SDK, CLI, local MCP, six skills, editor | Young 0.x ecosystem; source format is OPF, not PPTX |
| [Presenton](https://github.com/presenton/presenton) | Self-hosted AI presentation application | Verify import/edit scope separately | Internal REST API | Larger independent application to operate and integrate |
| [PPTist](https://github.com/pipipi-pikachu/PPTist) | Browser slide editor | Yes, with limitations | Frontend integration | AGPL licensing and round-trip fidelity require review |
| [ONLYOFFICE Docs](https://github.com/ONLYOFFICE/DocumentServer) | Full Office-like collaborative editor | Yes | Document server and editor APIs | Operational footprint and licensing; not the initial sandbox path |

**Initial evaluation order:**
- **Slidev** for agent-authored web slides, because the official CLI, skills and local MCP tools are well aligned with a file-based workspace.
- **Office Kit** for selective edits to uploaded PPTX documents, with **PptxGenJS** retained for mature new-PPTX generation.
- **OPF** as an architecture experiment if a validated JSON document, JSON Patch operations, embeddable editor, and durable source revisions are important.
- Avoid launching Presenton/ONLYOFFICE in the first PoC unless the requirement changes from tools to a complete end-user presentation application.

These are **candidates**, not verified production selections. Compare them using the same sample decks and fidelity metrics.

## 4. Skills, libraries, and MCP are distinct

A **skill** is instructions, conventions, and sometimes helper scripts that guide an agent. A **library/CLI** performs actual document operations. **MCP** is one possible tool transport, not a requirement for agent interoperability.

Potential reusable agent instructions:

- [Slidev official AI skills](https://sli.dev/guide/work-with-ai), paired with its installed CLI and optional local MCP server.
- [Office Kit PPTX skill](https://github.com/office-kit/pptx/blob/main/skill/SKILL.md) for typed TSX authoring and preview-based revisions.
- [OPF agent skills](https://www.openpresentation.org/docs/agent-skills) for authoring, layouts, presets, inspection, editing and export.
- [OpenAI skills repository](https://github.com/openai/skills) for presentation authoring and visual QA patterns, subject to checking the particular skill and its licensing/dependencies.
- [Anthropic PPTX skill](https://github.com/anthropics/skills/tree/main/skills/pptx) as a workflow reference, **not** an assumed freely redistributable dependency; inspect the license and any restrictions before reuse.

**MCP policy:** all MCP servers used by portal agents must be started and managed by our organization. **Never configure agents to connect to externally hosted or vendor-operated MCP servers.**

Suggested tool mechanism:

| Engine | Preferred first integration | MCP deployment |
| --- | --- | --- |
| Slidev | Guest CLI, files, optional guest-local MCP | Start installed Slidev MCP over stdio inside the session microVM |
| Office Kit | Guest TypeScript SDK invoked through controlled worker | No MCP needed |
| PptxGenJS | Guest Node SDK invoked through controlled worker | No MCP needed |
| OPF | Guest CLI/SDK; optional MCP for schema/validation | Self-hosted stdio MCP inside guest if beneficial |
| Reveal.js / Marp | Guest CLI/files | No MCP needed |
| Presenton | Only if selected later, an organization-operated internal service | No external MCP |

**Slidev-specific detail:** the official Slidev server supports a local stdio MCP that reads/edits slides, and its development server optionally exposes a local Streamable HTTP endpoint. Use stdio inside the guest where possible, rather than making its development endpoint reachable from other tenants.

**OPF-specific detail:** the local OPF MCP primarily supplies documentation, schema/catalog access, validation and text-format conversion. It is **not** a PPTX rendering/export service. Call the local CLI or SDK to produce PDF/PPTX/PNG/SVG.

Pin exact, internally approved library versions, package lockfiles, fonts, and guest image digests. Build/preinstall dependencies in approved images; **never allow an LLM tool call to fetch and execute latest packages from public registries at runtime**.

## 5. Deployment architecture

~~~text
                           ANGULAR PORTAL
                    uploads / preview / revisions
                                  |
                                  v
                     EXISTING FASTAPI HOST
               authentication + session + policies
                                  |
                 LangGraph / GitHub Copilot SDK
                                  |
                         Tool Registry/Router
                                  |
                  authorized presentation tool call
                                  |
                                  v
                  SandboxExecutionProvider (existing)
                                  |
                       MicroVM Controller
              allocation / lifecycle / resource policy
                                  |
               private command/artifact transport
                         e.g. virtio-vsock
                                  |
        +-------------------------+--------------------------+
        |                                                    |
        v                                                    v
  MICROVM PROFILE A                                    MICROVM PROFILE B
  document / light                                     render / browser
  - OPF SDK/CLI                                        - Node + Slidev
  - Office Kit SDK                                     - Chromium/Playwright
  - PptxGenJS                                          - LibreOffice if needed
  - local MCP optional                                 - local MCP optional
        |                                                    |
        +-------------------------+--------------------------+
                                  |
                            checked outputs
                                  |
                    SERVER-OWNED ARTIFACT STORAGE
           source JSON/MD/TSX + assets + PPTX/PDF + previews
~~~

The diagram depicts **logical worker profiles**, not necessarily separate codebases or permanent VM pools. Both profiles may initially be implemented using one base guest image and different resource limits.

Responsibilities:

- **Host:** authenticate the caller; enforce access to source documents; select a permitted tool; assign an authorized workspace; create jobs; store versions; review results; return artifact identifiers.
- **MicroVM manager:** boot/reuse an isolated guest; enforce CPU/RAM/time/disk quotas; transfer files and commands; track heartbeat/cancellation; clean up guests.
- **Guest supervisor:** run installed Node/Python tools and optionally stdio MCP processes; stream bounded logs; write artifacts to its workspace; exit with structured status.
- **Frontend:** upload source PPTX/Markdown; display slide thumbnails or HTML preview; show revision differences and error reports; allow downloads; request human approval for overwrites or publishing.

Do not give arbitrary guests a direct route to the LLM gateway, GPU nodes, database, host Unix sockets, portal tokens, user credentials, internal repositories, or private network by default.

## 6. Two modes, same microVM execution substrate

### Mode A: structured, bounded presentation operations

The agent requests a typed operation, for example:

~~~json
{
  "tool": "presentation_replace_text",
  "arguments": {
    "presentation_id": "deck-123",
    "expected_revision": 4,
    "slide_id": "s7",
    "element_id": "title",
    "new_text": "Revised outlook"
  }
}
~~~

The host validates identity, document access, revision, schema, and limits. It sends the authorized input and selected command to a microVM. A small **trusted guest application**, with Office Kit or OPF, performs the edit and returns a new artifact.

Advantages: narrow attack surface, deterministic validation, inexpensive agent reasoning, easier audit, minimal edits to existing decks. Even though the code is authored by us, it still processes user documents and executes **inside a microVM** under the current deployment policy.

### Mode B: agent-directed source/code editing

The agent creates or modifies Slidev Markdown, OPF JSON, or TypeScript/TSX project files and runs a permitted build/export command in a microVM.

Typical workflow:

1. Host allocates a session-scoped workspace and transfers only approved inputs.
2. Agent writes source files via the sandbox file API.
3. Agent invokes the installed Slidev/OPF/Office Kit command through the sandbox execution API.
4. Worker validates, builds, renders, and returns bounded logs and preview artifacts.
5. Agent inspects preview output and performs targeted fixes in the same workspace.
6. Host persists source, assets, outputs, validation results and revision metadata.

**Treat TSX/JavaScript, local package scripts, slide plugins, embedded fonts/assets, and browser rendering as potentially untrusted code.** More restrictive network/filesystem policy and a larger runtime profile are appropriate for this mode. Do not let an agent arbitrarily select a shell, image, mounts, or network policy without authorization.

### Guest reuse

Do not boot a new microVM for each slide, each tool invocation, or each agent turn. A **session-scoped guest** can process repeated commands within a bounded idle TTL and maximum lifetime. Do not reuse a dirty guest across users or tenants. Structured one-shot jobs may use short-lived guests when warm-session locality is unnecessary.

## 7. Proposed portal tool API

Keep names/framework adapters independent of the implementation engine:

| Tool | Purpose | Default execution mode |
| --- | --- | --- |
| presentation_create | Create from a brief, template, and source references | Structured or source-authoring VM |
| presentation_inspect | Return slides, element tree, notes, layout metadata, previews | Structured VM |
| presentation_edit | Apply validated targeted edits with revision precondition | Structured VM |
| presentation_render | Generate preview thumbnails and/or PDF | Rendering VM |
| presentation_export | Produce PPTX/PDF/HTML/PNG as supported by engine | Engine-specific VM |
| presentation_validate | Structural and visual checks; diagnostics | Structured + rendering VM |

Keep generic **sandbox.create / exec / write_file / read_file / list_files / terminate** at the underlying execution level, as already proposed in the microVM PoC. A presentation tool should typically be a controlled composition of these primitives.

Minimum request metadata: user/tenant/session IDs (server-derived), source artifact IDs, expected revision, desired target format, permitted engine/profile, job ID, timeout, output byte limit, and idempotency key. The server chooses the engine/profile; user prompts are not authority to change placement or permissions.

For **editing existing PPTX**: load the original source, apply only intended mutations, save a *new revision*, and record which slides/objects changed. Never overwrite the only copy. When import fidelity is insufficient, explain limitations instead of silently reconstructing and replacing unrelated slides.

## 8. Artifact and revision model

A presentation is a versioned object, not only a downloadable PPTX:

~~~text
presentation_id
  metadata: owner/tenant, permissions, title, format, timestamps
  revision N:
    engine + exact version
    canonical source type: PPTX | OPF JSON | Slidev Markdown | TSX
    original source artifact (immutable)
    editable source files + referenced assets
    exported artifact(s): PPTX, PDF, HTML, PNG
    preview thumbnails / visual snapshots
    validation report + fidelity warnings
    provenance / approved input references
    parent revision, changed slide IDs, digests
~~~

**No universal lossless canonical format exists across arbitrary HTML/Vue slides and all PPTX constructs.** Retain the native authoring source for each engine and treat conversions as derivative outputs. If OPF is selected as a canonical format for OPF-created decks, that does not imply arbitrary PowerPoint round trips become lossless.

Optimistic revisions prevent concurrent edits from silently overwriting each other. Keep immutable original uploads, limit retention according to corporate policy, and sanitize previews as untrusted content. Do not expose Slidev development servers or raw untrusted HTML to the main portal origin without a safe rendering/serving boundary.

## 9. Fidelity and quality gates

At a minimum, every successful job should report:

1. **Structural integrity**: PPTX is a well-formed ZIP/OOXML package; validate with an Open XML validator where available; detect missing relationships, media or broken references.
2. **Render success**: export every slide or every changed slide to an image using an approved backend, with known fonts.
3. **Visual checks**: detect clipping, overlapping elements, unreadable text, chart misalignment and insufficient contrast where practicable. Compare before/after renders for targeted edits.
4. **Content checks**: page/slide count, speaker notes, charts/tables, required claims, unsupported elements, and exact text replacements.
5. **Conversion warnings**: identify when editable objects were rasterized or when round-trip preservation is not guaranteed.
6. **Source reproducibility**: repeat the build with pinned dependencies and record hashes and tool versions.

Office Kit's preview renderer is useful for fast feedback, but it does not guarantee pixel-authoritative compatibility with PowerPoint. LibreOffice headless can provide an independent render/export check; test it rather than assuming perfect fidelity. Slidev's default PPTX is typically an image-based export; its *pptx-editable* path reconstructs editable elements with limitations. These are different deliverables and must be labeled accordingly.

## 10. Security and operational model

**VM isolation is necessary here as a deployment boundary, but not sufficient by itself.** Enforce:

- **Trusted host control plane**: the model cannot choose privileged guest profiles, launch unrestricted host commands, assign arbitrary mounts, or mint tool credentials.
- **Guest egress deny-by-default**: no external MCP endpoints; any approved image/assets retrieval happens through a host-controlled proxy/import pipeline. Block SSRF, metadata, management, and other-tenant networks at the host/VMM boundary.
- **No host secrets in guests**: do not pass gateway keys, SSH agents, host environment files, privileged sockets, or corporate proxy credentials.
- **Per-user workspaces**: normalize paths, reject symlink escapes, constrain workspace reads/writes and archive extraction, and cap total files/bytes.
- **Limits and lifecycle**: vCPU, memory, disk, process and wall-clock quotas; job cancellation, hard teardown, orphan cleanup, output truncation and audit trail.
- **Supply chain**: pinned versions, approved build-time dependencies, signed/checksummed images, font licensing and vulnerability scanning.
- **Untrusted file formats**: zip bombs, crafted OOXML/XML, embedded links/media, macros, HTML/JS and malformed fonts are hostile input even if the calling operation is structured.
- **Cross-tenant checks**: verify negative tests for workspace leaks, stale session reuse, artifact ownership, and mixed-user MCP state.
- **Transport**: authenticate the VM controller; use a bounded host-to-guest protocol, such as a guest supervisor over virtio-vsock, instead of exposing an unauthenticated SSH server or public MCP port.

Existing microVM note: Firecracker offers a VMM API, jailer, virtio-block and vsock, but not native virtio-fs sharing. Plan explicit guest disk and bounded artifact transfer. If using krun later, verify its **TSI networking behavior**: a superficial no-network option is not sufficient evidence of egress denial.

### Sizing

Do **not** assume the ultra-light shell profile will render PowerPoint or Slidev. Node, Chromium/Playwright, native fonts and LibreOffice materially increase memory/CPU/startup needs.

Initial *benchmark candidates only*:
- **Document operations**: 1–2 vCPU, 512 MiB–2 GiB RAM.
- **Browser/PDF/PPTX render operations**: 2–4 vCPU, 2–4 GiB RAM, with larger configurations if heavy decks require them.

These are starting hypotheses, **not measured requirements**. Measure cold-boot latency, warm reuse, peak guest memory, real host memory pressure, CPU, disk/IO, throughput, p95 job latency, output size and failure rate at 1/5/20 concurrent presentation jobs before deciding capacity.

## 11. Recommended phased PoC

**Phase 0: VM runtime prerequisite**
- Confirm KVM access and organization policy on the intended worker machine(s).
- Given no container infrastructure, prototype direct **Firecracker** with a read-only minimal rootfs, writable per-VM workspace and guest supervisor, reusing the general SandboxExecutionProvider interface.
- Verify command execution, file transfer, cancellation, hard teardown, deny-by-default egress, artifact ownership and tenant isolation.
- If direct Firecracker is unsuitable, evaluate Cloud Hypervisor. Revisit Podman+krun only when the stated infrastructure restriction changes.

**Phase 1: simplest presentation path**
- Preinstall pinned Node and **Slidev** in a presentation guest image.
- Generate a 5–10 slide deck from Markdown, then render PDF/PNG and optionally HTML.
- Test built-in Slidev **stdio MCP** from the guest supervisor, without external MCP connectivity.
- Persist Markdown/assets and rebuild the same deck deterministically.

**Phase 2: editable PPTX generation and modification**
- Add **Office Kit** and **PptxGenJS** to the guest and implement inspect/create/edit/export high-level tools.
- Test three decks: a new ten-slide deck with tables/charts, a corporate template with theme/master, and an imported deck with notes, hyperlinks and complex objects.
- Prove text/chart replacement and slide insertion without disturbing other slides. Validate with PowerPoint-compatible viewers and before/after render diffing.

**Phase 3: OPF and frontend experience**
- Compare OPF JSON + patch editing against Office Kit for repeated agent edits.
- Add an Angular deck workspace: slide thumbnails, selected-slide instructions, revision history, previews, download links and fidelity warnings.
- Explore OPF/Office Kit embeddable editors only after their maturity and licensing are verified.

**Phase 4: production hardening**
- Admission control, warm VM reuse, quotas, audit logs, metrics, repeated fuzz/negative tests, partial-output recovery, upgrade pinning, retention, and clean teardown.
- Scale guest resource profiles independently rather than overprovisioning every simple file operation for Chromium.

### Acceptance criteria for first useful release

- An agent produces a presentation and returns **source + PDF/PPTX + slide previews**.
- A user uploads an existing PPTX and gets a new revision with a **specific targeted edit**.
- Original artifact remains immutable and all emitted files belong to the correct user.
- Entire workflow runs without executing user/agent-generated code on the FastAPI host.
- No third-party remote MCP endpoint and no unrestricted guest egress are required.
- Failed or malformed jobs cannot access host secrets or another tenant's workspace.
- Render/structural checks run automatically; conversion limitations are reported rather than hidden.

## 12. Open decisions

1. **VM provider**: is KVM usable on the actual machines, and will direct Firecracker guest management be approved? The old Podman-first PoC is not directly runnable under today's restriction.
2. **Product priority**: is generating new decks more urgent than importing/precisely modifying existing company PPTX templates?
3. **Canonical editing source**: preserve per-engine source initially, or adopt OPF for new decks after a realistic round-trip test?
4. **Human editor**: is selecting a slide and asking for changes sufficient initially, or is a PowerPoint-like canvas needed?
5. **Data residency and assets**: may user-supplied templates/brand assets be staged on a dedicated worker host? Which controlled asset fetch paths are approved?
6. **Rendering profile**: what memory/cold-start/cost envelope is acceptable for simultaneous Chromium or LibreOffice jobs?

## 13. Primary references

- Slidev MCP: https://sli.dev/features/mcp
- Slidev agent skills: https://sli.dev/guide/work-with-ai
- Slidev export formats and caveats: https://sli.dev/guide/exporting
- Slidev CLI: https://sli.dev/builtin/cli
- Reveal.js: https://revealjs.com/
- Marp: https://marp.app/
- PptxGenJS: https://github.com/gitbrent/PptxGenJS
- Office Kit PPTX: https://github.com/office-kit/pptx
- Office Kit skill: https://github.com/office-kit/pptx/blob/main/skill/SKILL.md
- OpenPresentation: https://www.openpresentation.org/agents
- OPF MCP capability and boundaries: https://www.openpresentation.org/docs/reference/mcp
- OPF agent skills: https://www.openpresentation.org/docs/agent-skills
- Presenton: https://github.com/presenton/presenton
- PPTist: https://github.com/pipipi-pikachu/PPTist
- ONLYOFFICE Docs: https://github.com/ONLYOFFICE/DocumentServer
- Firecracker: https://github.com/firecracker-microvm/firecracker
- Cloud Hypervisor: https://github.com/cloud-hypervisor/cloud-hypervisor

---

**Recommendation:** Implement slides as another capability of the common sandbox/tool infrastructure. Start with Slidev in a Firecracker microVM for web-native decks, test Office Kit for targeted PPTX editing, keep PptxGenJS as a creation fallback, and investigate OPF if structured revisions become a core feature. **Keep the host as the control plane, keep code/render execution in guests, and keep all MCPs local to organization-operated infrastructure.**
