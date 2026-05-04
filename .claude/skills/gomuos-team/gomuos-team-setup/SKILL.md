---
name: gomuos-team-setup
description: Complete system overview of the GomuOS agent team — agents, skills, ticket flow, cron wiring, and how to add or modify agents. Use when making architectural changes to the agent team, adding new agents, updating routing logic, or onboarding to the system.
---

# GomuOS Agent Team — System Overview

The GomuOS agent team is a set of specialized Claude Code skills that autonomously monitor, review, and implement changes to the GomuOS food ordering platform. Agents communicate via lab tickets at `lab.gomuos.com`. Every action requires human approval through the ticket system before execution.

## Architecture

```
Vegapunk cron scheduler
        |
        v
gomuos-manager  (runs on schedule, routes tickets)
        |
        ├── gomuos-ui-reviewer       (reviews frontend UI/UX)
        ├── gomuos-code-reviewer     (reviews security + conventions)
        ├── gomuos-frontend-developer (implements frontend changes)
        ├── gomuos-backend-developer  (implements backend changes)
        └── gomuos-playwright-tester  (E2E tests on staging)
```

Agents do not call each other directly. All communication goes through lab tickets:
1. Agent creates a ticket → assigns `assignedAgent`
2. Human approves in `lab.gomuos.com`
3. Target agent picks it up on next run

## Agent Roster

| Agent | Skill file | Trigger | Does |
|-------|-----------|---------|------|
| `gomuos-manager` | `gomuos-team/gomuos-manager/` | Cron / manual | Routes unassigned tickets, checks git, reads GOALS.md |
| `gomuos-ui-reviewer` | `gomuos-team/gomuos-ui-reviewer/` | Frontend commits | Reviews UI with Vercel guidelines + GomuOS UX rules |
| `gomuos-code-reviewer` | `gomuos-team/gomuos-code-reviewer/` | Any commit | Security, API contract, convention checks |
| `gomuos-frontend-developer` | `gomuos-team/gomuos-frontend-developer/` | Approved ticket | Implements frontend features, deploys to staging |
| `gomuos-backend-developer` | `gomuos-team/gomuos-backend-developer/` | Approved ticket | Implements backend features, deploys to staging |
| `gomuos-playwright-tester` | `gomuos-team/gomuos-playwright-tester/` | Post-deploy ticket | E2E tests on testapp.gomuos.com |

## Skills Location

```
dotfiles: C:\Users\shpat\Desktop\projects\.dotfiles\.claude\skills\gomuos-team\
VPS:      ~/.claude/skills/  (installed flat by sync-skills)
```

## Dependency Skills (installed from GitHub)

These are referenced by the developer agents — install with `npx skills add`:

| Skill | Source | Used by |
|-------|--------|---------|
| `java-spring-development` | `mindrally/skills@java-spring-development` | gomuos-backend-developer |
| `java-spring-boot` | `pluginagentmarketplace/custom-plugin-java@java-spring-boot` | gomuos-backend-developer |
| `e2e-playwright-testing` | `asyrafhussin/agent-skills@e2e-playwright-testing` | gomuos-playwright-tester |
| `web-design-guidelines` | `vercel-labs/agent-skills@web-design-guidelines` | gomuos-ui-reviewer |

## Lab API

- **URL**: `https://lab.gomuos.com/api`
- **Auth**: `Authorization: Bearer $LAB_API_KEY`
- **Key endpoints**:
  - `GET /api/tickets?status=pending` — open tickets
  - `POST /api/tickets` — create ticket
  - `PATCH /api/tickets/{id}` — update (assign agent, mark done)
  - `GET /api/tickets/{id}/poll` — poll for approval status
  - `POST /api/jobs` — register cron job
  - `PATCH /api/jobs/{name}` — update job status

## Ticket Fields

| Field | Values | Notes |
|-------|--------|-------|
| `title` | string | Short imperative, starts with a verb |
| `description` | string | What, why, what to do |
| `severity` | `info \| warning \| error \| critical` | Reflects urgency |
| `status` | `pending \| approved \| rejected \| done` | Human controls approved/rejected |
| `jobName` | string | Which agent created it |
| `assignedAgent` | string | Which agent should act on it |

## Cron Integration (Vegapunk)

Agents run as cron jobs in `vegapunk/src/infra/cron.ts`. Each job:
1. Calls `lab.updateJobStatus(jobName, "running")`
2. Does its work
3. Calls `lab.updateJobStatus(jobName, "success" | "error", output)`

Current jobs: `health-monitor` (every 15 min).
Planned: `gomuos-manager` (daily or on-demand).

## How to Add a New Agent

1. **Create the skill** in `dotfiles/.claude/skills/gomuos-team/<agent-name>/SKILL.md`
   - Frontmatter: `name`, `description` (include trigger conditions)
   - Body: environment, workflow, ticket creation format, report format
2. **Add to agents.md** in `gomuos-manager/references/agents.md` — routing rules
3. **Wire as cron job** (optional) in `vegapunk/src/infra/cron.ts` if it should run on a schedule
4. **Commit and push** dotfiles → run `/sync-skills` on VPS to install

## How to Modify an Existing Agent

1. Edit the skill file in `dotfiles/.claude/skills/gomuos-team/<agent-name>/SKILL.md`
2. If routing logic changed, also update `gomuos-manager/references/agents.md`
3. Commit, push, run `/sync-skills` on VPS

## How to Set Project Goals

Create a `GOALS.md` in the active project repo root (e.g., `order-backend/GOALS.md`). The manager reads this on every run to understand current priorities. Format:

```markdown
# Current Goals

- [ ] <active goal>
- [x] <completed goal>

## Priority: <what matters most right now>
```

## Platform

- **VPS**: `46.224.215.213` (k3s, gomuos namespace)
- **Staging**: `testapp.gomuos.com` (testapp-backend + testapp-frontend pods)
- **Production**: `ordrupspizza.dk` (ordrupspizza-backend + ordrupspizza-frontend pods)
- **Lab**: `lab.gomuos.com` (gomuos-lab pod)
- **Vegapunk**: systemd service on VPS, `/home/vegapunk/vegapunk/`
