# CONTEXT.md — Ubiquitous Language

The domain glossary for **CyberStrikeAI**. Use these exact terms in code, prompts, commits,
and conversation. Each entry gives the meaning *in this codebase*; an `_Avoid_` line lists
words that mean something different here, so we don't blur them.

Reverse-engineered from the code (`internal/`, `roles/`, `skills/`, `agents/`, `tools/`).
When the code and this file disagree, the code wins — fix the entry.

---

## Agents & orchestration

**Agent** — a conversational AI loop: takes input, calls tools, folds results, streams a
reply. Two implementations coexist: the **legacy single-agent** (`internal/agent/`, one
ChatModel + tool loop) and the **modern multi-agent** (`internal/multiagent/`, Eino ADK).
Unqualified "agent" usually means the multi-agent engine now.
_Avoid_: "agent" for a C2 implant (that's a **Session**); for an IM connector (that's a
**Robot**); for a `sub-agent` when you mean the top **Orchestrator**.

**Orchestrator** — the coordinator agent in multi-agent mode. Picks an orchestration
**mode** and delegates to sub-agents via the `task` tool. Defined by `agents/orchestrator*.md`.

**Orchestration mode** — one of **Deep**, **Plan-Execute**, or **Supervisor**; selected per
request in `eino_orchestration.go`. Not a config toggle per conversation — it's how the
orchestrator decomposes work.

**Sub-agent** — a specialist delegated a scoped `task` by the orchestrator (recon,
penetration, vulnerability-triage, reporting, …). Sees **only its task description**, never
the parent conversation. Defined by `agents/*.md`.

**Eino / ADK** — CloudWeGo Eino Agent Development Kit, the framework the multi-agent engine
is built on. "Eino skills", "Eino middleware" are ADK-specific plumbing, distinct from our
own **Skill** packs (see below — the names collide, the concepts don't).

**Middleware** (Eino ADK) — pluggable steps around the agent loop: `tool_search`,
`patch_tool_calls`, `plantask`, `reduction`, `checkpoint`, `summarization`.
_Avoid_: HTTP middleware — that's `auth_middleware.go` in `internal/security/`.

**ChatModel / Model** — the LLM behind an agent (DeepSeek, Claude, GPT-4, …), configured in
`openai:`. "Model" always means the LLM, never a data/DB model.

---

## Filesystem-driven assets

**Role** — a YAML security-testing **persona** in `roles/` (渗透测试, Web应用扫描, CTF, …).
Customizes the system prompt and restricts the available tool list. User-selected in the UI.
_Avoid_: an IAM/auth role — there is no RBAC here; a Role is a prompt+toolset preset.

**Skill** — a knowledge/instruction **pack** in `skills/`, one `SKILL.md` (YAML front matter
+ Markdown) per directory, following the Anthropic Agent-Skills spec. Loaded on demand by the
Eino `skill` tool (progressive disclosure), multi-agent only.
_Avoid_: "skill" for a Role or a Tool; a Skill is documentation the model pulls in, not an
executable and not a persona.

**Tool** — a security utility an agent can call (nmap, sqlmap, nuclei, …), defined as a YAML
**recipe** in `tools/` (name, command, args, parameters, enabled). In the MCP sense it's a
callable resource; here recipe and MCP-tool are the same concept.
_Avoid_: "tool" for a Skill (docs) or for the **Executor** (the thing that runs tools).

**Tool recipe** — the YAML file describing one Tool. The **Executor** resolves it into a
command line at runtime.

**Executor** — the Go component (`internal/security/executor.go`) that turns a tool recipe +
parameters into a process invocation, runs it, and captures stdout/stderr/exit code.

---

## Protocol & integration

**MCP (Model Context Protocol)** — the standard protocol bridging agents and tools.
CyberStrikeAI runs a **native MCP server** (HTTP/stdio/SSE) and can **federate** to
**external MCP** servers. Tools are exposed as MCP resources.

**External MCP** — a third-party MCP server registered in the UI; its tools appear alongside
built-in tools. Managed by `internal/mcp/external_manager.go`.

**Robot** — an IM-platform adapter (`internal/robot/`) that bridges WeChat/WeCom, DingTalk,
Lark, Telegram, Slack, Discord, or QQ to the conversation engine. Config key: `robots:`.
_Avoid_: "bot" for an agent — a Robot is a transport, an Agent is the reasoning loop.

---

## Conversation & persistence

**Conversation** — a chat thread (SQLite row): id, title, project id, timestamps, messages.

**Message** — one turn in a Conversation: role (user/assistant/tool), content, tool-call
references, timestamps.

**Batch task** — a queued unit of work (`internal/database/batch_task.go`), mapped to a
Conversation and executed asynchronously with its own state.
_Avoid_: the Eino `task` tool call (orchestrator→sub-agent delegation) — same word, different
layer. "Batch task" = queued job; "task" = a sub-agent delegation.

**Vulnerability** — a discovered flaw: title, description, **severity** (critical/high/
medium/low/info), **status** (open/confirmed/fixed/false_positive), conversation id.

**WebShell** — a stored remote-shell connection (URL, auth, shell type, request method) an
agent can drive as a conversation surface.

---

## Shared knowledge & analysis

**Project** (a.k.a. **Blackboard**) — a cross-session shared store of **Facts** plus an edge
graph linking them (`internal/project/`). Groups related Conversations.
_Avoid_: "project" in the generic repo sense — here it's a specific first-class entity.

**Fact** — a piece of evidence/intelligence on the Project blackboard (an IP, domain, CVE,
credential, …). Auto-indexed into vectors on upsert; linked to other Facts by **edges** to
form attack paths. MCP tool: `upsert_project_fact`.

**Attack chain** — a directed graph auto-extracted per Conversation: nodes = targets/tools/
vulnerabilities, edges = causality. Built by `internal/attackchain/`, visualized and replayable.
_Avoid_: confusing with the Project fact-graph — Attack chain is per-conversation and derived;
the Project graph is cross-session and curated.

**Knowledge base (RAG)** — vector-indexed Markdown in `knowledge_base/`
(`internal/knowledge/`). Retrieval = MultiQuery rewrite → vector fusion → HTTP rerank. A
**KnowledgeItem** is a source Markdown file; a **KnowledgeChunk** is one indexed vector entry.
_Avoid_: "knowledge base" for the Project blackboard — KB is reference docs; the blackboard is
live engagement evidence.

---

## Execution control & safety

**C2 (Command & Control)** — the built-in lightweight remote-execution framework
(`internal/c2/`). Event-driven, HITL-gated. **Authorized testing only.**

**Listener** (C2) — an inbound endpoint awaiting implant check-ins. Types: `tcp_reverse`,
`http_beacon`, `https_beacon`, `websocket`. Holds sessions and tasks; per-listener crypto keys.

**Session** (C2) — a persistent implant connection to a Listener, with implant metadata
(host, user, PID). Tasks are queued to it; results return via events.
_Avoid_: "session" for a user login/auth session (`auth_manager.go`) or a shell/PTY session
(`internal/security/shell_*.go`) — three different "sessions", keep them qualified.

**Task** (C2) — a command queued to a C2 Session for an implant to execute. Distinct from a
**Batch task** (job queue) and the Eino `task` tool (sub-agent delegation).

**HITL (Human-in-the-Loop)** — the approval gate (`internal/hitl/`): a tool allowlist +
approval mode; high-risk tools (e.g. C2 tasks, raw `execute`) require human confirmation
before the Executor runs them.

**Workflow** — a visual DAG of nodes (Start/Agent/Tool/Condition/HITL/Output) in
`internal/workflow/`. A Role may bind a `workflow_id` so a chat auto-executes the graph.
_Avoid_: "workflow" for an Orchestration mode — a Workflow is an author-defined DAG; a mode is
how the orchestrator itself reasons.

**Audit** — platform audit trail (`internal/audit/`): auth events, config changes, tool
executions. **Not** conversation content — that lives in the Conversation/Message tables.
