---
name: agent-fleet
description: >
  Master-skill for hele agent-flåden — de 3 agent-teams (gomuos, kidsapp = Bumble og vegalab) der
  kører på Hetzner VPS'en. Kort beskrivelse af hvert hold, hvor deres
  skills bor, hvordan cron-schedulet er sammensat, hvor lab-dashboardet ligger, hvordan
  tickets flyder mellem holdene, og VS Code-workspacet der samler det hele. Brug den når
  du starter en session med hele flåden ("boot", "load systemkontekst", "agent-flåden"),
  eller vil have overblik på tværs af teams — ikke når du arbejder i et specifikt team.
---

# Agent Fleet — Master Index

Tre agent-teams kører på den samme Hetzner VPS (`46.224.215.213`),
orkestreret af én `vegapunk` cron-service og overvåget gennem ét lab-dashboard
(`lab.gomuos.com`). Hver team har sin egen skill med fuld dybde — denne fil er
oversigten over hvordan de hænger sammen.

## Session-start (boot)

Når du starter en session der arbejder med hele flåden:

1. Åbn VS Code-workspacet for brugeren (se afsnittet nedenfor):
   `code "C:\Users\shpat\Desktop\projects\agent-fleet.code-workspace"`
   Spring over hvis brugeren siger nej, eller hvis sessionen kører på VPS'en (ingen VS Code der).
2. Læs denne fil færdig.
3. Skal du røre vegapunk (cron, dispatcher, deploy): læs også `vegapunk-assistant` skill.
4. Skal du røre lab'en (tickets, workspaces, agent-controls, schema): læs også `gomuos-lab` skill.
5. Bekræft kort hvad du har forstået, og vent på instruktioner.

**Status at kende (se memory for detaljer):** gomuos og kidsapp er pauset permanent
siden 2026-05-15 — cron kører, men skipper deres jobs. Vegalab (fra 2026-11-10) er aktivt:
ticket-dispatcheren er undtaget fra workspace-pause og springer kun tickets i pausede workspaces over.

**Straw Hats og Revolutionaries er lagt ned og slettet (2026-11-10)** — skills, cron-jobs,
Lab-sider og DB-data er væk (backup i `/root/backups/` på VPS'en). Foreslå dem ikke igen.

## VS Code-workspace

`C:\Users\shpat\Desktop\projects\agent-fleet.code-workspace` samler hele flåden i ét
vindue: de 3 team-skill-mapper, lab-skills, denne skill, `vegapunk/`, `gomuos-lab/`
og hele `.dotfiles`. Åbn med `code <sti>` eller dobbeltklik. Tilføjer du et nyt team,
så tilføj også dets skill-mappe til `folders` i workspace-filen.

Samme mapper vises på **`lab.gomuos.com/fleet`** (Agent-flåden) med links og redigerbare
skills, så de kan tilgås uden VS Code. Holdes i sync med workspace-filen
(`ENTRIES` i `gomuos-lab/src/app/fleet/page.tsx`).

## De 3 teams

| Team | Formål | Skill | Workspace | Lab |
|---|---|---|---|---|
| **gomuos-team** | Vedligeholder food-platformen (audit + fix) | `gomuos-team/SKILL.md` | `gomuos` (id 1) | `/tickets`, `/hunters`, `/jobs` |
| **kidsapp-team** | Bumble App: børneapp med brumbassen (web + Capacitor, 2D via OpenAI, 3D i Blender). Unity droppet 2026-10-22 | `kidsapp-team/SKILL.md` | `kidsapp` (id 3), vises som "Bumble" | `/kidsapp`, `/kidsapp/agents` |
| **vegalab-team** | Udfører ændringer i Lab'en og Vegapunk, bestilt med ⚡ Udfør i Lab-chatten og godkendt én gang. Agenter: `lab-developer`, `vegapunk-developer`. Dispatcheren deployer | `vegalab-team/SKILL.md` | `vegalab` | `/vegalab`, `/chat` |

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
        │   - 2 aktive workspaces (gomuos, kidsapp)      │
        │   - sidebar grupperer pr. team                 │
        │   - per-team agent-side med feedback-form      │
        │   - approval flow: pending → approved → done   │
        │   - /fleet, /chat (Claude), 2FA-login          │
        └────────────────────────────────────────────────┘
```

## Cron-schedule (én sandhed: `vegapunk/src/infra/cron.ts`)

Job-timing er **ikke** styret af klassiske cron — det er styret af `setInterval` +
`setTimeout` i Node. Schedule-strenge i lab DB er parseable cron der bestemmer
fyringstidspunkt; de fleste jobs aligner til præcis UTC-tid.

| Kategori | Mønster | Eksempler |
|---|---|---|
| Hot jobs | `setInterval` fra opstart med per-job startup-offset (`HOT_JOB_STARTUP_OFFSET_MS`) | health-monitor (15m), ticket-dispatcher (3m), managers |
| Daglige jobs | Startup-stagger + alignment til næste UTC-tid + 24h interval | spredt over hele døgnet, mindst 2h mellem hver |
| Ugentlige jobs | Samme som daglige, alignment til ugedag+tid + 7d interval | gomuos-menu-specialist |

**Designprincipper når du tilføjer/justerer schedules:**
- Daglige jobs spredes over hele 24h-cyklen så vi undgår parallelle Opus-subprocesser
  og spreder API-load. Mindst 2h mellem to fixed-time jobs.
- To hot jobs med samme `intervalMs` MÅ ikke fyre på samme offset — tilføj en entry
  i `HOT_JOB_STARTUP_OFFSET_MS` (fx `kidsapp-manager: +1h` så 2h-cyklus rammer T+1h
  i stedet for samtidig med `gomuos-manager`).
- Pipelines der spawner flere agenter skal have inter-agent sleep (fx 30s) for at
  undgå Anthropic rate-limit per-minute window.

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
- Auth: `Authorization: Bearer $LAB_API_KEY` — **påkrævet på alle `/api/*`** (middleware,
  2026-11-10). Uden nøgle eller browser-session svarer API'et 401.
- Hvert ticket og job har `workspaceId`. Når en agent opretter tickets skal den
  inkludere `workspace: "<slug>"` (default er `gomuos` hvis udeladt).
- Lab-skemaet: se `gomuos-lab` skill for fuld Prisma schema, status-flows og UI.

UI-sider:
- GomuOS: `/tickets`, `/hunters` (agenter), `/jobs`
- Bumble: `/kidsapp`, `/kidsapp/agents`
- Globalt: `/fleet` (agent-flåden), `/chat` (spar med Claude om skills/tickets/jobs), `/settings`

Login kræver kodeord + TOTP-kode (2FA). Billeder kan vedhæftes tickets (`POST /api/attachments`).

Agent-feedback fra alle teams går igennem `/api/agents/feedback` der opretter en
ticket assignedAgent til den pågældende teams manager (gomuos-manager
eller kidsapp-manager).

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
├── kidsapp-team/
│   ├── SKILL.md
│   └── scouts/               ← 3 idé-scouts (pauset)
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
   så længe agent-navnet bruger team-prefixet (`gomuos-` eller `kidsapp-`)
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
