# agent-swarm

One-person dev team powered by Codex and Claude. You describe a task, agents write the code and open PRs, reviews run automatically, you merge.

No dashboards. No cloud. Just bash, tmux, and GitHub.

## The workflow


## Giving agents context

Agents start fresh each run. Without context they make generic decisions that don't fit your project. The more you give them upfront, the fewer corrections you need mid-run.

### AGENTS.md — project context

Put `AGENTS.md` in the repo root. Every agent reads it before starting. Cover:

- Stack and versions
- Directory layout
- Naming conventions
- How to run tests and type checks
- Things agents commonly get wrong in your codebase

```markdown
# AGENTS.md

## Stack
- NestJS 10, TypeScript 5, PostgreSQL 15
- Frontend: Next.js 14, Tailwind CSS

## Structure
- src/services/ — business logic
- src/dto/ — request/response types
- src/entities/ — database models

## Before committing
- Run: npm run type-check && npm test
- No console.log in production code
- All async functions must have try/catch
```

### Business context in the prompt

When you spawn a task, the prompt is the only channel for business context. Be explicit:

```bash
bash ~/.agent-swarm/scripts/spawn-agent.sh \
  --project myproject \
  add-invoice-pdf \
  codex \
  "Add PDF export for invoices.

  Business context: customers export invoices for accounting software.
  The PDF must match the on-screen layout exactly.
  Use the existing InvoiceService — don\'t create a new one.
  Prices are always in EUR with 2 decimal places.
  File: src/services/invoice.service.ts"
```

The more specific the prompt, the less the agent has to guess. Task scope, file to start in, constraints, edge cases — all of it.

### Product context file

For larger projects, maintain a shared context file that gets prepended to every prompt:

```markdown
# product-context.md

## What we\'re building
B2B SaaS for managing service contracts. Main users: operations managers.

## Core rules
- Multi-tenant: every query must be scoped to tenantId
- Prices are stored in cents, displayed in EUR
- All dates in UTC, displayed in user\'s timezone

## What not to touch
- auth/ — handled separately, don\'t modify
- billing/ — Stripe integration, changes require manual review
```

Then reference it in spawn-agent.sh prompts:

```bash
CONTEXT=$(cat ~/projects/myproject/product-context.md)
bash ~/.agent-swarm/scripts/spawn-agent.sh \
  --project myproject \
  fix-tenant-leak \
  codex \
  "$CONTEXT

  Fix: ProductRepository.findAll() is missing tenantId filter."
```
