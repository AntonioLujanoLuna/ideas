# Portal Agent Tool Execution Architecture

**Status:** Design proposal  
**Scope:** LangGraph and GitHub Copilot SDK agents, centrally hosted web tools, client-side filesystem/spreadsheet operations, and sandboxed code execution.
**Deployment requirement (confirmed 2026-11-08):** Search/metasearch, MCP servers, crawling, browser workers, caches, gateways and agent infrastructure must all run on organization-managed hosts. They MAY query external search engines and websites via the approved corporate proxy. No hosted web-search/crawling APIs or remote MCP services. Maintaining our own global web index is not required.

## 1. Objective

Provide a common set of tools to agents running in the portal without coupling a tool's interface to the machine where it executes.

The portal already supports LangGraph and GitHub Copilot SDK agents, as well as server-side and client-side tools. Extend that infrastructure with:

- **Centralized web research:** Search the public web and retrieve pages using an internally deployed Web Research Gateway, with an optional internally hosted MCP interface and outbound requests through the approved corporate proxy.
- **Client-side operations:** Read and modify user-selected Excel workbooks, access explicitly authorized local files, and interact with local applications.
- **Sandboxed code execution:** Execute generated code in an isolated environment, preferably on the client when the operation depends on local data.
- **A shared tool registry and router:** Present consistent tool schemas to both agent frameworks while selecting the appropriate execution backend and enforcing permissions.

**Key principle:** MCP is an interface/protocol, not a requirement that every operation run on an MCP server. Centrally hosted services can use MCP; existing client-side tool execution can use the portal's own request/response protocol.

## 2. Proposed architecture

\`\`\`mermaid
flowchart TB
    UI[Portal / client] <-->|Authenticated session| BACKEND[Agent backend]
    BACKEND --> LG[LangGraph]
    BACKEND --> CP[Copilot SDK]
    LG --> ROUTER[Tool registry and router]
    CP --> ROUTER

    ROUTER -->|Server-side MCP| WEB[Internal web-research MCP]
    WEB --> SEARX[SearXNG]
    WEB --> STATIC[HTTP + Trafilatura first-pass extraction]
    WEB --> FETCH[Crawl4AI browser fallback]
    WEB --> PDF[Existing Docling pipeline]
    SEARX --> PROXY[Corporate HTTP/HTTPS proxy]
    STATIC --> PROXY
    FETCH --> PROXY
    PROXY --> INTERNET[Public Internet]

    ROUTER <-->|Authenticated WebSocket| CLIENT[Client tool executor]
    CLIENT --> XLSX[Excel / local files]
    CLIENT --> SANDBOX[Sandboxed Python / code execution]

    ROUTER -->|Optional alternative| SERVER_SANDBOX[Isolated server workspace]
\`\`\`

Three distinct execution locations:

| Execution location | Examples | Interface |
| --- | --- | --- |
| Server services | Web search, web fetch, internal RAG, Jira | Internal MCP or native backend handlers |
| Client | Excel, local filesystem, locally installed applications | Existing client-tool transport, e.g. WebSocket |
| Isolated sandbox | Generated Python, shell commands, expensive data transformations | Client-side runner or isolated server-side workspace |

The user-facing client normally connects to the **portal backend**, not directly to the centralized MCP. The backend controls agent sessions, authorization and tool dispatch.

## 3. Self-hosted web research through the corporate proxy

**Interpretation of fully self-hosted:** The portal operates every service that agents invoke or depend on. In particular, hosted Brave/Exa/Parallel/Tavily/You.com search APIs, Firecrawl/Crawl4AI cloud offerings, public MCP endpoints and public SearXNG instances are not dependencies. **External search engines and public websites are permitted as outbound destinations.** SearXNG is a metasearch proxy, not our own web index: it forwards queries to configured engines, which may observe the originating corporate proxy and may rate-limit or challenge requests. Never send confidential prompts, sensitive data or internal document content as public search queries. Maintaining an independent/offline search index is out of scope.

### 3.1 Services

Deploy the following components on the backend's host/container network, or on reachable internal infrastructure:

| Component | Purpose |
| --- | --- |
| Internal web MCP server | Exposes a stable agent-facing tool interface |
| SearXNG | Private metasearch service with JSON results from permitted external search engines |
| HTTP fetch + Trafilatura | Lightweight, browser-free first-pass text/Markdown extraction, with source metadata |
| Crawl4AI | Browser-based fallback for JavaScript-heavy or extraction-resistant pages |
| Existing Docling pipeline | Processes fetched PDF documents |
| Corporate proxy | Controls outbound HTTP/HTTPS traffic from all services reaching the public web |
| Optional private cache | Reduces repeated search and retrieval traffic with appropriate expiration policies |
| Optional isolated Playwright browser workers | Interactive navigation only when ordinary search/extraction is insufficient |

The MCP server itself need not have unrestricted Internet connectivity. The processes making external requests must be able to reach the corporate proxy, and the proxy must allow the required destinations.

**Important:** A self-hosted gateway and SearXNG do not create Internet access. SearXNG still queries external search engines. If the corporate proxy blocks them, enable permitted engine routes or degrade gracefully; do not introduce a hosted search API as a fallback. Enable JSON explicitly in SearXNG's `search.formats` configuration; it may otherwise return HTTP 403 for JSON requests. Test permitted search engines for blocking, CAPTCHA, throttling and relevance.

A representative SearXNG configuration:

\`\`\`yaml
search:
  formats:
    - html
    - json

outgoing:
  request_timeout: 5.0
  max_request_timeout: 15.0
  proxies:
    all://:
      - http://proxy.internal:8080
\`\`\`

Configure SearXNG, the HTTP fetcher and Crawl4AI/Playwright browser egress independently. All outbound public-web paths must traverse the approved proxy. Enforce egress restrictions at the network boundary, rather than trusting HTTP-client settings alone. Handle corporate proxy authentication and CA certificates in the environment's secret and trust stores; do not hardcode credentials into agent tools. Keep MCP and backend endpoints private.

### 3.2 Tool interface

Start with a small set of operations:

| Tool | Input | Result |
| --- | --- | --- |
| \`web_search\` | Query, result limit, optional language/time filters | Ranked titles, URLs, snippets, publication dates where available |
| \`web_fetch\` | URL | Clean Markdown/text, source metadata, truncation marker |
| \`web_find\` | Previously fetched document ID, search pattern | Matching passages with locations |
| \`browser\` (optional) | Navigation or interaction request | Page state and bounded interaction results |

Use HTTP fetch + Trafilatura for ordinary articles and documentation. Fall back to Crawl4AI's self-hosted rendering for JavaScript-heavy or extraction-resistant pages. Use separate isolated Playwright workers only for truly interactive navigation. PDFs should go through the existing document-processing pipeline. Do not launch a browser for every `web_fetch` call.

Keep complete retrieved documents outside the LLM context where practical. Return bounded excerpts and let the agent request additional passages. Preserve source URLs and dates so answers can cite their evidence.

### 3.3 Network and security

- Authenticate callers to the MCP endpoint and authorize individual tools per user/agent.
- Permit outbound access only through the approved proxy. Apply destination allowlists where useful.
- Guard fetching and browsing against SSRF, DNS rebinding, redirects to private addresses, and cloud-metadata endpoints. URL string checks alone are insufficient.
- Restrict result and response sizes, timeouts, simultaneous searches and browser workers.
- Treat all retrieved web content as untrusted. A web page must not be able to override the agent's instructions or tool permissions.
- Cache search responses and fetched pages with separate, query-appropriate expiration policies.
- Record provider failures, latency, cache hit rate and tool usage using the existing observability stack.

No externally hosted search API is permitted as a fallback. If an engine throttles or blocks SearXNG, apply per-engine backoff, switch among approved engines, report partial failures and monitor coverage. The self-hosted gateway keeps the `web_search` contract stable while search engine configurations change.

### 3.4 Implementation candidates reviewed in November 2026

| Candidate | Fit | Decision |
| --- | --- | --- |
| **SearXNG + HTTP/Trafilatura + Crawl4AI** | Modular, minimizes unnecessary browser work and uses only self-hosted services | **Preferred reference architecture** |
| **searxNcrawl + private SearXNG** | Ready-made self-hostable MCP for search, page crawling and site crawling | **Time-boxed PoC** as an alternative to custom MCP glue. Disable its optional You.com integration; audit auth, supply chain and network egress |
| **Firecrawl self-hosted + private SearXNG** | Unified search/crawl API with more operational dependencies | Optional benchmark if complex multi-page crawling warrants a heavier stack; do not use hosted Firecrawl |
| **Playwright CLI / self-hosted Playwright MCP** | Interactive browser workflows | Separate restricted browser worker pool, not the ultralight command microVM |
| **YaCy / self-maintained web index** | Independent web indexing | Out of scope for the confirmed requirement |

Prefer a small internal **Web Research Gateway** with native HTTP handlers behind the shared Tool Router, adding a self-hosted MCP interface where advantageous for Copilot SDK interoperability. MCP is a protocol choice, not a requirement to insert a server hop for every tool. Normalize results across frameworks with source URL, title, publication/retrieval timestamps where available, engine attribution, extraction method, document ID and truncation indicators.

### 3.5 Self-hosted web MVP acceptance criteria

1. Deploy private SearXNG with JSON enabled, configured for approved engines through the corporate proxy. Evaluate search coverage, blocking, relevance and p50/p95 response times on realistic developer and Spanish-language queries.
2. Offer `web_search`, `web_fetch` and `web_find` through the shared router to both LangGraph and Copilot SDK. Keep full fetched documents in bounded, scoped server-side storage and return citation-ready excerpts.
3. Use HTTP + Trafilatura first; only fall back to Crawl4AI for dynamic pages; reuse existing Docling for PDFs. Benchmark extraction accuracy, end-to-end latency, memory and browser concurrency.
4. Enforce tool-level user authorization, quotas, timeouts, cancellation and logging. Validate SSRF, DNS rebinding, redirects, prohibited IP ranges, metadata endpoints, outbound proxy bypass and prompt-injection boundaries. Do not grant microVMs direct Internet access to enable web research.
5. Exercise 10/50/100 concurrent calls, cache reuse, engine backoff and failure isolation. Add interactive browser workers only where necessary.
6. Time-box a searxNcrawl trial before building custom MCP glue, but do not adopt it without reviewing provider defaults, version pinning, Chromium provisioning and production security controls.

## 4. Client-side tools

### 4.1 Why use a client executor?

The backend may not have access to the user's workbook, local filesystem, installed Excel application or authenticated desktop session. The agent can still request these operations if the portal routes them to a client-side executor.

Suggested tool catalog:

| Tool | Function | Execution |
| --- | --- | --- |
| \`excel_inspect\` | Enumerate sheets, ranges, dimensions and metadata | Client |
| \`excel_read\` | Read selected ranges, formulas or values | Client |
| \`excel_write\` | Change specific cells/ranges; optionally create a new file | Client |
| \`excel_chart\` | Create or modify a chart | Client |
| \`filesystem_read\` | Read an explicitly authorized local file | Client |
| \`filesystem_write\` | Write a permitted destination, preferably with user approval | Client |
| \`python_execute\` | Execute code with bounded resources and mounted inputs | Client sandbox |
| \`shell_execute\` (optional) | Execute a tightly permissioned command | Client sandbox |

Prefer structured operations for routine spreadsheet edits. Use arbitrary code execution for transformations that structured tools cannot express efficiently.

### 4.2 Browser-only versus native client

**Browser-only portal**

- Read user-selected workbooks via a JavaScript library such as ExcelJS.
- Run compatible Python through Pyodide/WebAssembly, with limitations on Python packages, native extensions, memory and OS access.
- Use browser file-selection and download APIs rather than assuming arbitrary filesystem access.
- Do not expose commercial search API credentials in browser code. Direct browser fetches are also subject to CORS restrictions.

**Local companion application**

A small authenticated local process, written in Python, Rust or another suitable language, can access explicitly approved local resources and run a more complete toolchain:

- \`openpyxl\` for XLSX structure, cells, styling and formulas.
- \`pandas\` for tabular manipulation.
- An isolated Python runtime for computations and visualizations.
- Optional OS- or Excel-specific automation when the user needs genuine application behavior.

Note that \`openpyxl\` can preserve and write formulas but does **not** calculate Excel formula results itself. If formula recalculation is required, use an appropriate spreadsheet calculation engine or the installed application, with explicit compatibility testing.

A companion process offers more capabilities than a browser-only client but introduces deployment, patching and endpoint-security requirements.

### 4.3 Client tool transport

For backend-hosted agents, use a request/response protocol over the existing authenticated client connection, such as a WebSocket.

Backend to client:

\`\`\`json
{
  "type": "tool.invoke",
  "request_id": "req-123",
  "conversation_id": "conv-456",
  "tool": "excel_read",
  "arguments": {
    "file_id": "user-selected-workbook",
    "sheet": "Forecast",
    "range": "A1:F20"
  }
}
\`\`\`

Client to backend:

\`\`\`json
{
  "type": "tool.result",
  "request_id": "req-123",
  "status": "success",
  "result": {
    "range": "A1:F20",
    "values": [["Year", "Revenue"], [2026, 1200000]]
  }
}
\`\`\`

The client resolves \`file_id\` against a user-approved file selection. Do not let the model choose arbitrary host paths without an independent authorization check.

The backend must associate every invocation with an authenticated user, device, conversation and agent run. Correlate responses with \`request_id\`; reject unsolicited or stale results. Implement timeouts, cancellation, client-disconnect handling and duplicate-response protection.

For large files and generated artifacts, use a controlled upload/download path or artifact storage rather than transferring entire workbooks as WebSocket JSON.

### 4.4 The local code sandbox

Generated code should **not** run with the same privileges as the portal client or unrestricted access to the user's home directory.

A preferred approach for machines where it is supported is an isolated, rootless Podman container with:

- A non-privileged user.
- Read-only mounts for approved input files.
- A dedicated writable output directory.
- No network access by default.
- CPU, memory, process-count and execution-time limits.
- No host Docker/Podman socket or privileged mounts.
- An explicit approval step before applying generated changes to the original files.

Containers alone are not a complete security boundary against hostile code. Enforce host-level isolation and endpoint-security requirements appropriate to the organization. Where a container runtime is unavailable, consider a more restricted language runtime or a remote isolated workspace instead of silently falling back to unrestricted host execution.

Return bounded standard output, standard error, exit status and artifact references. Support incremental output and cancellation for longer-running jobs.

## 5. Optional server-side sandbox for file-based tasks

Not every spreadsheet job requires a local executor. A simpler browser-compatible alternative is:

1. The user selects a workbook or dataset.
2. The client uploads it to an isolated, user-scoped server workspace.
3. The agent runs code in a server-side sandbox and produces modified files.
4. The portal returns the artifacts for review and download.

This is attractive when deploying a local Python runtime to every employee is impractical. It requires authorization to upload the data and well-defined retention, isolation and deletion policies.

**Selection rule:** Use client execution when work depends on local files, local applications or data that cannot leave the endpoint. Use a server-side sandbox when the user explicitly supplies transferable files and central execution is permitted.

## 6. Shared tool registry and framework adapters

Define tool metadata independently of either agent SDK. For example:

\`\`\`json
{
  "name": "python_execute",
  "description": "Run Python in an isolated workspace against approved inputs.",
  "execution": "client_sandbox",
  "permissions": ["execute_code"],
  "requires_approval": true,
  "timeout_seconds": 120,
  "input_schema": {
    "type": "object",
    "properties": {
      "code": { "type": "string" },
      "input_artifact_ids": {
        "type": "array",
        "items": { "type": "string" }
      }
    },
    "required": ["code"],
    "additionalProperties": false
  }
}
\`\`\`

The registry can specify execution location, schema, scopes, approval policy, timeouts and availability. Actual executors must independently enforce those restrictions.

### LangGraph adapter

Register tools as ordinary asynchronous LangChain/LangGraph tools:

- **Server tool:** Invoke the relevant local handler or internal MCP tool.
- **Client tool:** Send the invocation through the client transport and await the correlated result.
- **Sandbox tool:** Dispatch to the selected isolated runner.

For user approval or long-lived pauses, use LangGraph interrupts and resume with a persistent checkpointer. Ensure the tool dispatcher and pending requests survive backend restarts where required.

### Copilot SDK adapter

Use the Copilot SDK's custom-tool or external-tool mechanism:

- **Handler-backed tool:** Its handler invokes the same router used by LangGraph.
- **Externally executed tool:** The backend handles the SDK's external-tool request event, forwards it to the client and supplies the corresponding result back to the SDK.
- **Internal MCP:** Configure the centrally hosted web MCP as a server available to the agent session.

Built-in Copilot filesystem and shell tools execute where the Copilot runtime is hosted. When the runtime is on the backend, do not assume those built-ins access the user's PC. Restrict or replace them with explicitly routed client tools.

### Routing policy

The tool router should resolve:

1. Is the tool enabled for this agent and user?
2. Does it require approval, and has approval been granted for these exact arguments?
3. Where must it execute: backend, internal MCP, client or sandbox?
4. Is the required device connected and the execution environment available?
5. How will the result, errors and artifacts be returned and logged?

The model may request a tool, but it must not be able to override these decisions by changing tool arguments or inventing a destination.

## 7. Reliability, lifecycle and observability

- Give every call an immutable invocation ID. Design retries around idempotency; a spreadsheet write or shell command must not accidentally execute twice.
- Persist the state of long-lived runs and define behavior for client disconnects. Do not reassign a tool request to another user's device.
- Cancel pending executions when a run is cancelled or its authorization expires.
- Apply per-user and global quotas for web searches, browsers and sandboxes.
- Keep large tool outputs and files in scoped artifact storage and return references to the agent.
- Trace tool names, executor locations, durations, result sizes, costs, failure categories and approval decisions in Langfuse or the existing observability system.
- Avoid logging sensitive workbook contents, credentials or arbitrary code outputs without an explicit retention policy.

## 8. Example end-to-end workflows

### Web research

1. A user asks an agent to research a topic.
2. The agent requests \`web_search\`.
3. The backend calls the internal web MCP.
4. SearXNG sends its outbound requests through the corporate proxy.
5. The agent selects a result and calls \`web_fetch\`.
6. The extractor retrieves the page through the proxy and returns clean Markdown.
7. The agent answers with source references.

### Local spreadsheet analysis

1. A user selects a local workbook.
2. The client registers a scoped file identifier.
3. The agent calls \`excel_inspect\` and \`excel_read\` through the tool router.
4. The router forwards the requests to the authenticated client.
5. For complex calculations, the agent requests \`python_execute\` in a sandbox with the selected workbook mounted.
6. Generated workbook/chart artifacts are returned to the client for review.
7. The user approves any operation that overwrites an original file.

### Mixed research and spreadsheet workflow

An agent can search for public market data through the server-side web MCP, process approved local financial data in the client sandbox, and write the resulting calculations into a user-selected workbook. The tool router keeps those operations separate while presenting a single workflow to the agent.

## 9. Incremental implementation plan

1. **Internal web MVP:** Deploy private SearXNG with JSON enabled, HTTP + Trafilatura, Crawl4AI fallback and the internal Web Research Gateway. Validate the corporate proxy, CA trust, permitted search engines, PDF retrieval and default-deny egress. Compare searxNcrawl for the optional MCP interface.
2. **Shared tool registry:** Normalize schemas, permission policies and execution locations across LangGraph and Copilot SDK.
3. **Client dispatch:** Extend the existing client-tool protocol with correlation IDs, scoped authorization, timeouts, cancellation and artifact handling.
4. **Structured file tools:** Implement workbook inspection, reading and writing before introducing arbitrary code execution.
5. **Isolated execution:** Add a client sandbox where supported; alternatively, offer scoped server-side workspaces for uploaded files.
6. **Operational hardening:** Add approval flows, auditing, idempotent retries, quotas, isolation testing and disconnect/recovery behavior.
7. **Optional browser automation:** Add restricted browser workers only for workflows that cannot be satisfied by search and page extraction.

## 10. Decisions and open questions

- **Confirmed (2026-11-08):** All web-research infrastructure, MCP servers, crawlers, browser services and caches are self-hosted. Public engines/sites may be queried through approved outbound proxy egress. No externally hosted search/crawl/MCP API and no independent indexing requirement.
- **Recommended default:** Internal Web Research Gateway (native HTTP with optional self-hosted MCP) for centralized web research; existing client-tool protocol for local operations; a separate sandbox for generated code.
- **Network:** Can service hosts reach an approved corporate proxy? Which external search engines and sites are permitted?
- **Client type:** Is the portal strictly browser-based, or can a managed local companion be installed?
- **Runtime:** Are rootless Podman or another approved sandbox available on client machines?
- **Data handling:** Which file categories may be uploaded into server-side workspaces?
- **Authorization:** Which tool scopes need per-invocation approval, and which can be granted per session?
- **Scale:** What concurrency and latency targets should govern search backends, browser workers and code sandboxes?

## References

- [Trafilatura extraction documentation](https://trafilatura.readthedocs.io/en/latest/corefunctions.html)
- [searxNcrawl (PoC candidate)](https://github.com/DasDigitaleMomentum/searxNcrawl)
- [Firecrawl self-hosting (alternative)](https://docs.firecrawl.dev/contributing/self-host)

- [Model Context Protocol](https://modelcontextprotocol.io/)
- [SearXNG search API](https://docs.searxng.org/dev/search_api.html)
- [SearXNG outgoing request configuration](https://docs.searxng.org/admin/settings/settings_outgoing.html)
- [Crawl4AI](https://docs.crawl4ai.com/)
- [Playwright MCP](https://github.com/microsoft/playwright-mcp)
- [LangChain MCP adapters](https://github.com/langchain-ai/langchain-mcp-adapters)
- [LangGraph interrupts](https://docs.langchain.com/oss/python/langgraph/interrupts)
- [GitHub Copilot SDK](https://github.com/github/copilot-sdk)
- [ExcelJS](https://github.com/exceljs/exceljs)
- [Pyodide](https://pyodide.org/)
- [openpyxl](https://openpyxl.readthedocs.io/)
