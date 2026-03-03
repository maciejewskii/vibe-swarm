# agent-swarm

One-person dev team running on Codex and Claude. You write a task, the swarm spawns an agent in a tmux session, it opens a PR, reviews run automatically, you merge.

No dashboards. No cloud. Just bash, tmux, and GitHub.

## How it works

```
spawn-agent.sh  →  git worktree + tmux session
                →  Codex or Claude writes code, opens PR

check-agents.sh →  runs every 10 min via cron
                →  CI passed? → trigger local-review.sh
                →  all reviews green → notify "ready for review"
                →  CI or review failed → respawn with failure context
```

Each task runs in its own git worktree so multiple agents can work in parallel without stepping on each other.

## Requirements

- bash 5+
- tmux
- jq
- gh (GitHub CLI, authenticated)
- codex CLI — `npm install -g @openai/codex`
- claude CLI — `npm install -g @anthropic-ai/claude-code`

## Setup

```bash
git clone https://github.com/YOUR_USER/agent-swarm ~/.agent-swarm
```

Add to `~/.zshrc` or `~/.bashrc`:

```bash
export SWARM_HOME=~/.agent-swarm
```

Copy and edit the example project config:

```bash
cp ~/.agent-swarm/projects/example.json ~/.agent-swarm/projects/myproject.json
```

Add a cron:

```
*/10 * * * * SWARM_HOME=~/.agent-swarm bash ~/.agent-swarm/scripts/monitor-loop.sh >> ~/.agent-swarm/monitor.log 2>&1
```

## Project config

```json
{
  "name": "myproject",
  "repo": "/path/to/local/repo",
  "remote": "github-user/repo-name",
  "baseBranch": "main",
  "worktreeBase": "/path/to/worktrees",
  "tasksFile": "/path/to/worktrees/active-tasks.json",
  "logDir": "/path/to/worktrees/logs",
  "maxAttempts": 3,
  "reviewMode": "local",
  "requireGemini": false,
  "reviewContexts": {
    "codex": "local/codex-review",
    "claude": "local/claude-review",
    "gemini": "local/gemini-review",
    "screenshot": "local/screenshot-gate"
  }
}
```

## Spawning a task

```bash
bash ~/.agent-swarm/scripts/spawn-agent.sh \
  --project myproject \
  fix-login-bug \
  codex \
  "Fix the login bug where users get logged out on page refresh. Check auth/session.ts."
```

Agent type: `codex` | `claude` | `auto`

`auto` routes frontend/UI tasks (React, CSS, Tailwind, components) to Claude and everything else to Codex.

Use `--fix` for `fix/task-id` branches instead of `feat/task-id`.

## Reviews

After CI passes, `local-review.sh` runs automatically. It sets four GitHub commit statuses:

### local/codex-review

Codex reads the PR diff and outputs `VERDICT: PASS` or `VERDICT: FAIL` with a list of bugs, regressions, or security issues. Style nitpicks are ignored. The result is posted as a PR comment.

### local/claude-review

Claude runs in critical-only mode — it only flags things that will crash in production, cause data loss, create a security vulnerability, or break existing functionality. If nothing critical, it passes. Also posted as a PR comment.

### local/screenshot-gate

If the PR touches UI files (`.tsx`, `.jsx`, `.css`, `.scss`, `.vue`, `.svelte`), it checks whether the PR description contains a screenshot. No screenshot → fail. Non-UI PRs pass automatically.

### local/gemini-review

Checks whether Gemini Code Assist (GitHub App) has reviewed or commented on the PR. If `requireGemini: true` in the project config, this gate must pass before the task is marked `review_ready`. The app is free to install at [github.com/apps/gemini-code-assist](https://github.com/apps/gemini-code-assist).

To require Gemini before merge:

```json
"requireGemini": true
```

### GitHub token permissions

The `gh` CLI needs permission to set commit statuses. Make sure your token has the `repo` scope (or `statuses:write` for fine-grained tokens).

## Notifications

When something happens, a line is appended to `$SWARM_HOME/notifications.pending`:

```
[2026-03-03T10:00:00Z] NOTIFY: [myproject] Task fix-login-bug PR #12 ready for review
[2026-03-03T10:05:00Z] NOTIFY: [myproject] Task fix-login-bug PR #12 Claude review FAILED
[2026-03-03T10:10:00Z] NOTIFY: [myproject] AGENT EXHAUSTED fix-login-bug — all 3 attempts used
```

Read and clear it however you want — cron, webhook, custom script.

## Auto-retry

When CI or a review fails, the swarm respawns the agent with the failure context (review comments, failed check names, original task prompt) injected into the new prompt. This repeats up to `maxAttempts` times.

After all attempts are exhausted, you get an `AGENT EXHAUSTED` notification and handle it yourself.

Manual respawn with a corrected prompt:

```bash
bash ~/.agent-swarm/scripts/respawn-agent.sh \
  --project myproject \
  fix-login-bug \
  "Previous attempt broke the tests. Fix only session handling, don't touch the router."
```

## Mid-task steering

If the agent is going in the wrong direction, send it a message directly in tmux:

```bash
tmux send-keys -t fix-login-bug "Don't touch session.ts, the bug is in middleware/auth.ts" Enter
```

The inner session name matches the task ID.

## Checking status

```bash
bash ~/.agent-swarm/scripts/check-agents.sh --project myproject
# or all projects at once
bash ~/.agent-swarm/scripts/check-agents.sh --all
```

## Directory structure

```
~/.agent-swarm/
  scripts/
    spawn-agent.sh       # create worktree + start agent
    run-agent.sh         # run codex/claude in tmux, handle exit
    check-agents.sh      # poll PR/CI status, trigger reviews, auto-retry
    local-review.sh      # codex + claude + gemini + screenshot review
    respawn-agent.sh     # retry a failed task with a new prompt
    monitor-loop.sh      # cron entrypoint
    cleanup-agents.sh    # remove stale worktrees and tmux sessions
  projects/
    example.json
  .prompts/
    myproject/
      fix-login-bug.txt  # task prompt, auto-created by spawn-agent.sh
  notifications.pending  # append-only event log
  monitor.log            # full scheduler output
  patterns.log           # hints accumulated from successful runs
```

## AGENTS.md

Put an `AGENTS.md` in your repo root. The swarm reads it at the start of every task. Less context = more mistakes.

```markdown
# AGENTS.md

## Stack
- Backend: NestJS, TypeScript, PostgreSQL
- Frontend: Next.js, Tailwind

## Conventions
- Services in src/services/
- Always add error handling to async functions
- Run `npm run type-check` before committing
```

## Tips

- Keep tasks small. "Refactor the entire auth system" will fail. "Extract token refresh into a separate service" will work.
- Gemini Code Assist is free and independent from Codex/Claude — it catches different things. Worth installing.
- `patterns.log` accumulates hints from successful runs and gets injected into retry prompts automatically.
- The `auto` agent type works well as a default once you know your codebase is split between backend and frontend.

## Orchestration with OpenClaw

The swarm is headless by design — it doesn’t know what to build or when. That’s the orchestrator’s job.

In this setup, [OpenClaw](https://github.com/openclaw/openclaw) acts as the brain. It runs as a persistent AI agent connected to Telegram (or any other channel). It reads `notifications.pending` on a heartbeat schedule, decides what needs attention, spawns tasks, and steers agents mid-run.

### How the loop works

```
You (Telegram)  →  OpenClaw  →  spawn-agent.sh  →  Codex/Claude
                                                  →  PR opens
notifications.pending  →  OpenClaw heartbeat  →  you get pinged
You approve  →  gh pr merge
```

1. You describe a task to OpenClaw in plain language
2. OpenClaw calls `spawn-agent.sh` with the right project, task ID, agent type, and prompt
3. The agent works, opens a PR
4. `check-agents.sh` runs on cron, reviews complete, a line is written to `notifications.pending`
5. OpenClaw reads `notifications.pending` on the next heartbeat and messages you
6. You review the PR and merge

If a review fails or an agent gets stuck, OpenClaw handles the respawn — with context from the failure injected into the new prompt.

### OpenClaw HEARTBEAT.md example

```markdown
## Agent Swarm notifications

Check ~/. agent-swarm/notifications.pending.
- If it has content: read it, send each line to Telegram, then clear the file.
- If empty: do nothing.
```

### Why this works

The swarm handles the mechanical parts: worktrees, tmux sessions, PR lifecycle, commit statuses, retry logic. OpenClaw handles judgment: what to build, when to intervene, when to escalate. Neither needs to know the internals of the other.

You end up with a setup where you can describe a task over Telegram while commuting, and by the time you're back at your desk there's a PR waiting for review.
