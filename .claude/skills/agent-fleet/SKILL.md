---
name: agent-fleet
description: >
  Master index over alle 4 agent-teams (gomuos, strawhats, kidsapp, revolutionaries) der
  kører på Hetzner VPS'en. Kort beskrivelse af hvert hold, hvor deres skills bor, hvordan
  cron-schedulet er sammensat, hvor lab-dashboardet ligger, og hvordan tickets flyder
  mellem holdene. Læs denne skill først når du vil have overblik over hele agent-flåden,
  ikke når du arbejder i et specifikt team.
---

# Agent Fleet — Master Index

Fire uafhængige agent-teams kører på den samme Hetzner VPS (`46.224.215.213`),
orkestreret af én `vegapunk` cron-service og overvåget gennem ét lab-dashboard
(`lab.gomuos.com`). Hver team har sin egen skill med fuld dybde — denne fil er
oversigten over hvordan de hænger sammen.

## De 4 teams

```
┌────────────────┐      ┌────────────────┐      ┌────────────────┐      ┌────────────────┐
│ Revolutionaries│─────▶│   Strawhats    │─────▶│    GomuOS      │      │    KidsApp     │
│  (ideate)      │      │   (build)      │      │   (operate)    │      │   (build+ops)  │
└────────────────┘      └────────────────┘      └────────────────┘      └────────────────┘
   10 agenter              7 agenter              12 agenter              6 agenter
   workspace=4             workspace=2            workspace=1             workspace=3
```

| Team | Formål | Skill | Workspace | URL |
|---|---|---|---|---|
| **gomuos-team** | Vedligeholder eksisterende food-platform (audit + fix) | `gomuos-team/SKILL.md` | `gomuos` (id 1) | ordrupspizza.dk |
| **strawhat-team** | Bygger nye full-stack apps fra menneske-ønsker | `strawhat-team/SKILL.md` | `strawhats` (id 2) | `<slug>.gomuos.com` |
| **kidsapp-team** | Bygger og vedligeholder Unity WebGL-app for børn | `kidsapp-team/SKILL.md` | `kidsapp` (id 3) | kidsapp.gomuos.com |
| **revolutionary-team** | Genererer dagligt revenue-idéer | `revolutionary-team/SKILL.md` | `revolutionaries` (id 4) | n/a (kun tickets) |

Pipelinen mellem holdene: **Revolutionaries** producerer idé-tickets, mennesket
godkender, en idé sendes videre som "wish" til **Strawhats** der bygger en ny app
fra spec til deployed. Når appen er live bliver vedligehold til **GomuOS**' eller
**KidsApp**' domæne, afhængigt af type.

## Cross-cutting infrastruktur

```
        ┌────────────────────────────────────────────────┐
        │  vegapunk (systemd, /home/vegapunk/vegapunk/)  │
        │   src/infra/cron.ts ◀── kilden til timing      │
        │   - registrerer alle jobs i lab                │
        │   - poller tickets hvert 3. minut              │
        │   - spawner Claude-subprocesser per agent      │
        │   - skriver job-status tilbage til lab         │
        └────────────────────────────────────────────────┘
                              │
                              ▼
        ┌────────────────────────────────────────────────┐
        │  gomuos-lab (Next.js på k3s, lab.gomuos.com)   │
        │   - 4 workspaces, én pr. team                  │
        │   - sidebar grupperer pr. team                 │
        │   - per-team agent-side med feedback-form      │
        │   - approval flow: pending → approved → done   │
        └────────────────────────────────────────────────┘
```

## Cron-schedule (én sandhed: `vegapunk/src/infra/cron.ts`)

Job-timing er **ikke** styret af klassiske cron — det er styret af `setInterval` +
`setTimeout` i Node. Schedule-strenge i lab DB er parseable cron der bestemmer
fyringstidspunkt; de fleste jobs aligner til præcis UTC-tid.

| Kategori | Mønster | Eksempler |
|---|---|---|
| Hot jobs | `setInterval` fra opstart med per-job startup-offset (`HOT_JOB_STARTUP_OFFSET_MS`) | health-monitor (15m), ticket-dispatcher (3m), 3 managers |
| Daglige jobs | Startup-stagger + alignment til næste UTC-tid + 24h interval | spredt over hele døgnet, mindst 2h mellem hver |
| Ugentlige jobs | Samme som daglige, alignment til ugedag+tid + 7d interval | gomuos-menu-specialist |
| Sekventielle pipelines | Aligner til fast tid, ingen startup-run | revolutionaries-pipeline (~50min, kører 10 agenter sekventielt med 30s sleep) |

**Designprincipper når du tilføjer/justerer schedules:**
- Daglige jobs spredes over hele 24h-cyklen så vi undgår parallelle Opus-subprocesser
  og spreder API-load. Mindst 2h mellem to fixed-time jobs.
- To hot jobs med samme `intervalMs` MÅ ikke fyre på samme offset — tilføj en entry
  i `HOT_JOB_STARTUP_OFFSET_MS` (fx `kidsapp-manager: +1h` så 2h-cyklus rammer T+1h
  i stedet for samtidig med `gomuos-manager`).
- Pipelines der spawner flere agenter skal have inter-agent sleep (revolutionaries
  bruger 30s) for at undgå Anthropic rate-limit per-minute window.

For nøjagtige tidspunkter, læs `cron.ts` direkte — alt er deklareret i `jobs`-arrayet
nederst i `createCronScheduler`.

## Agent-controls (sleep/wake/run-now/model/cost-cap/notify)

Hver agent har et sæt human-styrede kontroller der lever i lab DB og pulles
ind i vegapunk via en hot-reload-loop hvert 60. sekund. Det betyder:
**ingen kode-deploy nødvendig** for at pause en agent, skifte dens model, sætte
en dagskost-cap, eller mute dens fejl-alerts.

Hvor styres det:
- **Per-agent**: klik en række på `/<team>/agents` → udvid kontrol-panelet
  (sleep/wake, run-now, model, effort, cost-cap, notify, last-runs)
- **Per-team**: bjælken øverst på `/<team>/agents` (pause hele workspace'et)
- **Globalt**: `/settings` — vacation mode (pauser ALT) + concurrency limit
  (max parallelle Claude-subprocesser) + "Genstart vegapunk"-knap

Hvad gates'er hvad i vegapunk-cron'en:
1. Vacation mode → skip alle jobs
2. Workspace.pausedUntil → skip jobs i det workspace
3. Job.pausedUntil → skip én agent
4. Job.enabled=false → skip én agent
5. dailyCostCapUsd ≤ today's SUM(JobRun.costUsd) → skip + alert
6. Concurrency-semafor (default 2) køer Claude-kald

Run-now bypasser sleep + workspace-pause men IKKE vacation/cost-cap.
Skipped runs landes som `JobRun{status:"skipped"}` så de optræder i UI'ens
last-runs-strip med et ⊘-ikon.

Cost-cap-data: hentes fra `/api/jobs/cost-summary` (SQL sum af `JobRun.costUsd`
for indeværende UTC-dag). Source of truth — overlever vegapunk-restarts.

Schedule-ændringer: `Job.schedule` opdateres straks i DB, men
`setInterval`-handles i vegapunk genskabes ikke ved hot-reload. Bruger klikker
"Genstart vegapunk" på `/settings` → flag sættes → vegapunk's hot-reload tjekker
flaget mod `BOOTED_AT` og kalder `systemctl restart vegapunk` hvis flaget er
nyere. Loop-safe: den genstartede proces ser samme flag men har et nyere
BOOTED_AT.

Detaljer på schema/API: `gomuos-lab` skill. Detaljer på cron-implementering:
`vegapunk-assistant` skill.

## Dispatcher-failure-modus

Når en agent fejler (rate-limit, exception, timeout, dirty repo, unpushed code)
reverter dispatcheren ticket'en til **`pending`** — ikke til `approved`. Det er med
vilje: approved tickets bliver picked up igen 3 min senere, hvilket vil hammre en
rate-limited API kontinuerligt. Med pending falder den ud af queuen, brugeren får
én Telegram-alert, og kan klikke "approve" igen i lab UI'et når årsagen er løst.

Dette gælder alle 5 revert-sites i `cron.ts`: pre-flight clean-check, agent
execution failure, post-flight push-check, og begge throw-catches i dispatcher-loopet.

## Lab API & workspaces

- Base URL: `https://lab.gomuos.com/api`
- Auth: `Authorization: Bearer $LAB_API_KEY`
- Hvert ticket og job har `workspaceId`. Når en agent opretter tickets skal den
  inkludere `workspace: "<slug>"` (default er `gomuos` hvis udeladt).
- Lab-skemaet: se `gomuos-lab` skill for fuld Prisma schema, status-flows og UI.

UI-sider per team:
- GomuOS: `/tickets`, `/hunters` (agenter), `/jobs`
- Straw Hats: `/strawhats`, `/strawhats/agents`
- KidsApp: `/kidsapp`, `/kidsapp/agents`
- Revolutionaries: `/revolutionaries`, `/revolutionaries/agents`

Agent-feedback fra alle teams går igennem `/api/agents/feedback` der opretter en
ticket assignedAgent til den pågældende teams manager (gomuos-manager,
strawhat-manager, kidsapp-manager eller revolutionary-captain).

## Hvor bor skill-filerne?

```
.dotfiles/.claude/skills/
├── agent-fleet/             ← denne fil (oversigt)
├── gomuos-team/
│   ├── SKILL.md              ← team-oversigt
│   ├── manager/              ← gomuos-manager
│   ├── hunters/              ← 6 hunters
│   ├── devs/                 ← frontend + backend
│   ├── validators/           ← code-reviewer + playwright
│   └── infra/
├── strawhat-team/
│   ├── SKILL.md
│   ├── captain/              ← strawhat-manager
│   ├── intake/, design/, devs/, ops/, qa/
├── kidsapp-team/
│   ├── SKILL.md
│   ├── manager/              ← kidsapp-manager
│   ├── scouts/, devs/, validators/, infra/
├── revolutionary-team/
│   ├── SKILL.md
│   ├── captain/              ← revolutionary-captain (også team-manager)
│   ├── scouts/, analysts/, critics/
├── gomuos-lab/SKILL.md       ← lab-dashboard arkitektur
├── vps-cluster/SKILL.md      ← k3s, Traefik, cert-manager
└── vegapunk-assistant/SKILL.md ← Telegram-bot der orkestrerer
```

Sync-skills installerer dem til VPS i to former:
- Flat: `~/.claude/skills/<agentName>/SKILL.md` (bruges af cron-spawnerne)
- Nested: `~/.claude/skills/<team>-team/<group>/<agentName>/SKILL.md` (bruges af nogle agenter ved ref)

## Tilføj et nyt team

1. Insert workspace i lab DB: `INSERT INTO lab_workspaces (slug, name) VALUES ('<slug>', '<name>')`
2. Tilføj `<slug>` til `WORKSPACE_SLUGS` i `gomuos-lab/src/lib/workspace.ts`
3. Tilføj sidebar-entries i `gomuos-lab/src/components/Sidebar.tsx`
4. Opret team-skill mappen `<team>-team/` i dotfiles
5. Tilføj jobs til `vegapunk/src/infra/cron.ts` — sørg for at `workspaceForJob()` mapper prefix → slug
6. Hvis teamet skal have feedback-form til agenter: tilføj `manager` + `teamSlug` til `TEAM_CONFIG` i `gomuos-lab/src/app/api/agents/feedback/route.ts`
7. Opret `/<slug>/agents/page.tsx` der bruger `<TeamAgents workspace="<slug>" />`

## Tilføj en ny agent til et eksisterende team

1. Opret `dotfiles/.claude/skills/<team>-team/<group>/<agent-name>/SKILL.md`
2. Hvis agenten kører på cron: tilføj job i `cron.ts`. `workspaceForJob()` autoroutes
   så længe agent-navnet bruger team-prefixet (`gomuos-`, `strawhat-`, `kidsapp-`,
   `revolutionary-`)
3. Hvis agenten skal route'es af manager til specifikke ticket-typer: opdater
   `<team>-manager`'s skill med routing-regler
4. Commit + push, sync-skills på VPS, restart vegapunk
5. Agenten dukker automatisk op på `/<team>/agents` (kort efter første job-status update)

## Når noget er galt

| Symptom | Først kig her |
|---|---|
| Agent kører ikke | `journalctl -u vegapunk -f` på VPS |
| Agent fyrer på forkert tidspunkt | `cron.ts` — er schedule-strengen parseable? |
| Agent dukker op på forkert team-side | DB: `SELECT name, workspaceId FROM lab_jobs` |
| Tickets fra agent havner i forkert workspace | Agentens skill — sender den `workspace: "<slug>"` med? |
| Lab-side viser intet | Tjek `getWorkspaceIdBySlug` cache + at workspace-rækken eksisterer |
| Ticket sat til pending efter agent-run | Dispatcheren reverter til pending ved fejl (rate-limit, dirty repo, unpushed code). Tjek Telegram for alert + `journalctl` for agent-output. Re-approve manuelt når årsagen er løst. |
| Vegapunk har gammel kode efter push | Tjek GitHub Actions run for `deploy.yml`. Hvis fejlet, kommer der Telegram-alert med link til run. |
