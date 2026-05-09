---
name: revolutionary-team
description: Complete overview of the Revolutionaries agent team — daily idea-generation crew that scans the world for revenue opportunities. 6 scouts gather evidence, 2 analysts deepen it, 1 contrarian stress-tests it, 1 captain synthesizes the top 3 ideas of the day into lab tickets. Fully independent team — ideas stand on their own, no handoff to other teams. Read this first when working with or modifying the Revolutionaries system.
---

# Revolutionary Agent Team

A team of specialized Claude Code agents that scan the internet daily for revenue opportunities. Six scouts gather independent evidence, two analysts deepen it, one contrarian stress-tests it, and one captain synthesizes the top 3 ideas of the day into lab tickets at `lab.gomuos.com`.

The Revolutionaries team is **fully independent**. It shares the lab dashboard
and the vegapunk cron host with the other teams (GomuOS, Strawhats, KidsApp),
but its output is self-contained: ideas land as lab tickets in the
`revolutionaries` workspace and stop there. There is no automatic handoff to
any other team — Shpat reads the tickets, and whatever happens next is a
separate decision made outside this system.

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

The team runs **once per day** at `04:30 UTC` (cron string `30 4 * * *`). All 10
agents run **sequentially** in one process, with 30s sleep between each to stay
under Anthropic's per-minute rate window. Total wall-clock time: ~50 minutes.

```
[04:30 UTC] Telegram: "Revolutionaries pipeline starter"
            ── scouts (Sonnet) ──
            revolutionary-pain-hunter            (≤12m, 30s sleep)
            revolutionary-launch-tracker         (≤12m, 30s sleep)
            revolutionary-ai-capability-watcher  (≤12m, 30s sleep)
            revolutionary-niche-specialist       (≤12m, 30s sleep)
            revolutionary-future-thinker         (≤12m, 30s sleep)
            revolutionary-competitor-scout       (≤12m, 30s sleep)
            ── analysts (Sonnet) ──
            revolutionary-customer-voice         (≤12m, 30s sleep)
            revolutionary-revenue-modeler        (≤12m, 30s sleep)
            ── critic (Opus) ──
            revolutionary-contrarian             (≤12m, 30s sleep)
            ── captain (Opus) ──
            revolutionary-captain                (≤12m — creates tickets)
            ── git commit + push to revolutionaries-ideas ──
[~05:20 UTC] Telegram: "Revolutionaries færdig — se nye idéer på lab"
```

Per-agent timeout is 12 minutes; if an agent throws, the loop catches the error
(`runRevolutionaryAgent.catch`) and continues to the next one. Captain failure
just means no new tickets that day. Models: scouts + analysts on Sonnet
(`REV_SONNET`), contrarian + captain on Opus (`REV_OPUS`) — synthesis and
adversarial reasoning carry the harder load.

Note: scouts/analysts cannot literally run in parallel here because each agent
is its own Claude subprocess and we'd hammer the API rate limit. Sequential +
30s gap is the proven-safe pattern; first-day evidence showed agents 7–10
fail-fast with `code 1 in 2-3s` without the delay.

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

Shpat reviews tickets in the lab UI and approves or rejects them. Approval is
the end of this team's job — there is no automatic downstream action.

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
| `GET /api/tickets?workspace=revolutionaries&status=approved` | List approved ideas |

The captain passes `workspace: "revolutionaries"` as a string in the POST body
— the lab API resolves it to `workspaceId` server-side via
`getWorkspaceIdBySlug`.

### Lab UI

`/revolutionaries` (list) and `/revolutionaries/agents` (per-agent feedback
form) both exist in the gomuos-lab repo. List view filters by
`workspace="revolutionaries"` with status tabs. Approve/reject are the only
ticket actions — no downstream handoff to other workspaces.

## Cron Integration (Vegapunk)

The team runs from `vegapunk/src/infra/cron.ts` as a **single pipeline job**
called `revolutionaries-pipeline`. There are no per-agent cron entries — the
pipeline job loops through `REVOLUTIONARIES_AGENTS` in order.

Key facts (source of truth: `cron.ts`, lines ~440–545 and ~680–720):

- **Schedule**: `30 4 * * *` (04:30 UTC, daily, 24h interval)
- **Pipeline name** in lab DB: `revolutionaries-pipeline` (workspace: `revolutionaries`)
- **Sub-agents** (`REVOLUTIONARIES_AGENTS`) are pre-registered as 10 lab jobs so
  per-agent `lastStatus` updates land in the right workspace
- **Per-agent timeout**: `REV_AGENT_TIMEOUT_MS = 12 * 60 * 1000` (12 min)
- **Inter-agent sleep**: `REV_INTER_AGENT_SLEEP_MS = 30 * 1000` (30s)
- **Working directory**: `/home/vegapunk/projects/revolutionaries-ideas/`
- **Model selection**: `REV_SONNET` for scouts + analysts, `REV_OPUS` for
  contrarian + captain (declared at the top of the agents array)
- **Startup behavior**: pipeline is in `SKIP_STARTUP_RUN` — it does **not**
  fire on service restart, only at the next 04:30 UTC alignment
- **Workspace routing**: `workspaceForJob()` maps the prefix `revolutionary-`
  (and the literal `revolutionaries-pipeline` name) to workspace
  `revolutionaries`

Each agent runs via `runRevolutionaryAgent(name, model)`, which:
1. Sets job status `running` in lab
2. Spawns Claude with prompt pointing to its skill at
   `~/.claude/skills/revolutionary-team/<name>/SKILL.md` and the team's
   `GOALS.md`
3. Writes `success` (with first 2000 chars of output) or `error` back to the lab

After the loop, the pipeline:
1. `git add <today>/`, commits as `vegapunk` user, pushes to
   `shpatjakupi/revolutionaries-ideas`
2. Sends a Telegram message with a link to `lab.gomuos.com/revolutionaries`

Telegram fires at **both** start (`pipeline starter`) and end
(`færdig — se nye idéer`). If the whole pipeline throws, a `⚠️ pipeline
fejlede` alert with the error message is sent instead.

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
2. Update this file's "Team Structure" and pipeline diagram
3. Update `revolutionary-captain/SKILL.md` Step 1 if captain should read its output
4. Add an entry to the `REVOLUTIONARIES_AGENTS` array in `vegapunk/src/infra/cron.ts`
   at the correct sequential position (scouts before analysts before contrarian
   before captain). Pick `REV_SONNET` or `REV_OPUS` based on reasoning load.
5. Adding an agent extends total pipeline time by ~30s + agent runtime — confirm
   we still finish well inside any downstream window
6. Commit + push, run sync-skills on VPS, restart vegapunk

## Modifying an Existing Agent

1. Edit `.dotfiles/.claude/skills/revolutionary-team/<group>/<agent-name>/SKILL.md`
2. If output format changed, also update captain's Step 1/2 (which reads it)
3. Commit + push, run sync-skills on VPS

## Deployment Notes

Before the team can run end-to-end, the following must exist:

### Lab schema
- Workspace `revolutionaries` (id 4) must exist in `lab_workspaces`. Tickets
  and jobs use `workspaceId` (FK), not a string field — the workspace was added
  alongside the strawhats migration. See `gomuos-lab` skill for the schema.
- 11 Job rows total: `revolutionaries-pipeline` (the cron entry) + 10
  sub-agent jobs in `REVOLUTIONARIES_AGENTS`. All upserted by cron on startup
  via `lab.registerJob(...)`.

### VPS folders
- `/home/vegapunk/projects/revolutionaries-ideas/` — git repo, cloned from `shpatjakupi/revolutionaries-ideas`
- Repo created (empty) on GitHub before first run

### Lab UI
- `/revolutionaries` page parallel to `/strawhats` (lower priority — captain works without UI, but Shpat needs UI to review tickets nicely)

### Telegram notifications
- Already wired: `pipeline starter` at start, `færdig` at end, `⚠️ fejlede`
  on pipeline-level throw. No extra step needed.

## Why This Team Exists

Without a system actively scanning for opportunities, ideas come from random
conversations and slow vibes. Revolutionaries makes idea generation a daily,
evidence-driven, contrarian-tested process — so Shpat opens his lab dashboard
each morning to 3 standalone ideas worth considering, each with full
traceability back to the evidence that made it worth picking.
