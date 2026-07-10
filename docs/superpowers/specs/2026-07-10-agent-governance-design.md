# Agent governance design

## Decision

`AGENTS.md` becomes the repository's sole agent instruction source. Remove
`CLAUDE.md` to avoid a second, drifting source of truth.

## Contents

`AGENTS.md` will retain the verified project facts needed to work safely:

- authorized-security-testing scope and HITL/audit boundaries;
- Go build, test, formatting, and configuration commands;
- architecture and filesystem-driven extension points;
- GitHub Issues, `CONTEXT.md`, and ADRs as project sources of truth.

It will also establish the operating model for future work:

- Codex is the project commander: verify state, keep the roadmap current, and
  provide evidence rather than self-certifying results;
- Wayfinder maps decide the route; do not implement from unresolved fog;
- an implementation starts only from a ready issue or approved task contract;
- `agent-tasks` owns the planner-to-worker-to-reviewer loop and worktree
  isolation; the target checkout is not used for process artifacts;
- no commit, push, merge, or pull request without the user's explicit request.

## Non-goals

- No product behavior, configuration, or issue state changes.
- No duplicate per-agent instructions or a second protocol document.

## Verification

Confirm `CLAUDE.md` is absent, `AGENTS.md` is present, the working tree shows
only the intended documentation changes, and the new instructions reference
the existing project paths accurately.
