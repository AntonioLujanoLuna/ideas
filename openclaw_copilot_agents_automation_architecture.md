# OpenClaw + GitHub Copilot Custom Agents

## Repository-native agents as scheduled, event-driven, persistent backend workers

**Status:** Architecture note  
**Updated:** 2026-09-09

---

## 1. Executive summary

GitHub Copilot CLI can be used in two complementary ways:

1. **As an ACP coding harness launched by OpenClaw** for scheduled or event-driven tasks.
2. **As a persistent backend agent runtime through the GitHub Copilot SDK** for multi-user applications, resumable conversations, custom tools, subagents, and long-lived agent services.

This enables a clean architecture in which repository-specific behavior remains version-controlled with the code:

```text
.github/agents/*.agent.md
.github/skills/*/SKILL.md
.github/copilot-instructions.md
.github/instructions/**/*.instructions.md
AGENTS.md
```

while orchestration and platform concerns remain outside the repository agent:

```text
OpenClaw / workflow layer
    schedules
    webhooks
    Jira / CI events
    approvals
    notifications
    routing

Agent backend
    API
    auth / tenancy
    session metadata
    locks
    workspace isolation

Copilot runtime
    persistent conversations
    reasoning loop
    tools
    skills
    subagents
    MCP
    hooks

Repository
    agent definitions
    engineering rules
    source / tests
```

For an existing backend that mainly uses LangGraph for persistent agent conversations, tools, streaming, and checkpointing to S3, the Copilot SDK is now a credible replacement candidate for much of the agent-runtime layer.

It is not automatically a replacement for a deterministic workflow engine when the application relies heavily on explicit graph nodes, arbitrary checkpoint state, timers, transactional transitions, or durable human-approval states.

---

## 2. Repository-native Copilot customization

A repository can keep its agent behavior locally:

```text
payments-service/
├── .github/
│   ├── agents/
│   │   ├── daily-maintainer.agent.md
│   │   ├── issue-implementer.agent.md
│   │   ├── ci-debugger.agent.md
│   │   ├── security-reviewer.agent.md
│   │   └── release-reviewer.agent.md
│   ├── skills/
│   │   ├── architecture-check/
│   │   │   └── SKILL.md
│   │   ├── run-tests/
│   │   │   └── SKILL.md
│   │   └── create-pr/
│   │       └── SKILL.md
│   ├── instructions/
│   │   ├── python.instructions.md
│   │   └── api.instructions.md
│   └── copilot-instructions.md
├── AGENTS.md
├── src/
└── tests/
```

A custom agent may describe one focused engineering role:

```markdown
---
name: daily-maintainer
description: Performs the daily repository health and maintenance review.
tools:
  - shell
  - github
---

1. Inspect relevant changes from the previous day.
2. Run the repository's architecture-check skill where appropriate.
3. Run relevant tests.
4. Identify regressions, maintenance problems and documentation drift.
5. Never modify production configuration directly.
6. For non-trivial changes, create a branch or pull request rather than pushing to the protected branch.
7. Produce a concise report of findings and actions.
```

Skills and instructions remain repository-owned. OpenClaw does not need to duplicate their content.

The useful separation is:

```text
OpenClaw decides WHEN and WHY work runs.
Copilot decides HOW repository work is performed.
The repository defines WHAT rules and skills apply.
```

---

## 3. OpenClaw as the orchestration layer

OpenClaw can use Copilot CLI as an external coding harness through ACP.

Conceptually:

```text
Teams / Jira / Git / CI / Cron
              │
              ▼
          OpenClaw
              │
              │ ACP
              ▼
       GitHub Copilot CLI
              │
              ▼
          repository
              │
      agents / skills / rules
```

A Copilot ACP invocation can target a repository working directory and ask Copilot to use a named custom agent.

This makes existing Copilot agents automatable without converting them into OpenClaw-native agents.

### Recommended ownership boundary

OpenClaw should own:

- schedules;
- external event ingestion;
- webhook authentication;
- requester identity;
- routing;
- cross-system workflows;
- notifications;
- retries;
- human approval gates;
- organization-level policy.

Copilot should own:

- repository understanding;
- `.agent.md` behavior;
- skills;
- path-specific coding instructions;
- code navigation;
- source changes;
- local tests;
- repository scripts;
- engineering-focused MCP tools.

The repository should own:

- coding conventions;
- architecture rules;
- test policy;
- application-specific security restrictions;
- custom agents;
- reusable skills.

---

## 4. Scheduled agents

An agent can be run periodically, for example every weekday morning:

```text
07:00
  │
  ▼
OpenClaw scheduler
  │
  ▼
Copilot ACP
  │
  ▼
daily-maintainer.agent.md
  │
  ├── skills
  ├── instructions
  ├── tests
  └── repository tools
```

A safe first rollout is:

```text
Phase 1: report only
Phase 2: report + draft patch
Phase 3: branch + pull request
Phase 4: event-driven execution
Phase 5: selected autonomous remediation
```

For unattended scheduled jobs, prefer fresh one-shot executions unless conversational continuity is specifically required. This makes it easier for each run to pick up the current repository agent definitions, skills and instructions.

---

## 5. Event-driven agents

The same pattern works for external events.

### Pull request opened

```text
PR opened
   │
   ▼
webhook
   │
   ▼
OpenClaw
   │
   ▼
Copilot
   │
   ▼
security-reviewer.agent.md
```

### CI failure

```text
CI failed
   │
   ▼
OpenClaw
   │
   ▼
ci-debugger.agent.md
   │
   ├── inspect logs
   ├── reproduce failure
   ├── modify code
   ├── run tests
   └── prepare proposed fix
```

### Jira transition

```text
Jira issue -> Ready for Development
              │
              ▼
           OpenClaw
              │
      validate requester
      determine repository
      apply workflow policy
              │
              ▼
          Copilot
              │
              ▼
     issue-implementer.agent.md
              │
      plan -> edit -> test -> PR
```

For security, event payloads should preferably contain trusted identifiers such as issue ID, PR number or commit SHA. The agent should retrieve full external content through authenticated, restricted tools rather than blindly injecting arbitrary webhook text into a privileged prompt.

---

## 6. Copilot SDK as a backend runtime

For a larger platform, Copilot does not need to be launched as a new terminal subprocess for every request.

GitHub documents a headless server mode:

```bash
copilot --headless --port 4321
```

or, when explicitly required between containers or hosts:

```bash
copilot --headless --host 0.0.0.0 --port 4321
```

The Copilot SDK can connect to that persistent runtime programmatically.

```text
Application backend
      │
      │ Copilot SDK / JSON-RPC
      ▼
Copilot CLI --headless
      │
      ├── Session A
      ├── Session B
      ├── Session C
      └── ...
```

The runtime should normally remain behind the application's authenticated backend rather than being exposed directly to users.

This leads to two useful deployment modes:

```text
Simple automation:
OpenClaw -> ACP -> Copilot CLI

Agent platform:
OpenClaw / Web UI / API
          │
          ▼
     Agent Backend
          │
          ▼
      Copilot SDK
          │
          ▼
  Copilot Runtime Pool
```

---

## 7. Parallel conversations and agents

There are several independent levels of parallelism.

### 7.1 Multiple independent sessions

```text
                  Copilot runtime
                       │
         ┌─────────────┼─────────────┐
         ▼             ▼             ▼
     Session A     Session B     Session C
         │             │             │
      User A         User B        Cron job
```

Each session has its own conversation and agent context.

The application should enforce its own concurrency limits, quotas and resource controls.

For larger installations, scale horizontally with multiple headless runtime processes or pods.

```text
                      API
                       │
                       ▼
                 Runtime router
                       │
          ┌────────────┼────────────┐
          ▼            ▼            ▼
      runtime 1    runtime 2    runtime 3
          │            │            │
       sessions     sessions     sessions
```

### 7.2 Parallel subagents inside one session

A session can delegate work to custom subagents. Fleet mode is intended for parallel dispatch of independent subagent tasks.

```text
Main agent
    │
    ├────────────┬────────────┬────────────┐
    ▼            ▼            ▼            ▼
 Explorer      Backend      Tests       Security
    │            │            │            │
    └────────────┴────────────┴────────────┘
                       │
                       ▼
                  parent session
```

A useful model is:

```text
runtime processes
    x sessions
        x subagents
```

Actual useful concurrency remains constrained by plan limits, external APIs, repository locks, CPU/memory, and the amount of work that is safe to parallelize.

---

## 8. Isolate mutable repository work

Independent coding sessions should not share one writable checkout.

Bad:

```text
Session A ─┐
           ├── /repos/payments
Session B ─┘
```

Better:

```text
                    repository
                        │
          ┌─────────────┼─────────────┐
          ▼             ▼             ▼
      worktree A    worktree B    worktree C
          │             │             │
      Session A     Session B     Session C
```

Use one worktree, ephemeral clone or isolated workspace per active coding job.

This avoids Git index races, branch confusion, tests observing another session's partial edits, and accidental commits containing unrelated changes.

---

## 9. Persistent and resumable conversations

Copilot SDK sessions can be made resumable by providing an explicit `sessionId`.

```typescript
const session = await client.createSession({
  sessionId: "tenant-42-user-7-conversation-123",
  model: "gpt-5.4"
});
```

Later, including from another SDK client instance:

```typescript
const resumed = await client.resumeSession(
  "tenant-42-user-7-conversation-123"
);
```

GitHub documents resumption across process restarts, container migrations and different client instances when persisted session storage remains available.

The lifecycle distinction is important:

```text
session.disconnect()
    releases the active in-memory session
    preserves resumable persisted state

client.deleteSession(sessionId)
    permanently removes persisted session state
```

---

## 10. Where session state is stored

With the normal local configuration, resumable session state lives under:

```text
~/.copilot/session-state/{sessionId}/
```

or under the equivalent configured `COPILOT_HOME` / base directory.

The persisted session state is used to restore the Copilot conversation and agent execution context.

Application-defined in-memory state inside custom tools is not automatically durable. Tools should be stateless or persist their own durable state in application services.

A sensible split is:

```text
Copilot session persistence
    conversation and agent execution state

Application database
    users, tenants, authorization, session metadata, workflow state

Tool-owned persistence
    domain-specific state
```

Do not use the Copilot session directory as the application's only database.

---

## 11. `sessionFs` and S3-backed persistence

For server deployments, the most relevant persistence primitive is the SDK's `sessionFs` abstraction.

GitHub documents it for scenarios where:

- local runtime disk is ephemeral;
- session state must live outside the Copilot process;
- tenant-aware storage paths are needed;
- session-scoped I/O should use application-managed storage;
- object storage is desired.

Conceptually:

```text
Copilot SDK / runtime
        │
        │ sessionFs
        ▼
application filesystem adapter
        │
        ├── S3
        ├── MinIO
        ├── object-storage-compatible layer
        └── durable filesystem
```

Client-level configuration is conceptually:

```typescript
const client = new CopilotClient({
  mode: "empty",
  sessionFs: {
    initialCwd: "/workspace",
    sessionStatePath: "/session-state",
    conventions: "posix"
  }
});
```

The application supplies the concrete provider/adapter according to the SDK language.

This is directly applicable to an architecture that already stores LangGraph checkpoint blobs in S3.

However, `sessionFs` is not a built-in `S3SessionSaver`. An S3-backed implementation still needs an adapter that provides the filesystem semantics required by the runtime and handles object naming, consistency and concurrency correctly.

---

## 12. Recommended production persistence architecture

```text
                         Agent Backend
                              │
          ┌───────────────────┼───────────────────┐
          ▼                   ▼                   ▼
      PostgreSQL            Redis             Copilot SDK
      metadata              locks                 │
                                                  ▼
                                             sessionFs
                                                  │
                                                  ▼
                                                  S3
```

PostgreSQL can remain authoritative for application metadata:

```text
agent_sessions
--------------
id
tenant_id
user_id
copilot_session_id
repository_id
workspace_id
created_at
updated_at
status
runtime_affinity
```

Resumption becomes:

```text
GET /conversations/123
          │
          ▼
lookup metadata
          │
          ▼
copilot_session_id
          │
          ▼
resumeSession(...)
          │
          ▼
sessionFs restores durable session state
```

### Same-session locking

The SDK does not provide distributed locking for multiple clients mutating the same session concurrently.

Therefore:

```text
request A ─┐
           ├── distributed lock / queue ──► Session 123
request B ─┘
```

should be serialized at the application layer, for example with Redis.

Different independent sessions can continue in parallel.

---

## 13. LangGraph + S3 versus Copilot SDK

A useful conceptual mapping is:

| LangGraph concept | Copilot SDK analogue |
|---|---|
| `thread_id` | `sessionId` |
| checkpointer | Copilot session persistence / `sessionFs` |
| S3 checkpoint blob | application-backed `sessionFs`, potentially S3 |
| messages | persisted conversation state |
| agent loop | Copilot runtime |
| tool calls | SDK tools / MCP |
| agent definitions | custom agents |
| reusable prompt/tool modules | skills |
| streamed graph events | Copilot session events |

The systems are not identical.

LangGraph checkpoints can represent arbitrary workflow state:

```python
{
    "messages": ...,
    "current_node": ...,
    "approval_required": ...,
    "documents": ...,
    "retry_count": ...,
    "business_state": ...
}
```

Copilot persistence primarily represents the durable Copilot agent session.

### Strong replacement candidate

If the current graph is mainly:

```text
request
   │
   ▼
agent loop
   │
   ├── LLM
   ├── tools
   ├── retrieval
   └── response
   │
   ▼
checkpoint conversation
```

then Copilot SDK may replace most of the LangGraph runtime layer.

It already provides:

- resumable sessions;
- streaming;
- tools;
- custom agents;
- skills;
- subagents;
- parallel subagent execution;
- MCP;
- hooks;
- tool permissions;
- context management and compaction capabilities;
- session events;
- model configuration.

### Keep a workflow engine when deterministic graph semantics matter

If the application relies on structures such as:

```text
             ┌── validation
             │
request ─────┼── human approval
             │
             └── execute
                    │
             ┌──────┴──────┐
             ▼             ▼
           retry        escalation
```

with explicit durable state-machine semantics, arbitrary checkpoint variables, timers, transactional transitions or business-level retries, Copilot sessions alone are not a full replacement.

In that case use:

```text
small deterministic workflow layer
             │
             ▼
       Copilot sessions
             │
      ┌──────┼───────┐
      ▼      ▼       ▼
   agents   tools   skills
```

OpenClaw can own schedules, external events and notifications above this layer.

---

## 14. Giving Copilot custom tools

Copilot SDK can expose application functions directly as tools using `defineTool`.

```typescript
import { CopilotClient, defineTool } from "@github/copilot-sdk";

const searchJira = defineTool("search_jira", {
  description: "Search Jira issues visible to the current requester",
  parameters: {
    type: "object",
    properties: {
      query: {
        type: "string",
        description: "JQL or search expression"
      }
    },
    required: ["query"]
  },
  handler: async ({ query }: { query: string }) => {
    return jiraService.search(query);
  }
});

const session = await client.createSession({
  model: "auto",
  tools: [searchJira]
});
```

This makes existing backend capabilities directly callable:

```text
search_jira
get_jira_issue
search_internal_rag
query_elasticsearch
get_gitlab_merge_request
run_internal_pipeline
fetch_document
```

Not every capability needs a separate MCP server.

---

## 15. MCP and per-agent tool scoping

For reusable external tool services, Copilot SDK also supports MCP.

```text
Copilot session
     │
     ├── native SDK tools
     │
     └── MCP
          │
          ├── Jira MCP
          ├── Git provider MCP
          ├── RAG MCP
          └── internal platform MCP
```

Different custom agents can have different tool sets.

```text
Research Agent
    search_jira
    search_rag
    read_repo

Developer Agent
    search_jira
    search_rag
    read_repo
    edit_repo
    run_tests
    create_pr

Reviewer Agent
    search_rag
    read_repo
```

This is significantly stronger than relying only on prompt instructions for least privilege.

A specialized subagent can also receive an expensive or privileged tool while the parent agent is prevented from using it directly.

---

## 16. Enforcing policy with hooks

Tool restrictions should not live only in natural-language prompts.

Copilot SDK exposes lifecycle hooks including `onPreToolUse` and `onPostToolUse`.

`onPreToolUse` can allow, deny or ask for approval and can validate or modify arguments before execution.

```typescript
const session = await client.createSession({
  hooks: {
    onPreToolUse: async (input) => {
      if (input.toolName === "deploy_production") {
        return {
          permissionDecision: "deny",
          permissionDecisionReason:
            "Production deployment requires the release workflow."
        };
      }

      return { permissionDecision: "allow" };
    }
  }
});
```

In a corporate platform the hook should consult a real policy layer:

```text
Agent requests tool
       │
       ▼
 onPreToolUse
       │
       ├── requester identity
       ├── tenant
       ├── repository
       ├── agent identity
       ├── tool
       └── arguments
              │
              ▼
         policy engine
              │
       ┌──────┼──────┐
       ▼      ▼      ▼
     allow   deny    ask
```

Post-tool hooks can support auditing, redaction, output filtering and compliance logging.

---

## 17. Multi-user backend security

For server deployments, treat Copilot as a backend runtime rather than as a developer's personal CLI.

Important principles:

- isolate tenants and workspaces;
- use explicit per-session credentials where appropriate;
- avoid ambient host credentials;
- use least-privilege tool lists;
- keep production credentials out of ordinary coding agents;
- enforce authorization in code/hooks, not only prompts;
- keep Git branch protection as a final safety boundary;
- record tool execution for auditability;
- cap runtime duration and concurrency;
- isolate mutable Git workspaces.

For shared server scenarios, GitHub documents `mode: "empty"` as the basis for explicitly providing capabilities instead of inheriting ambient local behavior.

---

## 18. Recommended end-state architecture

```text
                         Clients / events
                              │
             ┌────────────────┼────────────────┐
             │                │                │
           Web UI           Teams            Jira / CI
             │                │                │
             └────────────────┼────────────────┘
                              ▼
                         API Backend
                              │
             ┌────────────────┼────────────────┐
             ▼                ▼                ▼
         PostgreSQL         Redis          OpenClaw
         metadata           locks          automation
                              │                │
                              └───────┬────────┘
                                      ▼
                                Copilot SDK
                                      │
                                      ▼
                           Copilot runtime pool
                    ┌─────────────────┼─────────────────┐
                    ▼                 ▼                 ▼
                runtime 1         runtime 2         runtime 3
                    │
              ┌─────┼─────┐
              ▼     ▼     ▼
           session session session
              │
       ┌──────┼────────┐
       ▼      ▼        ▼
    agents   tools    skills
       │
       ▼
 isolated worktree / workspace
       │
       ▼
 sessionFs -> durable object storage
```

Responsibilities:

```text
OpenClaw
    schedules, triggers, cross-system workflow, notifications

Agent Backend
    API, authentication, tenancy, routing, locks, metadata

Copilot SDK/runtime
    durable conversations, agent loops, tools, subagents, hooks

Repository
    custom agents, skills, instructions, engineering policy

S3/object storage
    durable Copilot session persistence via sessionFs

PostgreSQL
    authoritative application metadata and workflow references

Redis / queue
    same-session locking and short-lived coordination
```

---

## 19. Migration plan from an existing LangGraph backend

Do not start by deleting LangGraph.

### Phase 1: parity prototype

Reimplement one existing agent with Copilot SDK using:

```text
same frontend request
same tools
same model/provider constraints
same persistence requirement
```

Keep the existing S3 infrastructure and implement durable session persistence through `sessionFs`.

Measure:

- response quality;
- tool-call reliability;
- latency;
- concurrency;
- storage growth;
- restart/resume behavior;
- observability;
- authorization complexity;
- operational burden.

### Phase 2: persistence test

Verify:

```text
create session
-> several turns
-> disconnect
-> terminate runtime
-> start another runtime
-> resume same session from durable storage
-> continue correctly
```

### Phase 3: tool parity

Expose existing internal tools using `defineTool()` or MCP.

Enforce requester-scoped authorization through pre-tool hooks.

### Phase 4: concurrency test

Test:

```text
many independent sessions
+
parallel subagents
+
isolated worktrees
+
S3-backed sessionFs
```

while serializing writes to the same session.

### Phase 5: classify current LangGraph responsibilities

For every graph node and state field, classify it as:

```text
A. Copilot agent-runtime responsibility
B. deterministic application-workflow responsibility
C. obsolete after migration
```

If most of the current graph is type A, replacing LangGraph may substantially simplify the backend.

If many nodes are type B, retain a smaller deterministic workflow layer above Copilot.

---

## 20. Final mental model

```text
┌───────────────────────────────────────────────┐
│               ORCHESTRATION                   │
│ OpenClaw / workflow service                   │
│ schedules, events, approvals, notifications   │
└──────────────────────┬────────────────────────┘
                       │
                       ▼
┌───────────────────────────────────────────────┐
│               AGENT BACKEND                   │
│ API, tenancy, auth, routing, locks             │
│ PostgreSQL metadata                           │
└──────────────────────┬────────────────────────┘
                       │
                       ▼
┌───────────────────────────────────────────────┐
│              COPILOT RUNTIME                  │
│ resumable sessions                            │
│ reasoning + tools + subagents                 │
│ hooks + permissions + streaming               │
└──────────────────────┬────────────────────────┘
                       │
          ┌────────────┴─────────────┐
          ▼                          ▼
┌─────────────────────┐   ┌─────────────────────┐
│     REPOSITORY      │   │       STORAGE       │
│ .agent.md           │   │ sessionFs -> S3     │
│ SKILL.md            │   │ PostgreSQL metadata │
│ instructions        │   │ Redis coordination  │
│ source / tests      │   │                     │
└─────────────────────┘   └─────────────────────┘
```

The central design question is no longer whether Copilot can persist conversations or receive custom tools. It can.

The important question is whether the application still needs LangGraph's explicit durable graph semantics after Copilot sessions, subagents, skills, custom tools, MCP, hooks, S3-backed `sessionFs`, and OpenClaw/workflow orchestration are available.

---

## 21. References

### OpenClaw

1. ACP agents  
   https://docs.openclaw.ai/tools/acp-agents

2. ACP setup  
   https://docs.openclaw.ai/tools/acp-agents-setup

3. Automations  
   https://docs.openclaw.ai/automation/cron-jobs

4. Inbound webhooks  
   https://docs.openclaw.ai/automation/cron-jobs/webhooks

### GitHub Copilot CLI and SDK

5. Copilot CLI customization overview  
   https://docs.github.com/en/copilot/how-tos/copilot-cli/customize-copilot/overview

6. Creating custom agents for Copilot CLI  
   https://docs.github.com/en/copilot/how-tos/copilot-cli/customize-copilot/create-custom-agents-for-cli

7. Adding skills  
   https://docs.github.com/en/copilot/how-tos/copilot-cli/customize-copilot/add-skills

8. Copilot SDK  
   https://docs.github.com/en/copilot/how-tos/copilot-sdk

9. Backend services setup  
   https://docs.github.com/en/copilot/how-tos/copilot-sdk/setup/backend-services

10. Scaling  
    https://docs.github.com/en/copilot/how-tos/copilot-sdk/setup/scaling

11. Multi-tenancy and server deployments  
    https://docs.github.com/en/copilot/how-tos/copilot-sdk/setup/multi-tenancy

12. Session resume and persistence  
    https://docs.github.com/en/copilot/how-tos/copilot-sdk/features/session-persistence

13. Custom agents and subagent orchestration  
    https://docs.github.com/en/copilot/how-tos/copilot-sdk/features/custom-agents

14. MCP  
    https://docs.github.com/en/copilot/how-tos/copilot-sdk/features/mcp

15. Hooks  
    https://docs.github.com/en/copilot/how-tos/copilot-sdk/features/hooks

16. Pre-tool use hook  
    https://docs.github.com/en/copilot/how-tos/copilot-sdk/hooks/pre-tool-use

17. SDK custom tools / getting started  
    https://docs.github.com/en/copilot/how-tos/copilot-sdk/getting-started

18. SDK and CLI compatibility  
    https://docs.github.com/en/copilot/how-tos/copilot-sdk/troubleshooting/compatibility
