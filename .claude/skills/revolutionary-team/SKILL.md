---
name: revolutionary-team
description: Complete overview of the Revolutionaries agent team — daily idea-generation crew that scans the world for revenue opportunities. 6 scouts gather evidence, 2 analysts deepen it, 1 contrarian stress-tests it, 1 captain synthesizes the top 3 ideas of the day into lab tickets. Approved ideas flow to the Strawhats team to be built. Read this first when working with or modifying the Revolutionaries system.
---

# Revolutionary Agent Team

A team of specialized Claude Code agents that scan the internet daily for revenue opportunities. Six scouts gather independent evidence, two analysts deepen it, one contrarian stress-tests it, and one captain synthesizes the top 3 ideas of the day into lab tickets at `lab.gomuos.com`.

This is **separate from GomuOS and Straw Hats**:
- **GomuOS team** maintains the existing food-ordering platform (audit & fix)
- **Straw Hats team** turns approved wishes into deployed apps (build)
- **Revolutionaries team** generates the wishes in the first place (ideate)

```
Revolutionaries (ideate) → Human filter → Strawhats (build) → GomuOS (operate)
```

## Team Structure

```
revolutionary-team/
├── captain/
│   └── revolutionary-captain          ← synthesizes top 3 ideas, creates tickets
├── scouts/                             ← gather evidence (no tickets)
│   ├── revolutionary-pain-hunter        — global pains from forums
│   ├── revolutionary-launch-tracker     — products with revenue signals
│   ├── revolutionary-ai-capability-watcher  — new AI unlocks
│   ├── revolutionary-niche-specialist   — Danish underserved markets
│   ├── revolutionary-future-thinker     — 2–5 year shifts
│   └── revolutionary-competitor-scout   — competitive landscape maps
├── analysts/                           ← deepen evidence (no tickets)
│   ├── revolutionary-customer-voice     — buyer profiles (JTBD)
│   └── revolutionary-revenue-modeler    — monetization models with math
└── critics/                            ← stress-test signals (no tickets)
    └── revolutionary-contrarian         — tries to kill every strong signal
```

**Only the captain creates lab tickets.** Everyone else produces evidence as markdown files.

## Daily Pipeline

The team runs **once per day** in this order:

```
[06:30 UTC] All 6 scouts run in parallel        ← independent evidence gathering
[06:50 UTC] Both analysts run in parallel       ← deepen one segment + monetization
[07:00 UTC] Contrarian runs                     ← reads scouts/analysts, attacks signals
[07:10 UTC] Captain runs                        ← synthesizes top 3, creates tickets
[07:15 UTC] Telegram ping to Shpat (optional)   ← "3 new ideas to review"
```

Total run time: ~25 minutes. Total cost: ~$0.50–$2.00 in API calls per day (rough estimate, depends on WebSearch usage).

If a scout fails, the pipeline continues — captain notes the missing input. If contrarian fails, captain notes the synthesis is unstressed. Only captain failure means no tickets that day.

## Output Storage

Two parallel storage paths:

### 1. Markdown archive (history)
```
/home/vegapunk/projects/revolutionaries-ideas/
├── 2026-05-07/
│   ├── pain-hunter.md
│   ├── launch-tracker.md
│   ├── ai-capability-watcher.md
│   ├── niche-specialist.md
│   ├── future-thinker.md
│   ├── competitor-scout.md
│   ├── customer-voice.md
│   ├── revenue-modeler.md
│   ├── contrarian.md
│   └── captain-synthesis.md      ← top 3 ideas + traceability
├── 2026-05-08/
│   └── ...
```

The repo is `shpatjakupi/revolutionaries-ideas` (private). Git auto-commits at end of pipeline.

### 2. Lab tickets (action)

Captain creates tickets at `lab.gomuos.com/api/tickets` with:
- `workspace: "revolutionaries"`
- `severity: "info"`
- `jobName: "revolutionary-captain"`

Shpat reviews tickets in the lab UI. Approving a ticket triggers (later) a "Send to Strawhats" action that copies the pitch into a new strawhats project at status `intake`.

## Triangulation — how ideas form

The captain doesn't pick ideas based on one signal. The strongest ideas are where multiple scouts independently point at the same opportunity:

```
pain-hunter:           "Solo lawyers complain about phone interruptions"
+ ai-capability-watcher: "Voice agents now reliably take phone calls"
+ customer-voice:        "Lawyers already pay for human answering services"
=
IDEA: AI phone receptionist for solo legal practices
```

Captain scores candidates:
- Triangulation: 3+ scouts (3 pts), 2 (2 pts), 1 (1 pt)
- Survived contrarian: +2, wounded: +1, killed: drop
- Has buyer profile: +1
- Has revenue model: +1
- Whitespace exists: +1
- Capability ready: +1

Top 3 by score → tickets.

## Lab API — /revolutionaries

Captain creates tickets via the standard ticket endpoint with `workspace: "revolutionaries"`.

| Endpoint | Use |
|----------|-----|
| `POST /api/tickets` | Create idea ticket. Body: `{ title, description, severity: "info", workspace: "revolutionaries", jobName: "revolutionary-captain" }` |
| `GET /api/tickets?workspace=revolutionaries` | List all idea tickets |
| `GET /api/tickets?workspace=revolutionaries&status=approved` | List approved ideas waiting for Strawhats handoff |

**Note**: assumes Ticket model has `workspace` field (added when Strawhats was built). If your lab schema doesn't have it yet, see "Deployment notes" below.

### Lab UI requirements (separate from agent work)

The lab needs a `/revolutionaries` page (parallel to `/strawhats`):
- List view: all tickets with `workspace="revolutionaries"`, sortable by date and status
- Detail view: full pitch markdown rendered, approve/reject buttons
- "Send to Strawhats" action on approved tickets — creates a new strawhat project from the pitch

This UI is built in the gomuos-lab repo, not in the agent skills.

## Cron Integration (Vegapunk)

The team runs from `vegapunk/src/infra/cron.ts`. All agents are scheduled by the cron module.

```typescript
// In vegapunk cron.ts
const REVOLUTIONARY_SCOUTS = [
  "revolutionary-pain-hunter",
  "revolutionary-launch-tracker",
  "revolutionary-ai-capability-watcher",
  "revolutionary-niche-specialist",
  "revolutionary-future-thinker",
  "revolutionary-competitor-scout",
];

const REVOLUTIONARY_ANALYSTS = [
  "revolutionary-customer-voice",
  "revolutionary-revenue-modeler",
];

// Schedule: daily at 06:30 UTC
async function runRevolutionariesPipeline() {
  // Step 1: scouts in parallel
  await Promise.allSettled(REVOLUTIONARY_SCOUTS.map(spawnAgent));
  // Step 2: analysts in parallel
  await Promise.allSettled(REVOLUTIONARY_ANALYSTS.map(spawnAgent));
  // Step 3: contrarian
  await spawnAgent("revolutionary-contrarian");
  // Step 4: captain
  await spawnAgent("revolutionary-captain");
  // Step 5: optional Telegram notification
  await notifyTelegram("Revolutionaries: 3 new ideas in lab");
}
```

`spawnAgent(name)` spawns Claude Code with the relevant skill, working directory `/home/vegapunk/projects/revolutionaries-ideas`.

Each agent updates job status via `PATCH /api/jobs` with its name (`revolutionary-pain-hunter`, etc.) so the lab dashboard shows last-run time and result.

## Goals File

Active priorities live in `~/.claude/skills/revolutionary-team/GOALS.md`. The captain reads this when synthesizing — if Shpat has set a focus (e.g., "kun B2B Danmark denne uge"), captain weighs ideas accordingly.

## Skills Location

```
dotfiles: .dotfiles/.claude/skills/revolutionary-team/
local:    ~/.claude/skills/  (installed flat by sync-skills, like gomuos-team and strawhat-team)
```

After modifying any agent skill, run sync-skills on the VPS to update the runtime.

## Adding a New Agent

1. Create `.dotfiles/.claude/skills/revolutionary-team/<group>/<agent-name>/SKILL.md`
2. Update this file's "Team Structure" and pipeline steps
3. Update `revolutionary-captain/SKILL.md` Step 1 if captain should read its output
4. Add to `vegapunk/src/infra/cron.ts` pipeline at the right step
5. Commit + push, run sync-skills on VPS

## Modifying an Existing Agent

1. Edit `.dotfiles/.claude/skills/revolutionary-team/<group>/<agent-name>/SKILL.md`
2. If output format changed, also update captain's Step 1/2 (which reads it)
3. Commit + push, run sync-skills on VPS

## Deployment Notes

Before the team can run end-to-end, the following must exist:

### Lab schema
- `Ticket.workspace` field (string, default null) — already used by Strawhats; if not yet on the model, add it via SQL migration:
  ```sql
  ALTER TABLE lab_tickets ADD COLUMN workspace VARCHAR(64) DEFAULT NULL;
  ```
- 10 Job rows for the team agents — created by the cron upsert on first run

### VPS folders
- `/home/vegapunk/projects/revolutionaries-ideas/` — git repo, cloned from `shpatjakupi/revolutionaries-ideas`
- Repo created (empty) on GitHub before first run

### Lab UI
- `/revolutionaries` page parallel to `/strawhats` (lower priority — captain works without UI, but Shpat needs UI to review tickets nicely)

### Telegram notification (optional)
- Add `revolutionariesNotification` step to vegapunk cron

## Why This Team Exists

GomuOS is operating revenue. Strawhats can build new things on demand. But neither generates the next idea. Without a system actively scanning for opportunities, ideas come from random conversations and slow vibes. Revolutionaries makes idea generation a daily, evidence-driven, contrarian-tested process — so Shpat opens his lab dashboard each morning to 3 ideas worth considering, with full traceability back to the evidence that made each idea worth picking.
