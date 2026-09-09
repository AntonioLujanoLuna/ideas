# OpenClaw + GitHub Copilot Custom Agents

## Using repository-native Copilot agents as scheduled and event-driven workers

**Status:** Architecture note  
**Updated:** 2026-09-09

---

## 1. Summary

OpenClaw can use **GitHub Copilot CLI as an external coding-agent runtime** through the Agent Client Protocol (ACP).

This means an organization can keep development behavior inside the repository using GitHub Copilot's native customization mechanisms:

- `.github/agents/*.agent.md` or `.github/agents/*.md`
- `.github/skills/<skill>/SKILL.md`
- `.github/copilot-instructions.md`
- `.github/instructions/**/*.instructions.md`
- `AGENTS.md`
- Copilot hooks
- MCP servers
- repository scripts and tools

while OpenClaw handles the higher-level operational concerns:

- scheduling
- event triggers
- routing
- orchestration
- workflow state
- notifications
- approvals
- identity
- integration with external systems

The resulting separation is:

```text
OpenClaw decides WHEN and WHY work runs.
Copilot decides HOW repository work is performed.
The repository defines WHAT rules, skills and conventions Copilot follows.
```

This allows an existing Copilot custom agent to become an **automatable executable capability** without rewriting the agent as an OpenClaw-native agent.

---

## 2. Core architecture

```text
                    External systems
                         │
       ┌─────────────────┼──────────────────┐
       │                 │                  │
     Cron             Git/Jira            Chat
       │              Webhooks            Request
       │                 │                  │
       └─────────────────┼──────────────────┘
                         ▼
                  ┌─────────────┐
                  │  OpenClaw   │
                  │             │
                  │ Scheduler   │
                  │ Router      │
                  │ Workflow    │
                  │ Identity    │
                  │ Delivery    │
                  └──────┬──────┘
                         │
                         │ ACP
                         ▼
                ┌──────────────────┐
                │ GitHub Copilot   │
                │       CLI        │
                └────────┬─────────┘
                         │
                         ▼
                   Repository
                         │
          ┌──────────────┼──────────────┐
          │              │              │
          ▼              ▼              ▼
      .agent.md       SKILL.md      instructions
          │              │              │
          └──────────────┼──────────────┘
                         ▼
                    tools / MCP
                         │
                         ▼
                 code / tests / PR
```

The important point is that **OpenClaw does not need to parse or reproduce the Copilot configuration itself**.

OpenClaw launches the Copilot harness in the appropriate repository. Copilot then operates using the customization available to Copilot in that working directory.

---

## 3. What OpenClaw supports

OpenClaw supports external coding harnesses through ACP.

With the official `@openclaw/acpx` runtime plugin, `copilot` is a supported harness identifier:

```text
agentId = "copilot"
```

A Copilot ACP run can therefore be started conceptually as:

```json
{
  "task": "Use the daily-maintainer agent and perform today's repository maintenance.",
  "runtime": "acp",
  "agentId": "copilot",
  "mode": "run",
  "cwd": "/srv/repos/payments"
}
```

Here:

- `runtime: "acp"` selects an external ACP harness.
- `agentId: "copilot"` selects GitHub Copilot CLI.
- `cwd` determines the repository in which Copilot operates.
- `mode: "run"` represents a one-shot execution.

OpenClaw also supports persistent ACP sessions where that is useful.

---

## 4. Repository-side Copilot configuration

A repository could contain:

```text
payments-service/
│
├── .github/
│   │
│   ├── agents/
│   │   ├── daily-maintainer.agent.md
│   │   ├── issue-implementer.agent.md
│   │   ├── ci-debugger.agent.md
│   │   ├── security-reviewer.agent.md
│   │   └── release-reviewer.agent.md
│   │
│   ├── skills/
│   │   ├── architecture-check/
│   │   │   ├── SKILL.md
│   │   │   └── check_architecture.py
│   │   │
│   │   ├── run-tests/
│   │   │   └── SKILL.md
│   │   │
│   │   ├── jira/
│   │   │   └── SKILL.md
│   │   │
│   │   └── create-pr/
│   │       └── SKILL.md
│   │
│   ├── instructions/
│   │   ├── python.instructions.md
│   │   ├── api.instructions.md
│   │   └── database.instructions.md
│   │
│   └── copilot-instructions.md
│
├── AGENTS.md
├── pyproject.toml
├── src/
└── tests/
```

Copilot CLI supports repository custom agents, skills and multiple forms of custom instructions.

Therefore an OpenClaw-triggered Copilot execution can make use of the same repository customization that a developer would use when running Copilot manually.

---

## 5. Example custom Copilot agent

For example:

```markdown
---
name: daily-maintainer
description: Performs the daily health and maintenance review for this repository.
tools:
  - shell
  - github
---

Perform the daily repository maintenance workflow.

1. Inspect commits and pull requests merged during the previous day.
2. Identify suspicious regressions, TODOs, dead code and obvious maintainability problems.
3. Use the architecture-check skill where relevant.
4. Run the appropriate unit and integration tests.
5. Check whether documentation is inconsistent with recent implementation changes.
6. Check dependency and security alerts available through the configured tools.
7. Do not modify production configuration.
8. Make changes only when they are low-risk and clearly justified.
9. For non-trivial changes, create an issue or pull request rather than pushing directly.
10. Produce a concise final report describing:
    - findings,
    - actions taken,
    - outstanding risks,
    - links to created issues or pull requests.
```

The agent can use shared repository skills rather than embedding all workflow knowledge in one very large prompt.

---

# 6. Skills remain Copilot's responsibility

For example:

```text
.github/skills/architecture-check/
├── SKILL.md
├── architecture_rules.md
└── check_dependencies.py
```

A `SKILL.md` might describe:

```markdown
---
name: architecture-check
description: Validates changes against the service architecture and dependency rules.
---

When reviewing architectural changes:

1. Read `docs/architecture.md`.
2. Check package boundaries.
3. Detect forbidden dependency directions.
4. Execute `check_dependencies.py`.
5. Report violations with file and symbol references.
```

Copilot can discover and use the skill when relevant.

OpenClaw does not need to know the implementation of `architecture-check`.

From OpenClaw's perspective, the high-level request can remain:

```text
Run the daily-maintainer Copilot agent.
```

Copilot handles:

```text
daily-maintainer
      │
      ├── architecture-check
      ├── run-tests
      ├── security rules
      ├── repository instructions
      └── create-pr
```

---

# 7. Instructions and rules also continue to apply

Copilot CLI supports repository instructions such as:

```text
.github/copilot-instructions.md
.github/instructions/**/*.instructions.md
AGENTS.md
```

For example:

```markdown
---
applyTo: "**/*.py"
---

- Python must target 3.13.
- Public functions require type annotations.
- Do not introduce synchronous HTTP calls inside async services.
- Use the project's structured logging wrapper.
- New business logic requires tests.
```

A Copilot custom agent executing through OpenClaw is still a **Copilot CLI session operating inside that repository**, so these Copilot-side customization mechanisms remain relevant.

This is an important architectural advantage because the central OpenClaw deployment does not need to reproduce every team's coding rules.

---

# 8. Running an agent every day

OpenClaw has a persistent automation scheduler.

A useful design is to define a dedicated OpenClaw agent whose runtime is Copilot ACP and whose working directory points to the desired repository.

Conceptually:

```json5
{
  agents: {
    entries: {
      "payments-maintainer": {
        runtime: {
          type: "acp",
          acp: {
            agent: "copilot",
            backend: "acpx",
            mode: "oneshot",
            cwd: "/srv/repos/payments-service"
          }
        }
      }
    }
  }
}
```

Then schedule it:

```bash
openclaw automations create "0 7 * * *" \
  "Use the daily-maintainer custom agent. Perform today's repository maintenance." \
  --name "Payments daily maintenance" \
  --tz "Europe/Madrid" \
  --agent payments-maintainer
```

The logical execution becomes:

```text
07:00 every day
      │
      ▼
OpenClaw scheduler
      │
      ▼
payments-maintainer
      │
      │ ACP
      ▼
Copilot CLI
      │
      ▼
/srv/repos/payments-service
      │
      ▼
daily-maintainer.agent.md
      │
      ├── skills
      ├── instructions
      ├── tools
      └── MCP
```

OpenClaw keeps the schedule and run history.

Copilot performs the repository work.

---

# 9. Event-driven execution

The same model works when execution should happen because something changed rather than because a clock fired.

OpenClaw exposes authenticated inbound webhooks that can start agent work.

The general pattern is:

```text
External event
     │
     ▼
Webhook
     │
     ▼
OpenClaw
     │
     ▼
Configured Copilot ACP agent
     │
     ▼
Repository custom agent
```

---

## 9.1 Pull request opened

For example:

```text
Git provider
     │
     │ PR opened
     ▼
OpenClaw webhook
     │
     ▼
Copilot ACP
     │
     ▼
security-reviewer.agent.md
     │
     ├── inspect diff
     ├── architecture-check
     ├── security skill
     ├── test impact
     └── publish findings
```

A webhook request can target a configured OpenClaw agent:

```bash
curl https://openclaw.internal/hooks/agent \
  -H "Authorization: Bearer <HOOK_TOKEN>" \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: pr-1842-review" \
  --data '{
    "agentId": "payments-maintainer",
    "message": "Use the security-reviewer custom agent to review PR #1842.",
    "sessionMode": "isolated",
    "deliver": false
  }'
```

The webhook sender should send only the trusted metadata needed to identify the event.

Where untrusted issue, PR, email or document content is involved, the workflow should retrieve it through appropriately restricted tools rather than blindly injecting arbitrary external text into a privileged agent prompt.

---

# 10. Jira-driven example

A corporate workflow could look like:

```text
Jira issue
status -> Ready for Development
          │
          ▼
      Jira webhook
          │
          ▼
       OpenClaw
          │
          ├── determine requester
          ├── validate permissions
          ├── determine repository
          └── establish workflow state
                  │
                  ▼
            Copilot ACP
                  │
                  ▼
       issue-implementer.agent.md
                  │
        ┌─────────┼─────────┐
        ▼         ▼         ▼
      Jira    architecture  tests
      skill      skill      skill
        │
        └─────────┼─────────┘
                  ▼
             source edits
                  │
                  ▼
              create PR
                  │
                  ▼
            human approval
```

This gives each layer a clear responsibility.

---

# 11. CI failure example

Another useful event is a failed CI pipeline:

```text
CI pipeline failed
       │
       ▼
event / webhook
       │
       ▼
OpenClaw
       │
       ▼
Copilot ACP
       │
       ▼
ci-debugger.agent.md
       │
       ├── inspect CI logs
       ├── identify failing tests
       ├── reproduce failure
       ├── modify code
       ├── run focused tests
       ├── run regression tests
       └── create proposed fix
```

The agent could be instructed not to create a PR unless it can reproduce the failure locally.

This is a good example of why the actual coding policy belongs in the repository rather than in the central scheduler.

---

# 12. Multi-agent repository workflows

A repository may contain several specialized Copilot agents:

```text
.github/agents/
├── issue-triager.agent.md
├── implementation-planner.agent.md
├── backend-engineer.agent.md
├── test-engineer.agent.md
├── security-reviewer.agent.md
└── release-reviewer.agent.md
```

OpenClaw can decide which one should run according to the workflow.

For example:

```text
Jira READY
    │
    ▼
issue-triager
    │
    ▼
implementation-planner
    │
    ▼
backend-engineer
    │
    ▼
test-engineer
    │
    ▼
security-reviewer
    │
    ▼
PR / approval
```

However, the orchestration should generally **not** be hidden inside a single enormous `.agent.md`.

A better division is:

```text
                 OPENCLAW
              workflow layer
                   │
        ┌──────────┼──────────┐
        ▼          ▼          ▼
    scheduling   events    approvals
                   │
                   ▼
              COPILOT CLI
                   │
          ┌────────┼────────┐
          ▼        ▼        ▼
       agents    skills   rules
                   │
                   ▼
                code
```

---

# 13. Recommended ownership boundary

## OpenClaw should own

- time-based schedules
- external event ingestion
- webhook authentication
- workflow routing
- choosing which repository/agent executes
- requester identity
- authorization
- cross-system coordination
- notifications
- retries
- run history
- workflow state
- human approval gates
- organization-level policy

## Copilot should own

- repository understanding
- custom `.agent.md` definitions
- skills
- repository instructions
- path-specific coding rules
- code navigation
- source modification
- local test execution
- repository-specific scripts
- developer-facing MCP servers
- pull-request preparation
- implementation details

## The repository should own

- coding conventions
- architecture rules
- supported workflows
- reusable agent skills
- agent definitions
- test policy
- security policy specific to the application
- deployment restrictions specific to the application

This avoids duplicating repository knowledge in a central OpenClaw configuration.

---

# 14. Dynamic behavior and "on the fly" updates

One of the most attractive properties of this architecture is that Copilot customization is version-controlled with the application.

For example:

```text
developer changes:
.github/agents/release-reviewer.agent.md

        │
        ▼

pull request review

        │
        ▼

merge

        │
        ▼

repository checkout updated

        │
        ▼

next new Copilot execution
uses the new definition
```

The same applies to:

```text
.github/skills/*
.github/copilot-instructions.md
.github/instructions/*
AGENTS.md
```

### Important nuance

Do not interpret this as guaranteed hot-reload inside an already-running persistent Copilot process.

GitHub documents restarting Copilot CLI when loading newly created custom agents.

Therefore the safest operational model for automated work is:

```text
scheduled/event run
      │
      ▼
fresh one-shot Copilot session
      │
      ▼
load current repository customization
```

This is desirable anyway for most deterministic automation workloads.

Persistent sessions are useful when conversational continuity matters, but a long-lived session should not be assumed to immediately pick up every repository customization change.

For infrastructure automation, prefer:

```text
mode = one-shot / run
```

unless persistent state is explicitly required.

---

# 15. Infrastructure that still has to exist

Repository configuration can be dynamically discovered, but runtime dependencies do not magically appear.

For example:

```text
Copilot custom agent
        │
        ├── instructions                 ✓ repository
        ├── skills                       ✓ repository
        ├── coding rules                 ✓ repository
        │
        ├── git                           ⚠ must exist
        ├── python/node/java/etc.         ⚠ must exist
        ├── kubectl                       ⚠ must exist if required
        ├── gh / provider CLI             ⚠ must exist if required
        ├── MCP endpoint                  ⚠ must be reachable
        ├── Copilot authentication        ⚠ must exist
        ├── Git credentials               ⚠ must exist
        ├── Jira credentials              ⚠ must exist
        └── filesystem permissions        ⚠ must allow operations
```

A useful rule is:

> Repository behavior can be hot-pluggable. Runtime infrastructure and authorization must still be provisioned explicitly.

---

# 16. ACP permission model

This part is especially important for unattended agents.

OpenClaw's ACP sessions are non-interactive from the point of view of normal terminal permission prompts.

Current ACPX permission configuration distinguishes:

```text
permissionMode:
    approve-all
    approve-reads
    deny-all
```

and behavior when an interactive prompt would otherwise be required:

```text
nonInteractivePermissions:
    fail
    deny
```

The default behavior is intentionally conservative.

An unattended coding agent that needs to write files or execute commands must therefore have a deliberate permission design.

Do not solve this globally with unrestricted `approve-all` unless the whole harness environment is an acceptable trust boundary.

A better corporate pattern is to isolate workers by repository and purpose.

For example:

```text
worker: payments-reviewer
    repo: payments-service
    filesystem: read-only
    network: internal Git + required MCP only
    agent: security-reviewer
    permissions: reads

worker: payments-maintainer
    repo: payments-service
    filesystem: writable checkout
    git credentials: branch + PR only
    production credentials: none
    agent: daily-maintainer
```

---

# 17. Important security boundary

OpenClaw's documentation currently notes that ACP harness sessions execute on the **host runtime**, not inside the OpenClaw sandbox.

Therefore:

```text
OpenClaw sandbox
      ≠
ACP harness sandbox
```

The external Copilot process can act according to:

- operating-system permissions
- filesystem access
- Copilot CLI permissions
- credentials present in its environment
- network access
- MCP configuration
- Git provider authorization

For corporate use, this means ACP workers should be treated similarly to CI workers or build agents.

Recommended controls include:

- dedicated service identities
- least-privilege Git permissions
- repository-scoped credentials
- isolated worktrees or ephemeral clones
- no production credentials by default
- network segmentation
- restricted MCP endpoints
- explicit write policies
- branch protection
- human approval for sensitive actions
- audit logging
- secret redaction
- bounded execution time
- concurrency limits

---

# 18. OpenClaw tools are not automatically inherited by Copilot

Another important boundary:

```text
OpenClaw plugin tools
        │
        X  not automatically visible
        │
Copilot ACP
```

OpenClaw provides optional MCP bridges for exposing selected OpenClaw tools to ACP harnesses.

The plugin-tools bridge can be enabled explicitly:

```bash
openclaw config set \
  plugins.entries.acpx.config.pluginToolsMcpBridge true
```

OpenClaw also provides a separate bridge for selected built-in tools:

```bash
openclaw config set \
  plugins.entries.acpx.config.openClawToolsMcpBridge true
```

The latter initially exposes selected core tools such as automation/cron functionality.

These bridges increase the capability and therefore the trust surface of the Copilot worker.

They should be enabled only when the agent genuinely needs them.

In many cases a cleaner architecture is:

```text
OpenClaw
   │
   ├── Jira
   ├── notifications
   ├── scheduling
   └── workflow state

Copilot
   │
   ├── Git
   ├── repo tools
   ├── tests
   └── repo-specific MCP
```

rather than exposing all OpenClaw capabilities to every coding agent.

---

# 19. Initial OpenClaw ACP setup

The official OpenClaw ACPX plugin can be installed and enabled with:

```bash
openclaw plugins install @openclaw/acpx

openclaw config set plugins.entries.acpx.enabled true
```

Then check readiness:

```text
/acp doctor
```

A minimal ACP configuration focused on Copilot could conceptually be:

```json5
{
  acp: {
    enabled: true,
    dispatch: {
      enabled: true
    },
    backend: "acpx",
    defaultAgent: "copilot",
    allowedAgents: [
      "copilot"
    ]
  }
}
```

The host also needs valid GitHub Copilot CLI/runtime authentication.

---

# 20. Suggested corporate deployment model

For a multi-team organization, a useful architecture would be:

```text
                         ┌────────────────────┐
                         │     OpenClaw       │
                         │                    │
 Teams / chat ─────────► │ request handling   │
 Jira ─────────────────► │ workflow state     │
 Git events ───────────► │ event routing      │
 CI events ────────────► │ automation         │
 schedules ────────────► │ approval gates     │
                         └─────────┬──────────┘
                                   │
                              ACP dispatch
                                   │
                 ┌─────────────────┼──────────────────┐
                 │                 │                  │
                 ▼                 ▼                  ▼
             Copilot           Codex              Qwen Code
                 │
                 ▼
           repository
           customization
                 │
      ┌──────────┼───────────┐
      ▼          ▼           ▼
  .agent.md   SKILL.md   instructions
      │
      ▼
  team-owned
  behavior
```

The central platform team maintains:

```text
OpenClaw
ACP workers
identity
security
observability
credentials
routing
integration infrastructure
```

Individual application teams maintain:

```text
.github/agents/
.github/skills/
.github/instructions/
AGENTS.md
repository scripts
tests
architecture policy
```

This scales much better than centrally maintaining prompts for hundreds of repositories.

---

# 21. Example workflow: autonomous daily repository health check

A complete daily workflow could be:

```text
01. OpenClaw cron fires at 07:00.
02. OpenClaw targets `payments-maintainer`.
03. The ACP runtime starts Copilot in `/srv/repos/payments-service`.
04. The checkout is updated to the desired branch/revision.
05. Copilot loads the current repository customization.
06. OpenClaw asks Copilot to use `daily-maintainer`.
07. Copilot invokes relevant skills.
08. Copilot runs repository tests and checks.
09. Low-risk corrective work may be performed.
10. A branch/PR is created if policy permits.
11. Copilot returns a structured result.
12. OpenClaw records the run.
13. OpenClaw sends the report to the configured destination.
```

The same agent can still be used manually by a developer.

That is valuable because automation and interactive developer use share the same policy.

---

# 22. Example workflow: event-triggered issue implementation

```text
01. Jira issue transitions to READY.
02. Jira sends an authenticated webhook to OpenClaw.
03. OpenClaw validates project, requester and workflow policy.
04. OpenClaw maps the Jira project to a repository.
05. OpenClaw starts the configured Copilot ACP worker.
06. Copilot loads that repository.
07. OpenClaw requests `issue-implementer`.
08. Copilot retrieves the issue through an approved tool/skill.
09. Copilot creates an implementation plan.
10. Copilot modifies the code.
11. Copilot runs tests.
12. Copilot performs repository-specific validation.
13. Copilot creates a branch and PR if policy permits.
14. OpenClaw records the workflow state.
15. OpenClaw notifies the requester/reviewers.
16. Human review remains the release gate.
```

---

# 23. Why this design is attractive

### 23.1 Repository-local behavior

The code and the agent policy evolve together.

```text
application version
+
agent behavior
+
skills
+
coding rules
```

can all be reviewed in Git.

### 23.2 No duplicated prompt configuration

The platform does not need its own copy of every team's:

- architecture rules
- test commands
- language standards
- security checks
- release conventions

### 23.3 Agents are reusable interactively and automatically

The same:

```text
security-reviewer.agent.md
```

can be run:

```text
developer manually
OpenClaw nightly
OpenClaw on PR
OpenClaw before release
```

### 23.4 Tooling remains specialized

OpenClaw is optimized for orchestration and integrations.

Copilot is optimized for repository-centric software engineering.

Each system can remain responsible for the problem it is best positioned to solve.

### 23.5 Git becomes the control plane for application-specific agent behavior

A change to an agent can go through:

```text
branch
→ review
→ CI
→ approval
→ merge
```

just like code.

That is far preferable to invisible prompt changes in a central production service.

---

# 24. Recommended design principles

## Principle 1: Keep orchestration outside the coding agent

Bad:

```text
one gigantic agent
does scheduling + Jira + coding + testing + approvals + notification
```

Better:

```text
OpenClaw
    orchestrates

Copilot agents
    execute specialized repository work
```

---

## Principle 2: Prefer one-shot sessions for automation

For unattended scheduled/event-driven work:

```text
fresh execution
current checkout
current agent definitions
current skills
current rules
```

is normally easier to reason about than a long-lived conversational session.

---

## Principle 3: Give agents narrow authority

For example:

```text
reviewer
    read code
    run static analysis
    comment on PR
    cannot push

implementer
    write branch
    run tests
    create PR
    cannot merge

release agent
    validate release
    cannot deploy production without approval
```

---

## Principle 4: Make actions idempotent

External events can be retried.

Use stable identifiers such as:

```text
repository + PR number + head SHA + workflow name
```

to prevent duplicate:

- PR reviews
- branches
- comments
- issues
- remediation attempts

OpenClaw webhooks support idempotency keys and should use them for event-driven workflows.

---

## Principle 5: Do not inject untrusted event bodies into powerful prompts

Prefer:

```text
Webhook:
"PR 1842 changed"

Agent:
fetch PR 1842 using authenticated/restricted Git tooling
```

rather than:

```text
Webhook:
<entire arbitrary PR body copied into privileged system prompt>
```

This reduces prompt-injection exposure.

---

## Principle 6: Separate machine identity from human identity

For autonomous maintenance:

```text
service identity
```

is usually appropriate.

For user-requested actions where authorization should follow the requester:

```text
requester-scoped identity
```

may be appropriate.

This distinction should be enforced by the orchestration layer, not left to natural-language prompts.

---

# 25. Practical first proof of concept

A good first implementation would avoid Jira and complex multi-agent flows.

Start with one repository and one agent.

### Repository

```text
.github/agents/daily-maintainer.agent.md
.github/skills/run-tests/SKILL.md
.github/skills/architecture-check/SKILL.md
.github/copilot-instructions.md
```

### OpenClaw

```text
configured ACPX
configured Copilot auth
one repository checkout
one Copilot-backed OpenClaw agent
one daily automation
```

### Initial task

```text
Every weekday at 07:00:

1. update the repository checkout;
2. run daily-maintainer;
3. inspect changes from the previous 24 hours;
4. run relevant tests;
5. identify actionable problems;
6. do not push or create a PR;
7. produce a report.
```

Initially make it read-only.

Once the quality is acceptable:

```text
Phase 1: report only

Phase 2: report + draft patch

Phase 3: branch + PR

Phase 4: event-driven operation

Phase 5: selected autonomous remediation
```

This provides a controlled path to increasing autonomy.

---

# 26. Final mental model

The most useful way to think about the system is:

```text
┌────────────────────────────────────────────────────┐
│                    OPENCLAW                        │
│                                                    │
│   "When should something run?"                     │
│   "Which repository/worker should receive it?"     │
│   "Who requested it?"                              │
│   "Is it allowed?"                                 │
│   "Where should the result go?"                    │
│                                                    │
└────────────────────────┬───────────────────────────┘
                         │
                         │ ACP
                         ▼
┌────────────────────────────────────────────────────┐
│                 GITHUB COPILOT CLI                 │
│                                                    │
│   "How do I perform this engineering task?"        │
│   "Which custom agent should handle it?"           │
│   "Which skills are relevant?"                     │
│   "Which repository rules apply?"                  │
│   "Which tools should I execute?"                  │
│                                                    │
└────────────────────────┬───────────────────────────┘
                         │
                         ▼
┌────────────────────────────────────────────────────┐
│                    REPOSITORY                      │
│                                                    │
│   .github/agents/                                  │
│   .github/skills/                                  │
│   .github/instructions/                            │
│   copilot-instructions.md                          │
│   AGENTS.md                                        │
│   tests / scripts / docs / source                  │
│                                                    │
└────────────────────────────────────────────────────┘
```

In short:

> **OpenClaw becomes the orchestration and automation plane, while GitHub Copilot custom agents become repository-native executable workers.**

This makes it possible to take custom Copilot agents that teams already use interactively and run them:

- manually,
- every day,
- on a schedule,
- when a pull request appears,
- when CI fails,
- when a Jira transition occurs,
- before a release,
- or as one stage of a larger automated workflow.

The repository remains the source of truth for engineering behavior, while OpenClaw provides the event-driven operational layer around it.

---

# 27. References

Current documentation used for this architecture note:

1. OpenClaw, **ACP agents**  
   https://docs.openclaw.ai/tools/acp-agents

2. OpenClaw, **ACP agents quickstart / setup**  
   https://docs.openclaw.ai/tools/acp-agents/quickstart  
   https://docs.openclaw.ai/tools/acp-agents-setup

3. OpenClaw, **ACP sessions and bindings**  
   https://docs.openclaw.ai/tools/acp-agents/sessions  
   https://docs.openclaw.ai/tools/acp-agents/bindings

4. OpenClaw, **Automations**  
   https://docs.openclaw.ai/automation/cron-jobs

5. OpenClaw, **Inbound webhooks**  
   https://docs.openclaw.ai/automation/cron-jobs/webhooks

6. GitHub, **Creating and using custom agents for GitHub Copilot CLI**  
   https://docs.github.com/en/copilot/how-tos/copilot-cli/customize-copilot/create-custom-agents-for-cli

7. GitHub, **Adding agent skills for GitHub Copilot CLI**  
   https://docs.github.com/en/copilot/how-tos/copilot-cli/customize-copilot/add-skills

8. GitHub, **Adding custom instructions for GitHub Copilot CLI**  
   https://docs.github.com/en/copilot/how-tos/copilot-cli/customize-copilot/add-custom-instructions

9. GitHub, **Copilot CLI customization overview**  
   https://docs.github.com/en/copilot/how-tos/copilot-cli/customize-copilot/overview
