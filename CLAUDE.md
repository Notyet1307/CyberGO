# CLAUDE.md

This file guides Claude Code (claude.ai/code) when working in this repository.

> Secondary-development fork of **CyberStrikeAI**. History was reset to a single
> initial commit; upstream is no longer tracked. This file and `CONTEXT.md` were
> reverse-engineered from the code, not inherited from upstream — trust the code
> over any stale doc, and update these when you learn otherwise.

## What this is

**CyberStrikeAI** is an AI-native, end-to-end security-testing platform written in
Go. It drives 100+ security tools through natural-language AI agents to automate
vulnerability discovery, attack-chain analysis, and evidence collection. It bridges
conversational surfaces (web UI, IM bots, MCP clients) to a tool-execution engine, a
built-in lightweight **C2**, a RAG **knowledge base**, and role-based multi-agent
orchestration. **For authorized security assessments and red-team engagements only** —
the codebase carries audit trails and human-in-the-loop approval gates by design; keep
that posture in any change.

Domain vocabulary (agent, role, skill, tool, fact, attack chain, C2, HITL, …) is defined
in **`CONTEXT.md`** — read it first and use those exact terms.

## Commands

No Makefile — the entry points are `run.sh` and plain `go` commands. Module name is
`cyberstrike-ai`; Go **1.25**.

```bash
# One-click deploy + run (checks Go/Python, sets up venv, builds, starts).
./run.sh            # HTTPS with a self-signed cert (default)
./run.sh --http     # plain HTTP

# Manual build / run of the main server.
go build -o cyberstrike-ai cmd/server/main.go
go run cmd/server/main.go -config config.yaml --https

# MCP stdio server (for Cursor / CLI clients).
go build -o cyberstrike-ai-mcp cmd/mcp-stdio/main.go

# In-place upgrade (preserves local tools/roles/skills).
./upgrade.sh [--tag vX.Y.Z] [--yes]
```

Host/port/TLS come from the `server:` block in `config.yaml` — don't hard-code them.
If `go mod download` is slow (CN networks): `go env -w GOPROXY=https://goproxy.cn,direct`.

### Tests

```bash
go test ./...                         # all Go tests
go test ./internal/multiagent/...     # one package
go test -run TestName ./internal/...  # a single test
```

There is no lint target — use `gofmt -l .` and `go vet ./...`.

## Configuration

`config.yaml` (committed, currently `version: v1.6.51`) is **required** and drives almost
all behavior. Top-level sections: `server`, `auth`, `log`, `audit`, `monitor`, `openai`
(LLM API key/base-url/model), `vision`, `fofa`, `agent`, `hitl`, `multi_agent`,
`database`, `security` (incl. `tools_dir`), `mcp`, `external_mcp`, `knowledge`, `robots`,
`skills_dir`, `agents_dir`, `roles_dir`, `project`. `config.yaml` holds secrets (LLM keys,
auth) — don't commit real credentials.

## Architecture

Single Go binary (`cmd/server/main.go`). Everything user-facing — roles, skills, agents,
tools — is **filesystem/config-driven and hot-reloadable**, not hard-coded. The rough flow:

```
IM bots / web UI / MCP clients
        │
   internal/handler  (HTTP + SSE APIs, Gin)
        │
   internal/multiagent  (CloudWeGo Eino ADK: deep | plan_execute | supervisor)
        │           └─ internal/agent (legacy single-agent ChatModel loop)
        ▼
   internal/security  (Executor: tool YAML → CLI args → process → output)
        │
   internal/mcp  (native MCP server + federation to external MCPs)
        │
   internal/database  (SQLite: conversations, vulns, projects, C2, audit, KB vectors)
```

### Core packages (`internal/`)

- **`multiagent/`** — the modern engine. Eino ADK orchestration; `eino_adk_run_loop.go`
  routes chat/WebShell/batch requests to Deep/Plan-Execute/Supervisor orchestrators.
  `eino_skills.go` (progressive skill loading), `eino_middleware.go` (tool_search,
  plantask, reduction, checkpoint, summarization).
- **`agent/`** — legacy single-agent ChatModel + tool loop, superseded by `multiagent`.
- **`security/`** — `Executor` resolves a tool YAML recipe, builds the command line, runs
  the process, streams stdout/stderr. Also shell/PTY sessions and auth middleware.
- **`mcp/`** + **`einomcp/`** — native Model Context Protocol server (HTTP/stdio/SSE),
  built-in tool registry, and federation to external MCP servers.
- **`c2/`** — built-in Command & Control: listeners (`tcp_reverse`, `http_beacon`,
  `https_beacon`, `websocket`), sessions, tasks, payload builders, event bus. HITL-gated.
- **`hitl/`** — human-in-the-loop approval gate (tool allowlist + approval modes).
- **`knowledge/`** — RAG over Markdown: indexer (chunk + embed), retriever (MultiQuery
  rewrite → vector fusion → HTTP rerank).
- **`project/`** — cross-session **blackboard** of facts + edge graph (attack paths).
- **`attackchain/`** — auto-extracts a target/tool/vuln graph from each conversation.
- **`database/`** — SQLite persistence (hand-coded schema, no migration framework).
- **`handler/`** — HTTP API handlers, one file per surface (agent, multi_agent, c2, hitl,
  knowledge, conversation, batch_task_manager, …).
- **`robot/`** — IM adapters (WeChat/WeCom, DingTalk, Lark, Telegram, Slack, Discord, QQ).
- **`workflow/`** — visual DAG orchestration (Start/Agent/Tool/Condition/HITL/Output).
- **`app/`** — application lifecycle: init, route wiring, shutdown.

### Filesystem-driven assets (add features here, not in Go)

- **`roles/`** — YAML testing personas (name, description, user_prompt, tools[], enabled).
- **`skills/`** — Agent-Skills packs; one `SKILL.md` (YAML front matter + Markdown) per dir,
  loaded on demand by the Eino `skill` tool.
- **`agents/`** — Markdown multi-agent definitions (`orchestrator*.md` + specialist
  sub-agents: recon, penetration, triage, reporting, …).
- **`tools/`** — 90+ YAML tool recipes (nmap, sqlmap, nuclei, metasploit, …).
- **`knowledge_base/`** — Markdown content auto-indexed into the vector store on scan.
- **`mcp-servers/`** — standalone MCP servers; **`plugins/`** — e.g. the Burp extension.
- **`web/`** — `static/` SPA assets + `templates/` Go HTML templates.

Design references live in **`docs/`** (`MULTI_AGENT_EINO.md`, `robot.md`, `VISION.md`,
`workflow-graph.md`, `frontend-i18n.md`).

## Conventions worth repeating

- **To add a tool/role/skill/agent, drop a file** in the matching directory — don't wire
  it into Go. The loaders parse at startup; missing/invalid entries are skipped gracefully.
- Skill packs follow the Anthropic Agent-Skills spec (`SKILL.md`), not a custom format.
- Sub-agents see **only their `task` description**, not the parent conversation — the
  orchestrator must pack known facts, scope boundary, and success criteria into it.
- SSE is the streaming backbone (chat, MCP, C2 events); Gin handles the long-lived conns.
- SQLite is the single store; schema is created in code, so schema changes are code changes.

## Agent skills

### Issue tracker

Issues and PRDs live in GitHub Issues (`Notyet1307/CyberGO`), managed via the `gh` CLI.
External PRs are **not** a triage surface. See `docs/agents/issue-tracker.md`.

### Triage labels

Default vocabulary: `needs-triage`, `needs-info`, `ready-for-agent`, `ready-for-human`,
`wontfix`. See `docs/agents/triage-labels.md`.

### Domain docs

Single-context: one `CONTEXT.md` + `docs/adr/` at the repo root. See `docs/agents/domain.md`.
