# Browser-Based Agent Runtime with Local Tools and Server-Enforced Personalization

## Idea

Build a web application for interacting with an LLM where the **agent harness runs primarily in the browser**, similarly to how agentic functionality in VS Code is orchestrated by the client.

The LLM itself remains a stateless model endpoint. The browser is responsible for the agent loop, conversation state, tool dispatch, approvals, and interaction with local capabilities.

The main architectural principle is:

> **The LLM is not the agent. The client plus its harness is the agent.**

A browser can therefore act as the agent client, while privileged or security-sensitive capabilities remain enforced by backend services.

---

## High-Level Architecture

```text
                         ┌─────────────────────┐
                         │   LLM / vLLM        │
                         │ OpenAI-compatible   │
                         └──────────▲──────────┘
                                    │
                             LLM Gateway
                                    ▲
                                    │
┌──────────────────── Browser Agent ┴─────────────────────┐
│                                                        │
│  Web UI                                                │
│                                                        │
│  Agent Runtime                                         │
│  ├─ conversation state                                 │
│  ├─ system/developer context                           │
│  ├─ context management                                 │
│  ├─ tool schemas                                       │
│  ├─ tool-call loop                                     │
│  ├─ approval handling                                  │
│  ├─ cancellation / streaming                           │
│  └─ checkpoints                                        │
│                                                        │
└───────────────┬───────────────────────┬────────────────┘
                │                       │
                ▼                       ▼
       Browser-local tools       Backend tools
       / local companion         / corporate services
                │                       │
       ├─ filesystem             ├─ RAG
       ├─ git                    ├─ Jira
       ├─ shell                  ├─ GitLab
       ├─ local MCP              ├─ databases
       └─ processes              └─ internal APIs
```

The browser is the orchestrator. The backend does not need to own the agent loop.

---

## Agent Loop

The frontend can implement the complete model/tool interaction cycle:

```text
messages
+ system instructions
+ available tools
+ tool schemas
        │
        ▼
       LLM
        │
        ▼
tool_call(...)
        │
        ▼
browser dispatches tool
        │
        ▼
tool result
        │
        ▼
       LLM
        │
        ▼
next tool call or final answer
```

Conceptually:

```text
while model requests a tool:
    execute tool
    append result to trajectory
    call model again
```

This is fundamentally the same pattern used by coding-agent clients.

---

## Local Capabilities

A browser can directly support a useful subset of local operations.

Possible browser-native capabilities include:

- File System Access API
- IndexedDB
- Origin Private File System (OPFS)
- WebAssembly
- Web Workers
- WebContainers
- Pyodide
- HTTP APIs
- browser storage and application state

Example local tool interface:

```text
read_file(path)
write_file(path, contents)
list_directory(path)
search_files(query)
apply_patch(path, diff)
```

After the user explicitly grants access to a workspace directory, the browser can read and modify those files.

---

## Limitation: Operating-System Access

A normal webpage cannot arbitrarily access the user's shell or spawn local operating-system processes.

Unlike VS Code, the browser cannot directly do:

```text
git status
pytest
docker ps
kubectl get pods
```

This is an intentional browser security boundary.

For a full coding-agent experience, introduce a small local companion process.

---

## Local Companion

A lightweight local daemon can expose controlled local capabilities to the browser over localhost.

```text
Browser Agent
      │
      │ localhost HTTP / WebSocket
      ▼
Local Companion
      │
      ├─ filesystem
      ├─ shell
      ├─ git
      ├─ Docker
      ├─ Kubernetes CLI
      ├─ local processes
      ├─ local MCP servers
      └─ other machine capabilities
```

For example:

```text
http://127.0.0.1:9843
```

with an RPC-like interface such as:

```json
{
  "method": "shell.exec",
  "params": {
    "command": "git diff"
  }
}
```

The browser remains responsible for:

- deciding which tool to call
- requesting user approval when necessary
- presenting results
- feeding tool results back to the LLM

The companion only provides controlled access to local capabilities.

A small Rust or Go binary would be a reasonable implementation.

---

## Conversation History

Because the agent loop lives in the client, conversation persistence must be explicitly designed.

A useful local model is:

```text
Browser
  └─ local database
       ├─ conversations
       ├─ turns
       ├─ observations
       ├─ tool calls
       ├─ tool results
       ├─ checkpoints
       └─ attachments
```

IndexedDB is sufficient for many applications.

For a more structured implementation, SQLite compiled to WASM and persisted through OPFS is attractive:

```text
React
  ↓
Agent Runtime
  ↓
SQLite WASM
  ↓
OPFS
```

### Suggested data model

```text
Conversation
  id
  title
  created_at
  updated_at
  model
  workspace_id

Turn
  id
  conversation_id
  user_message

Observation
  id
  turn_id
  parent_id
  type
  payload
  timestamp
```

Observation types might include:

```text
user
assistant
reasoning
tool_call
tool_result
final
```

It is useful to distinguish between:

1. **UI conversation history**
2. **full execution trajectory**

The second can include:

- model calls
- tool calls
- tool outputs
- patches
- token usage
- checkpoints
- errors
- retries

This makes a coding-agent session reproducible and inspectable.

---

## Persistence Options

### Fully local

```text
Browser
  ↓
IndexedDB / SQLite WASM / OPFS
```

Advantages:

- privacy
- low backend complexity
- fast local resume

Disadvantages:

- tied to a browser/device
- browser data can be cleared
- harder to use across machines

---

### Server-side history

```text
Browser
  ↓
Conversation API
  ↓
Postgres
```

The browser still owns the agent loop, while the server only stores durable product state.

Possible tables:

```text
users
conversations
turns
observations
agent_runs
attachments
```

---

### Hybrid

A strong default for a corporate implementation:

```text
                  ┌─ local IndexedDB / SQLite
Browser Agent ────┤
                  └─ central history service
```

Local persistence provides immediate state and resilience.

The server provides:

- synchronization
- cross-device access
- backup
- retention
- organizational controls

Langfuse or similar observability systems should remain telemetry systems rather than the canonical product conversation database.

---

## Workspace State

Conversation history and workspace state are different things.

A conversation may have been created against:

```text
repository: foo
branch: feature/login
commit: 71ad3f2
```

but later reopen against:

```text
repository: foo
branch: feature/login
commit: d98f177
```

The application should persist workspace metadata such as:

```text
workspace:
  repository_id
  directory_handle
  git_remote
  branch
  head_commit
```

On resume, the agent can detect that the workspace has changed.

This avoids silently treating old observations as if they still reflected the current repository.

---

# Agent Personalization

Running the agent harness in the frontend does **not** prevent customized agents.

An agent registry can define agents such as:

```text
General Coding
Python Expert
Java Platform
OpenShift
Internal Documentation
```

Selecting an agent can return configuration like:

```json
{
  "agent_id": "openshift",
  "model": "qwen",
  "system_prompt": "...",
  "tools": [
    "local.read_file",
    "local.shell",
    "corporate.rag_search"
  ],
  "max_iterations": 30
}
```

The browser can use this configuration to construct the model context and run the agent.

---

## Important Security Boundary

Anything delivered to browser JavaScript must be considered:

- visible to the user
- inspectable
- modifiable
- replayable

Therefore there must be a distinction between:

### Agent behavior

Configuration that may safely reach the browser:

- system or role prompt
- tool descriptions
- model choice
- temperature
- context strategy
- maximum iterations
- UI configuration
- default approval behavior

These settings influence normal behavior, but should not be treated as security controls.

### Server-side authority

Security-sensitive configuration must remain on the backend:

- RAG index selection
- document ACLs
- tenant boundaries
- Jira permissions
- GitLab credentials
- database credentials
- API secrets
- allowed model policy
- rate limits
- authorization rules

The browser should be a **policy consumer**, not the policy authority.

---

## RAG Design

RAG is particularly well suited to this architecture.

The frontend can expose a generic tool:

```text
search_corporate_docs(query)
```

The model calls:

```text
search_corporate_docs(
    query="How do we deploy services to OpenShift?"
)
```

The browser then calls a backend tool endpoint.

The browser should **not** decide which physical vector or Elasticsearch index to query.

Bad design:

```text
rag_search(
    index=user_provided_index,
    query=...
)
```

A modified client could simply request a restricted index.

Better design:

```text
rag_search(
    agent_id="openshift",
    query=...
)
```

and on the server:

```text
authenticated identity
      +
agent policy
      ↓
allowed RAG collection
      +
document ACL filtering
      ↓
search
```

Even better, the backend can derive the permitted agent context from the authenticated session rather than trusting an arbitrary `agent_id`.

The browser sees the logical capability:

```text
corporate.rag_search(query)
```

The backend controls:

```text
which index
which filters
which credentials
which tenant
which documents
```

---

## Protected System Instructions

If some system instructions must not be exposed to the browser, they can be injected by the LLM gateway.

```text
Browser Agent
      │
      │ conversation + tool context
      ▼
LLM Gateway
      │
      ├─ authenticate user
      ├─ resolve agent
      ├─ inject protected instructions
      ├─ enforce model policy
      └─ forward request
      ▼
     vLLM
```

The frontend can still own the complete interaction loop.

The gateway only owns protected or authoritative configuration.

Prompts should never contain credentials or secrets regardless.

---

# Recommended Separation of Responsibilities

## Browser

Owns:

```text
UI
conversation state
agent loop
context management
streaming
tool dispatch
approvals
local history
checkpoints
local tool execution
```

## Local Companion

Owns controlled access to:

```text
filesystem
shell
git
Docker
local processes
local MCP
developer environment
```

## LLM Gateway

Owns:

```text
authentication
model routing
quotas
protected prompt injection
allowed model policy
audit hooks
credential injection
```

## Corporate Tool Gateway

Owns:

```text
RAG
Jira
GitLab
databases
internal APIs
authorization
ACL enforcement
tenant isolation
credentials
```

## History Service

Owns:

```text
durable conversations
turns
trajectories
cross-device synchronization
retention
```

## Observability

Systems such as Langfuse receive:

```text
traces
model latency
token usage
cost
tool activity
errors
agent trajectories
```

but should not necessarily be the application's canonical conversation store.

---

# Resulting Architecture

```text
                              ┌──────────────────┐
                              │      vLLM        │
                              └────────▲─────────┘
                                       │
                              ┌────────┴─────────┐
                              │   LLM Gateway    │
                              │ auth / policies  │
                              └────────▲─────────┘
                                       │
                                       │
┌────────────────────────── Browser ────┴─────────────────────────┐
│                                                               │
│  Web UI                                                       │
│                                                               │
│  Agent Runtime                                                │
│  ├─ conversation                                              │
│  ├─ context                                                   │
│  ├─ tool loop                                                 │
│  ├─ approvals                                                 │
│  ├─ checkpoints                                               │
│  └─ local persistence                                         │
│                                                               │
└────────────┬───────────────────────────────┬───────────────────┘
             │                               │
             ▼                               ▼
    ┌─────────────────┐            ┌───────────────────────┐
    │ Local Companion │            │ Corporate Tool GW     │
    │                 │            │                       │
    │ files           │            │ RAG                   │
    │ shell           │            │ Jira                  │
    │ git             │            │ GitLab                │
    │ docker          │            │ internal APIs         │
    │ MCP             │            │ databases             │
    └─────────────────┘            └───────────────────────┘
             │                               │
             └───────────────┬───────────────┘
                             │
                       Agent capabilities

Browser Agent
      │
      ├────────► History Service / Postgres
      │
      └────────► Langfuse / telemetry
```

---

# Core Principles

1. **The model is not the agent.**  
   Agent behavior emerges from the client runtime, tools, context, and model.

2. **The browser can own orchestration.**  
   The backend does not need to execute the agent loop.

3. **Local tools can remain local.**  
   Files and development operations need not pass through the central server.

4. **Full OS access requires a local companion.**  
   Browser sandboxing prevents arbitrary shell/process execution.

5. **Conversation persistence is independent of orchestration.**  
   History can be local, server-side, or hybrid.

6. **Personalization remains possible with a frontend runtime.**  
   Agents can have different prompts, tools, models, and RAG profiles.

7. **Frontend configuration is not a security boundary.**  
   Anything shipped to the browser can be inspected or modified.

8. **Backend services enforce authority.**  
   RAG indexes, ACLs, credentials, tenant boundaries, and permissions stay server-side.

9. **Logical tools should hide physical resources.**  
   The browser asks for `rag_search`, not arbitrary Elasticsearch indexes.

10. **Observability and product state are separate concerns.**  
    Langfuse can trace the agent, while the application owns canonical conversation history.

---

## One-Sentence Summary

A web application can provide a VS Code-like agent experience by running the **agent harness and conversation loop in the browser**, using a **small local companion for privileged machine access**, while keeping **RAG, permissions, credentials, protected prompts, and other authoritative policies enforced server-side**.
