# agent-swarm

One-person dev team powered by Codex and Claude. You describe a task, agents write the code and open PRs, reviews run automatically, you merge.

No dashboards. No cloud. Just bash, tmux, and GitHub.

## The workflow

The idea comes from how Elvis Presley ran his operation — one person at the center, everything else delegated. You stay at a high level. Agents handle the implementation.

```
You (phone/Telegram)
  ↓
OpenClaw (AI orchestrator)
  ↓ spawn-agent.sh
Codex / Claude (in tmux + git worktree)
  ↓ opens PR
check-agents.sh (cron, every 10 min)
  ↓ CI passed → local-review.sh
  ↓ Codex review + Claude review + Gemini + screenshot gate
  ↓ all green → notifications.pending
OpenClaw heartbeat
  ↓ reads notifications.pending
  ↓ pings you on Telegram
You
  → review the PR, merge
```

If CI or a review fails, the swarm respawns the agent automatically with the failure context injected into the prompt. Up to `maxAttempts` retries. After that, you get an `AGENT EXHAUSTED` ping and decide what to do.

You can run multiple agents in parallel. Each task gets its own git worktree and tmux session.

## OpenClaw as the orchestrator

[OpenClaw](https://github.com/openclaw/openclaw) is a self-hosted AI agent that acts as the brain of the operation. It connects to Telegram (or Slack, Signal, Discord) and runs continuously.

It handles:
- Spawning tasks when you describe them in natural language
- Reading `notifications.pending` on a heartbeat and forwarding events to your phone
- Mid-run steering (send a correction to an agent without touching the terminal)
- Deciding when to intervene vs. let the swarm retry automatically

Without OpenClaw you can still run the swarm manually — it’s just bash scripts. OpenClaw is the layer that makes it hands-free.

### OpenClaw HEARTBEAT.md setup

```markdown
## Agent swarm

Check $SWARM_HOME/notifications.pending.
- If it has content: read it line by line, send each NOTIFY line to Telegram, then clear the file.
- If empty: skip.
```

### Instant notifications (no cron delay)

`notify-instant.sh` uses a filesystem watcher (macOS `launchd` WatchPaths or Linux `inotifywait`) to fire the moment something is appended to `notifications.pending`. No waiting for the next cron tick.

launchd plist example (macOS):

```xml
<key>WatchPaths</key>
<array>
    <string>/path/to/.agent-swarm/notifications.pending</string>
</array>
<key>ProgramArguments</key>
<array>
    <string>bash</string>
    <string>/path/to/.agent-swarm/scripts/notify-instant.sh</string>
</array>
```

Set `NOTIFY_CHANNEL` and `NOTIFY_TARGET` env vars, plus `OPENCLAW_CONFIG` pointing to your OpenClaw config file.

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

After CI passes, `local-review.sh` runs automatically and sets four GitHub commit statuses.

### local/codex-review

Codex reads the PR diff and outputs `VERDICT: PASS` or `VERDICT: FAIL` followed by a list of bugs, security issues, or regressions. Style nitpicks are ignored. The result is posted as a PR comment.

### local/claude-review

Claude runs in critical-only mode. It only reports things that will crash in production, cause data loss, create a security vulnerability, or break existing functionality. If nothing critical is found, it passes. Also posted as a PR comment.

### local/screenshot-gate

If the PR touches UI files (`.tsx`, `.jsx`, `.css`, `.scss`, `.vue`, `.svelte`), it checks whether the PR description contains a screenshot. No screenshot → fails. Non-UI PRs pass automatically.

### local/gemini-review

Checks whether [Gemini Code Assist](https://github.com/apps/gemini-code-assist) (GitHub App, free) has reviewed or commented on the PR. With `requireGemini: true` in the project config, this gate must pass before the task is marked `review_ready`. Gemini is independent from Codex and Claude and tends to catch different things.

### GitHub token permissions

The `gh` CLI needs permission to set commit statuses. Make sure your token has the `repo` scope (or `statuses:write` for fine-grained tokens).

## Auto-retry

When CI or a review fails, `check-agents.sh` automatically calls `respawn-agent.sh` with a new prompt that includes:
- Review comments from the PR
- Inline comments from the diff
- Names of failed checks
- The original task prompt
- Hints from `patterns.log` (see below)

This repeats up to `maxAttempts` times. After that, you get an `AGENT EXHAUSTED` notification.

Manual respawn:

```bash
bash ~/.agent-swarm/scripts/respawn-agent.sh \
  --project myproject \
  fix-login-bug \
  "Previous attempt broke the tests. Fix only session handling, don\'t touch the router."
```

## patterns.log

Each time an agent succeeds after a retry, a hint is written to `$SWARM_HOME/patterns.log`. On the next retry for a similar task, these hints are injected into the prompt automatically.

Over time this file becomes a list of common failure modes and what fixed them. You can also edit it manually to add project-specific guidance that should always be considered during retries.

## Mid-task steering

If an agent is going in the wrong direction, send a correction directly into its tmux session:

```bash
tmux send-keys -t fix-login-bug "Don\'t touch session.ts, the bug is in middleware/auth.ts" Enter
```

The inner session name matches the task ID.

## Checking status

```bash
bash ~/.agent-swarm/scripts/check-agents.sh --project myproject
# or all projects
bash ~/.agent-swarm/scripts/check-agents.sh --all
```

## Notifications

When something happens, a line is appended to `$SWARM_HOME/notifications.pending`:

```
[2026-03-03T10:00:00Z] NOTIFY: [myproject] Task fix-login-bug PR #12 ready for review
[2026-03-03T10:05:00Z] NOTIFY: [myproject] Task fix-login-bug PR #12 Claude review FAILED
[2026-03-03T10:10:00Z] NOTIFY: [myproject] AGENT EXHAUSTED fix-login-bug — all 3 attempts used
```

Read and clear it however you want. With OpenClaw, this happens automatically via heartbeat or `notify-instant.sh`.

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
    notify-instant.sh    # instant notification on pending file change
    cleanup-agents.sh    # remove stale worktrees and tmux sessions
  projects/
    example.json
  .prompts/
    myproject/
      fix-login-bug.txt  # task prompt, auto-created by spawn-agent.sh
  notifications.pending  # append-only event log
  monitor.log
  patterns.log
```

## AGENTS.md

Put an `AGENTS.md` in your repo root. The swarm reads it at the start of every task.

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

- Keep tasks small and scoped. "Refactor the entire auth system" will fail. "Extract token refresh into a separate service" will work.
- Gemini Code Assist is free and independent — it catches different things than Codex and Claude. Worth installing.
- `auto` agent routing works well as a default once you have a clear frontend/backend split.
- The whole thing runs on a laptop or a cheap VPS. No cloud infra needed.
