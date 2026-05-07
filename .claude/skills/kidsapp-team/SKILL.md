---
name: kidsapp-team
description: Complete overview of the KidsApp agent team — structure, roles, ticket pipeline, Unity build flow, and how to add or modify agents. Read this first when working with or modifying the KidsApp agent system.
---

# KidsApp Agent Team

A team of specialized Claude Code agents that build and maintain the KidsApp Unity educational app. The app is built as Unity 6.3 WebGL, deployed to `kidsapp.gomuos.com` via GameCI → GHCR → ArgoCD → k3s. All agent actions require human approval via lab tickets at `lab.gomuos.com/kidsapp` before execution.

This is a **separate workspace from gomuos and strawhats**. Tickets live under workspace slug `kidsapp`.

## Team Structure

```
kidsapp-team/
├── manager/
│   └── kidsapp-manager              ← orchestrator, routes tickets, reads GOALS.md
├── scouts/                           ← proactive idea-researchers (cron, weekly)
│   ├── kidsapp-bimiboo-scout        ← Bimi Boo + lignende app-makere
│   ├── kidsapp-edu-research-scout   ← Montessori, Piaget, pædagogisk forskning
│   └── kidsapp-physical-toy-scout   ← fysisk legetøj → digital oversættelse
├── devs/
│   └── kidsapp-unity-developer      ← C# scripts, scenes, prefabs, ScriptableObjects
├── validators/
│   └── kidsapp-code-reviewer        ← Unity best practices, performance, security
└── infra/
    └── kidsapp-infra                ← deploy knowledge (build pipeline, k3s, ArgoCD)
```

Playwright-tester / WebGL-tester comes later when there is enough gameplay to audit.

## What Each Group Does

### Manager
Orchestrates the team: routes unassigned tickets to the right agent, reads `~/.claude/skills/kidsapp-team/GOALS.md` for current priorities, creates tickets for uncaptured work. Never writes code.

### Scouts (run on cron, weekly)
Proactively research external sources for mini-game ideas and create idea-tickets in the kidsapp workspace. Each scout has a different angle so we don't get monoculture ideas.

| Scout | Angle |
|-------|-------|
| `kidsapp-bimiboo-scout` | What Bimi Boo and competitors already ship |
| `kidsapp-edu-research-scout` | What pedagogy (Montessori, Piaget, etc.) suggests |
| `kidsapp-physical-toy-scout` | Physical toys/games translated into digital mechanics |

Scouts create tickets with `assignedAgent: null` — the human approves the idea, then the manager routes approved ideas to `kidsapp-unity-developer`.

### Devs
Implement approved tickets. Do NOT write code until a ticket is approved by the human.

| Agent | Domain |
|-------|--------|
| `kidsapp-unity-developer` | Unity 6.3, C# scripts, scenes, prefabs, UI Toolkit, ScriptableObjects |

### Validators
Review what the dev produced.

| Agent | Domain |
|-------|--------|
| `kidsapp-code-reviewer` | C# code quality, Unity API misuse, performance, security |

### Infra (knowledge skill)
Reference for build pipeline, GameCI, k3s deployment, ArgoCD wiring. Not a runnable agent — it is loaded by other agents who need deployment knowledge.

## Full Pipeline

```
[Human writes ticket via lab.gomuos.com/kidsapp]
   ↓
   Auto-routed by kidsapp-manager → kidsapp-unity-developer (or human assigns directly)
   ↓
   Human approves
   ↓
   ticket-dispatcher (Vegapunk cron) spawns kidsapp-unity-developer
   ↓
   Implements in /home/vegapunk/projects/kids-app
   Pushes to main → GitHub Actions builds WebGL via GameCI (~6 min)
                  → Docker image to GHCR
                  → ArgoCD syncs (~3 min)
                  → Live at kidsapp.gomuos.com
   ↓
   Dev creates follow-up ticket → kidsapp-code-reviewer
   ↓
   Human approves → reviewer reads diff, creates issue tickets if needed
```

## Ticket Routing (Manager logic)

```
Unassigned ticket arrives (no assignedAgent)
  ↓
kidsapp-manager reads ticket
  ↓
Is it an implementation task? → kidsapp-unity-developer
Is it a review task?          → kidsapp-code-reviewer
```

In normal operation the human writes tickets in lab UI, leaves `assignedAgent` empty, and the manager assigns on its next cron tick (every 2 hours) — or the human assigns directly.

## Ticket Status Flow

```
pending ──→ approved ──→ in_progress ──→ done
   │                                      ↑
   │  human clicks "Spørg agent"
   ├──→ needs_response ──→ in_progress ──→ pending (agent answered)
   │
   └──→ rejected
```

## Lab API

- **URL**: `https://lab.gomuos.com/api`
- **Auth**: `Authorization: Bearer $LAB_API_KEY`
- **Workspace**: always `kidsapp` — every ticket created by this team MUST include `"workspace": "kidsapp"` in the body

| Endpoint | Use |
|----------|-----|
| `GET /api/tickets?workspace=kidsapp&status=<s>` | Fetch tickets |
| `POST /api/tickets` | Create ticket — body must include `"workspace": "kidsapp"` |
| `PATCH /api/tickets/{id}` | Update status, assignedAgent, executionLog |
| `POST /api/tickets/{id}/comments` | Post comment as agent |

**Create ticket:**
```json
{
  "title": "Verb-first short description",
  "description": "What was observed, why it matters, what to do",
  "severity": "info | warning | error | critical",
  "jobName": "kidsapp-<your-name>",
  "assignedAgent": "kidsapp-<target-agent>",
  "workspace": "kidsapp"
}
```

## Cron Integration (Vegapunk)

| Job | Interval |
|-----|----------|
| `kidsapp-manager` | Every 2 hours |
| `kidsapp-bimiboo-scout` | Daily 13:00 |
| `kidsapp-edu-research-scout` | Daily 14:00 |
| `kidsapp-physical-toy-scout` | Daily 15:00 |
| `ticket-dispatcher` | Every 3 min — picks up `approved` and `needs_response` for all workspaces |

The dispatcher chooses `cwd` based on agent name. For all `kidsapp-*` agents it uses `/home/vegapunk/projects/kids-app`.

## Project Goals

Active priorities live in `~/.claude/skills/kidsapp-team/GOALS.md`. The manager reads this to decide what to work on next. Format:

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
| Repo (VPS) | `/home/vegapunk/projects/kids-app` |
| Repo (local) | `C:\Users\shpat\Desktop\projects\kids-app` |
| Live URL | `https://kidsapp.gomuos.com` |
| Stack | Unity 6.3 LTS (6000.3.0f1), C#, WebGL build target |
| Build | GitHub Actions (GameCI) → GHCR → ArgoCD → k3s |
| Lab | `https://lab.gomuos.com/kidsapp` |

## Skills Location

```
dotfiles: .dotfiles/.claude/skills/kidsapp-team/
local:    ~/.claude/skills/  (installed flat by sync-skills)
```

## How to Add a New Agent

1. Create `dotfiles/.claude/skills/kidsapp-team/<group>/<agent-name>/SKILL.md`
2. Update this file's "Team Structure"
3. Update `kidsapp-manager/SKILL.md` routing rules
4. If reactive only — the dispatcher already maps `kidsapp-*` → `kids-app` cwd
5. If runs on schedule — add a cron job in `vegapunk/src/infra/cron.ts`
6. Commit + push, run sync-skills on VPS

## How to Modify an Existing Agent

1. Edit the skill file in `dotfiles/.claude/skills/kidsapp-team/<group>/<agent-name>/SKILL.md`
2. Commit + push → sync-skills on VPS
