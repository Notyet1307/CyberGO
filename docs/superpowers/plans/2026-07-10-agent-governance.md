# Agent Governance Implementation Plan

> For agentic workers: execute this single documentation task inline; no subagent or worktree is needed.

**Goal:** Make AGENTS.md the only repository instruction source for CyberGO.

**Architecture:** Replace the Claude-specific guide with one platform-neutral guide. Preserve project and safety facts, then add the Codex commander, Wayfinder, and agent-tasks operating boundaries.

**Tech Stack:** Markdown, Git, Go toolchain, GitHub CLI, existing agent-tasks runtime.

## Global Constraints

- Preserve the authorized-security-testing, audit, and HITL safety posture.
- Do not change product code, configuration, GitHub Issues, or agent-tasks runtime configuration.
- Do not commit, push, merge, or create a pull request.
- AGENTS.md is the only repository-level agent instruction source after this work.

---

### Task 1: Replace the repository instruction source

**Files:**
- Create: AGENTS.md
- Delete: CLAUDE.md
- Verify: repository root and git status --short

**Interfaces:**
- Consumes: CONTEXT.md, docs/agents/issue-tracker.md, docs/agents/domain.md, config.yaml, and ~/agent-tasks/_template/PROTOCOL.md.
- Produces: one platform-neutral repository instruction document for every future agent.

- [ ] **Step 1: Verify the precondition**

Run:

    test -f CLAUDE.md && test ! -e AGENTS.md && git status --short

Expected: CLAUDE.md exists, AGENTS.md does not exist, and only the approved planning documents are untracked.

- [ ] **Step 2: Create AGENTS.md with these exact sections and rules**

1. Role and authority:
   - State that Codex is CyberGO's project commander and must provide real-worktree evidence.
   - Declare AGENTS.md the sole instruction source and forbid a parallel CLAUDE.md.
   - Limit all security work to authorized assessments; preserve audit and HITL controls.
   - Require CONTEXT.md and relevant docs/adr/ files before design or implementation; code wins over stale docs.

2. Project facts:
   - Module cyberstrike-ai; Go 1.25; run.sh and cmd/server/main.go are entry points.
   - config.yaml owns server and TLS configuration and may contain secrets.
   - Record the handler, multiagent, security, MCP, and database flow.
   - Prefer filesystem-driven Role, Skill, sub-agent, Tool, and knowledge-base changes before Go wiring.
   - Require code-backed treatment for SQLite schema changes.

3. Verification:
   - Include exact commands: go test ./..., go test ./internal/multiagent/..., go test -run TestName ./internal/..., gofmt -l ., go vet ./..., and go build -o cyberstrike-ai cmd/server/main.go.
   - Require narrow checks first and prohibit real-target startup without explicit operational authorization.

4. Command and roadmap workflow:
   - Codex verifies branch, worktree, issue, diff, and checks; it must not silently take over a blocked worker task.
   - GitHub Issues in Notyet1307/CyberGO are live source of truth and use docs/agents/issue-tracker.md.
   - Wayfinder resolves one decision at a time; unresolved fog is not implementation work; a ready wayfinder:task needs user authorization before dispatch.

5. agent-tasks execution boundary:
   - Use ~/agent-tasks/_bin/agent_loop.py and its protocol at ~/agent-tasks/_template/PROTOCOL.md for approved implementation.
   - Turn an agreed issue into scope, file allowlist, anchors, non-goals, acceptance criteria, and runnable checks.
   - Use the judgment lane for intent-level work; use --task-file plus --worker-cmd only for deterministic pinned operations.
   - Check out the intended base branch, review TASK.md, run gate T00N, and obtain human confirmation before confirm T00N --run.
   - Keep task artifacts and state.json under ~/agent-tasks/T00N_*; allow the runtime to create the worktree and agent/T00N_* branch.
   - Use status and watch T00N as evidence; passed and failed_escalate retain the branch and worktree; never merge, push, or delete without explicit approval.
   - State that planner, worker, and reviewer commands in ~/agent-tasks/_bin/agents.conf are runtime settings, distinct from this file, and worker/reviewer must be different model families.

6. Change discipline:
   - No commit, push, merge, PR, GitHub mutation, external contact, destructive Git command, secret disclosure, or unrelated-dirty-change removal without explicit authorization.
   - Keep diffs small, reuse existing code and native features, test meaningful behavior, and report in Simplified Chinese with paths, commands, and observed output.

- [ ] **Step 3: Delete the obsolete instruction source**

Run:

    rm CLAUDE.md

Expected: only AGENTS.md remains as the repository-level instruction file.

- [ ] **Step 4: Verify the documentation migration**

Run:

    test -f AGENTS.md && test ! -e CLAUDE.md
    git diff --check
    git status --short

Expected: the first command exits 0, git diff --check produces no output, and status lists only the approved agent-governance documents plus the replacement.

- [ ] **Step 5: Do not commit**

Leave the changes uncommitted for the user to inspect, as required by the global constraints.

