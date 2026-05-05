---
name: gomuos-team
description: Complete overview of the GomuOS agent team — structure, roles, how tickets originate, the full pipeline from audit to implementation to testing, Lab API, cron wiring, and how to add or modify agents. Read this first when working with or modifying the GomuOS agent system.
---

# GomuOS Agent Team

A team of specialized Claude Code agents that autonomously audit, implement, and validate changes to the GomuOS food ordering platform. All agent actions require human approval via lab tickets at `lab.gomuos.com` before execution.

## Team Structure

```
gomuos-team/
├── hunters/       ← proactive auditors, run on cron, find issues and create tickets
│   ├── gomuos-ui-reviewer
│   ├── gomuos-admin-specialist
│   ├── gomuos-checkout-specialist
│   ├── gomuos-menu-specialist
│   ├── gomuos-orders-specialist
│   └── gomuos-wolt-specialist
├── devs/          ← implementers, triggered by approved tickets
│   ├── gomuos-frontend-developer
│   └── gomuos-backend-developer
├── validators/    ← validate after implementation
│   ├── gomuos-code-reviewer
│   └── gomuos-playwright-tester
├── manager/       ← orchestration, routes tickets, reads GOALS.md
│   └── gomuos-manager
└── infra/         ← GomuOS deployment knowledge
    └── gomuos-infra
```

## What Each Group Does

### Hunters (run on cron)
Proactively read their domain code and create tickets for bugs, improvements, and UX issues they find. **They are not triggered by git commits** — they audit the existing codebase on a schedule. Each hunter knows its domain deeply and applies that knowledge to spot real issues.

| Hunter | Domain |
|--------|--------|
| `gomuos-ui-reviewer` | UI/UX quality across all customer-facing flows |
| `gomuos-admin-specialist` | Admin panel: shop config, work hours, employees, accounts |
| `gomuos-checkout-specialist` | Checkout flow, Bambora payment, promo codes |
| `gomuos-menu-specialist` | Food, beverages, fillings, sauces, combos, recommendations |
| `gomuos-orders-specialist` | Order lifecycle, WebSocket updates, order history |
| `gomuos-wolt-specialist` | Wolt delivery webhooks and API client |

### Devs (triggered by approved tickets)
Implement changes. Do NOT write code until a ticket is approved by the human.

- `gomuos-frontend-developer` — Next.js 14, TypeScript, Mantine UI 7
- `gomuos-backend-developer` — Spring Boot 3.1, Java 17, Hibernate, MySQL

### Validators (triggered after implementation)
Run after a developer deploys to staging.

- `gomuos-code-reviewer` — security, API contracts, conventions
- `gomuos-playwright-tester` — E2E tests on `testapp.gomuos.com`

### Manager
Orchestrates the team: routes unassigned tickets to the right agent, reads `GOALS.md` for current priorities, and creates tickets for uncaptured work.

## How Tickets Originate

Tickets come from three sources:

**1. Hunters (primary source)**
A hunter reads its domain code, finds an issue, and creates a ticket:
```
gomuos-checkout-specialist reads checkout components
  → finds: "Betal-knappen mangler loading state"
  → POST /api/tickets { assignedAgent: "gomuos-frontend-developer" }
  → human approves → dev implements
```

**2. Validators (follow-up tickets)**
After a dev implements and deploys to staging, they create tickets for validators:
```
gomuos-frontend-developer deploys fix
  → POST /api/tickets { assignedAgent: "gomuos-playwright-tester" }
  → POST /api/tickets { assignedAgent: "gomuos-code-reviewer" }
```
If a validator finds a problem, it creates a new ticket back to the dev.

**3. Manager (gaps)**
The manager creates tickets for work that hunters haven't captured — spotted via git activity or GOALS.md priorities.

## Full Pipeline

```
[Cron] Hunter reads domain code
         ↓ finds issue
      Creates ticket (assignedAgent: gomuos-*-developer)
         ↓
      Human approves in lab.gomuos.com
         ↓
      Dev implements → deploys to testapp.gomuos.com
      Dev creates follow-up tickets:
        → gomuos-playwright-tester
        → gomuos-code-reviewer
         ↓
      Human approves both
         ↓
      Playwright runs E2E tests → pass/fail ticket
      Code reviewer reviews → issues ticket or clear
         ↓
      Done (or loop back to dev if issues found)
```

## Ticket Routing (Manager logic)

Hunters **create** tickets — they are never assigned tickets. The manager routes unassigned tickets to devs and validators.

```
Unassigned ticket arrives (no assignedAgent)
  ↓
gomuos-manager reads ticket
  ↓
Is it an implementation task?
  ├── frontend change needed → gomuos-frontend-developer
  └── backend change needed  → gomuos-backend-developer
  ↓
Is it a validation task?
  ├── code review needed     → gomuos-code-reviewer
  └── E2E test needed        → gomuos-playwright-tester
  ↓
Is it a vague/complex ticket the human created manually?
  → route to the relevant hunter for analysis + sub-ticket creation:
    ├── checkout/payment/promo     → gomuos-checkout-specialist
    ├── wolt/delivery/webhook      → gomuos-wolt-specialist
    ├── menu/food/beverage/filling → gomuos-menu-specialist
    ├── order lifecycle/ws         → gomuos-orders-specialist
    └── admin/shop config          → gomuos-admin-specialist
```

In normal operation, hunters run on cron and create tickets directly with `assignedAgent` already set — the manager doesn't need to touch them.

## Lab API

- **URL**: `https://lab.gomuos.com/api`
- **Auth**: `Authorization: Bearer $LAB_API_KEY`

| Endpoint | Use |
|----------|-----|
| `GET /api/tickets?status=pending` | Fetch open tickets |
| `POST /api/tickets` | Create ticket |
| `PATCH /api/tickets/{id}` | Assign agent, mark done |
| `GET /api/tickets/{id}/poll` | Poll for human approval |
| `POST /api/jobs` | Register cron job |
| `PATCH /api/jobs/{name}` | Update job status |

**Ticket format:**
```json
{
  "title": "Verb-first short description",
  "description": "What was found, why it matters, what to do",
  "severity": "info | warning | error | critical",
  "jobName": "gomuos-<agent-name>",
  "assignedAgent": "gomuos-<target-agent>"
}
```

## Cron Integration (Vegapunk)

Agents run as cron jobs in `vegapunk/src/infra/cron.ts`. Pattern:
```typescript
async function runAgent() {
  await lab.updateJobStatus("gomuos-agent-name", "running");
  try {
    // spawn Claude Code with the agent skill
    await lab.updateJobStatus("gomuos-agent-name", "success", output);
  } catch (err) {
    await lab.updateJobStatus("gomuos-agent-name", "error", err.message);
  }
}
```

Current cron jobs: `health-monitor` (every 15 min) + alle 6 hunters (dagligt, staggered 2 min apart). Menu-specialist kører ugentligt (mandag).

## Project Goals

Each project repo can have a `GOALS.md` in its root. The manager reads this to understand current priorities:
```markdown
# Current Goals
- [ ] <active goal>
- [x] <completed goal>
## Priority: <what matters most right now>
```

## Platform

| | |
|--|--|
| VPS | `46.224.215.213` — `ssh root@46.224.215.213` |
| Production | `ordrupspizza.dk` |
| Staging | `testapp.gomuos.com` |
| Lab | `lab.gomuos.com` |
| Vegapunk | systemd on VPS — `/home/vegapunk/vegapunk/` |

## Skills Location

```
dotfiles: .dotfiles/.claude/skills/gomuos-team/
local:    ~/.claude/skills/  (installed flat by sync-skills)
```

## How to Add a New Agent

1. Create `dotfiles/.claude/skills/gomuos-team/<group>/<agent-name>/SKILL.md`
2. Update `manager/gomuos-manager/references/agents.md` with routing rules
3. Add cron job in `vegapunk/src/infra/cron.ts` if it runs on a schedule
4. Commit, push → run sync-skills on VPS

## How to Modify an Existing Agent

1. Edit the skill file in `dotfiles/.claude/skills/gomuos-team/<group>/<agent-name>/SKILL.md`
2. If routing changed → update `manager/gomuos-manager/references/agents.md`
3. Commit, push → sync-skills on VPS
