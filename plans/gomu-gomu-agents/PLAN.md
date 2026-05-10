# gomu-gomu-agents — PLAN

One Piece-tematiseret pixel-art verden der visualiserer agent-flåden i `lab.gomuos.com/world`.

**Status**: Planning phase. Ingen kode skrevet endnu.
**Ejer**: shpatjakupi
**Startet**: 2026-05-10
**Sidst opdateret**: 2026-05-10

---

## 1. Mål

Erstatte/supplere det klassiske ticket/job-dashboard på `lab.gomuos.com` med en levende isometrisk pixel-art verden hvor:

- Hvert af de 4 agent-teams er sit eget område (skib eller ø) i en One Piece-inspireret Grand Line
- Hver agent er en One Piece-karakter der bevæger sig, animerer, snakker og fejrer
- Real-time data fra lab DB driver alt: agent-status, ticket-flow, comments
- Mennesket kan klikke på en agent og få sidepanel med detaljer
- Approve/reject sker stadig på normale lab-sider — verdenen er primært **observation**

Inspiration: [JamsusMaximus/codemap](https://github.com/jamsusmaximus/codemap) (Habbo-stil agent-visualisering) + androoagi's space-station dashboard (#openclaw).

---

## 2. Arkitektur

```
┌─────────────────────────────────────────────────────────┐
│              gomu-gomu-agents (PRIVAT npm-pakke)        │
│                                                          │
│  Generisk isometrisk world-engine. INGEN One Piece IP.  │
│                                                          │
│  - PixiJS-rendering (sprites, animation, zoom)          │
│  - React component <GomuGomuWorld config={...} />       │
│  - WebSocket/SSE klient til real-time events            │
│  - Asset-loader (sprites injiceres af forbruger)        │
│  - Toggle alle/aktive, mobil-responsive                 │
│                                                          │
└─────────────────────────────────────────────────────────┘
                          ▲
                          │ npm dependency
                          │
┌─────────────────────────────────────────────────────────┐
│              gomuos-lab (PRIVAT, eksisterende)           │
│                                                          │
│  - /world side bruger <GomuGomuWorld>                   │
│  - public/onepiece/ One Piece sprites (free + tegnede) │
│  - /api/world/state — agent-positions, statuses         │
│  - /api/world/events — SSE stream af tickets/comments   │
│  - Konfig: team→ø/skib mapping, agent→karakter mapping  │
│                                                          │
└─────────────────────────────────────────────────────────┘
                          ▲
                          │ HTTP/WS
                          │
┌─────────────────────────────────────────────────────────┐
│              Lab DB + Vegapunk cron (eksisterende)       │
│                                                          │
│  - Workspace, Job, Ticket, Comment, Execution           │
│  - Vegapunk skriver job-status til lab DB               │
│  - Lab pusher events videre til /api/world/events       │
└─────────────────────────────────────────────────────────┘
```

**Hvorfor split mellem npm-pakke og lab?**
- Pakken er IP-ren → kan publiceres bredere senere uden licens-tvang
- One Piece-assets bor kun i privat lab-repo → minimal juridisk risk
- Engine kan genbruges (vegapunk-bot side, demo, andre projekter)

---

## 3. Endelig spec (samlet fra dialog 2026-05-10)

| Kategori | Beslutning |
|---|---|
| **Verden** | Hybrid Grand Line: havkort med både øer og skibe afhængig af team |
| **Perspektiv** | Isometrisk (Habbo-stil) |
| **Map-navigation** | Zoom-able: oversigt → klik område = zoom ind |
| **Map-størrelse** | Verdens-overblik passer på desktop-skærm; zoom-in scenes er individuelt størrelse |
| **Hav-atmosfære** | Tomt hav, blød bølge-animation |
| **Sunny** | Fast position |
| **Placering** | `/world` rute i lab (ny side i sidebar) |
| **Real-time** | WebSocket eller SSE fra lab |
| **Mobil** | Responsive (pinch-zoom, tap-interactions) |
| **Tech** | PixiJS + React + TypeScript + Vite (npm-pakke build) |
| **Repo-stil** | Standalone npm-pakke importeret i lab |
| **Repo-synlighed** | Privat (begge) |
| **Pakke-navn** | `gomu-gomu-agents` |

### Karakter-system

| | |
|---|---|
| Visuel mapping | Manuelt — du tildeler karakter pr. agent |
| Mapping-strategi | Pr. team løbende, ikke alle 35 på forhånd |
| Labels | Synlige kun ved hover |
| Ranks | Manager=kaptajn (større/anden hat), specialist=officer, dev=crew-medlem |
| Antal synlige | Toggle: alle agents / kun aktive |
| Animation | Rig — specifikke pr. karakter (Gear 5 vs Kaido loop, Sunny rocking, Sanji koger, Zoro hugger osv) |
| Cross-team rejser | Ja — agent sejler/går over til andet teams område når tildelt cross-team ticket |

### Interaktion

| | |
|---|---|
| Klik agent | Sidepanel slider ind fra højre |
| Sidepanel indhold | Karakter-navn, agent-navn, role, current ticket, recent activity |
| Action fra verden | Kun visning — approve/reject sker på normale lab-sider |
| Comments-visning | Både fly-by tale-bobler over hoveder OG bund-bar Ship Comms-chat |
| Activity feed | Bund-bar (højde-collapsible) |

### Events & feedback

| | |
|---|---|
| Completion celebration | Fuld show: skattekiste/fyrværkeri, varighed ~5 sek |
| Failure | Stormsky/Sea King effekt |
| Lyd ved celebration | Ingen ekstra lyd ved celebration (konflikt med musik undgået) |
| Baggrundsmusik | Bink's Sake (instrumentel) — toggle on/off |
| SFX-niveau | Mellem: footsteps, click, ding ved comment, pop ved complete |

### Team-mapping (One Piece)

| Team | Sted/Setting | Type | Detaljer |
|---|---|---|---|
| **strawhats** | Thousand Sunny | Skib | Luffy's crew. Fast position på kortet. |
| **gomuos** | Egghead-øen | Ø | Vegapunk's lab/by. |
| **kidsapp** | Wano | Ø m. animeret kamp-loop | Luffy Gear 5 vs Kaido i evig kamp i baggrunden mens kids-agenter arbejder. |
| **revolutionaries** | Baltigo (skjult base) | Ø | Dragons revolutionsarmé. |

### Agent-karakter mapping (laves løbende — TODO-skema)

```
Strawhats team (7 agenter):
  strawhat-manager           → Luffy (kaptajn)
  strawhat-product-intake    → ?
  strawhat-architect         → ?
  strawhat-devops            → ?
  strawhat-frontend-developer→ ?
  strawhat-backend-developer → ?
  strawhat-qa-tester         → ?

Gomuos team (12 agenter):
  gomuos-manager             → ? (Vegapunk hoved-figur?)
  gomuos-frontend-developer  → ?
  gomuos-backend-developer   → ?
  gomuos-code-reviewer       → ?
  gomuos-playwright-tester   → ?
  gomuos-admin-specialist    → ?
  gomuos-checkout-specialist → ?
  gomuos-menu-specialist     → ?
  gomuos-orders-specialist   → ?
  gomuos-wolt-specialist     → ?
  gomuos-ui-reviewer         → ?
  (… verificer i lab DB)

Kidsapp team (6 agenter):
  kidsapp-manager            → ?
  (… udfyldes)

Revolutionary team (10 agenter):
  revolutionary-captain      → Dragon
  (… udfyldes)
```

**Format pr. mapping**: `<agent-navn> → <one-piece-karakter> [+ kort visuel beskrivelse]`

---

## 4. Asset-strategi

Baseret på research 2026-05-10 (se [research-rapport](#asset-research-summary) i Appendix A).

**OBS 2026-05-10**: Hybrid mix af Kenney pirate ship-sprite + kode-tegnede ellipse-øer blev forsøgt og afvist — inkonsistent. S3.5 skal vælge ÉN sammenhængende stil (sandsynligvis Kenney tilemap mosaic) og bygge proper tile-baserede øer.

**Hybrid model:**
1. **Free itch.io sprites** for hovedfigurer hvor de findes:
   - Luffy 32x32 (SpriteLunchbox)
   - Shanks (DAZ)
   - Dragon (DAZ)
   - Attribution i lab-repo `assets/onepiece/CREDITS.md`
2. **Code-drawn (CodeMap-stil)** for resten: Vegapunk, Kaido (Gear 5-modstander), Sanji, Zoro, Nami, Robin, Chopper, Brook, Franky, Jinbei, Sabo, Ivankov osv.
3. **Tilesets**: gratis isometriske ocean/island/skibs-tiles fra OpenGameArt eller Kenney.nl (CC0)
4. **Baggrunde** (Wano kamp-scene, Egghead-bygninger): code-drawn

**Filplacering**:
```
gomuos-lab/public/onepiece/
├── characters/
│   ├── luffy.png        (32x32 sprite-sheet, fra itch.io)
│   ├── shanks.png       (fra itch.io)
│   ├── dragon.png       (fra itch.io)
│   ├── vegapunk.png     (code-drawn / vores egen)
│   ├── kaido.png        (code-drawn)
│   ├── ...
├── tiles/
│   ├── ocean.png
│   ├── sand.png
│   ├── wood-deck.png
├── scenes/
│   ├── sunny-deck.png
│   ├── egghead-lab.png
│   ├── wano-bg.png
│   └── baltigo-base.png
└── CREDITS.md          (attribution til alle eksterne kilder)
```

---

## 5. Faser & sessions

| Session | Fokus | Output | Estimat |
|---|---|---|---|
| **S1 (i dag)** | Research + PLAN.md | Denne fil + research-rapport | ✓ done |
| **S2** | Bootstrap repo | Privat GitHub-repo `gomu-gomu-agents`. Vite+React+PixiJS+TS scaffold. Tom `<GomuGomuWorld />` der eksporteres. CI til GitHub Packages. | 1-2t |
| **S3** | Statisk verden (kode-tegnet) | Top-down 2D overview-map med 4 zoner, ocean, zoom + pan. Kode-tegnede placeholders. | 4-6t ✓ |
| **S3.5** | **Statisk visuel design-pass** | Ét sammenhængende sprite-system (sandsynligvis Kenney mosaic). Proper tilemap-øer (ikke ellipse+1 sprite). Source One Piece captain-sprites. INGEN animering. Slut når 4 zoner + 4 captains ser pæne ud i screenshot. | 4-8t |
| **S4** | Real-time data | Nye lab API endpoints `/api/world/state` + SSE `/api/world/events`. Verden læser data og placerer karakterer korrekt. Connect/reconnect logik. | 4-6t |
| **S5** | Animationer | Idle/walk/work/sit animationer på allerede-pæne sprites. Cross-team rejse-animation. Gear 5 vs Kaido kamp-loop. | 6-10t |
| **S6** | Comments + celebration + lyd + mobil | Fly-by tale-bobler, Ship Comms bund-bar, treasure chest celebration, Bink's Sake musik, mellem SFX, mobil pinch-zoom, deploy. | 4-6t |

Total estimat: **20-30 timer** kode + research.

Hver session afsluttes med:
1. Commit + push til pakkens repo og lab
2. Update på `SESSION_STATE.md` (se næste sektion)
3. Generer handover-prompt hvis sessionen ikke afsluttede en hel fase

---

## 6. Handover-strategi

For at undgå tab af kontekst mellem sessioner (rate limit, ny dag, ny chat):

### `SESSION_STATE.md` (i `gomu-gomu-agents/` repo-roden)

Opdateres ved slutning af hver session. Format:

```markdown
# Session State — gomu-gomu-agents

**Sidste session**: 2026-05-10 (S2 — bootstrap)
**Næste session**: S3 — statisk verden
**Aktiv branch**: main (eller feature/<navn>)

## Done
- [x] Privat repo oprettet på GitHub
- [x] Vite + React + PixiJS + TS scaffold
- [x] Tom <GomuGomuWorld /> eksporteret
- [x] GitHub Actions: build + publish til GitHub Packages

## In progress
- [ ] (intet)

## Next session entry-point
1. Klon repo: `git clone git@github.com:shpatjakupi/gomu-gomu-agents.git`
2. `npm install && npm run dev`
3. Begynd med PixiJS isometric scene-setup i `src/components/IsometricWorld.tsx`
4. Følg Fase S3 fra PLAN.md

## Open decisions
- Hvilken karakter til strawhat-architect? (TODO under session)
- Skal kamp-loop (Wano) være 4 eller 8 frames? (test under session)

## Gotchas / lessons learned
- (tom)
```

### Handover-prompt (når session afsluttes midt-fase)

Hvis vi rammer rate limit eller stopper midt i kompleks state, leverer jeg en prompt på denne form:

```
Du fortsætter på gomu-gomu-agents — One Piece-tema agent-dashboard for lab.gomuos.com.

Læs disse filer FØRST i rækkefølge:
1. ~/Desktop/projects/.dotfiles/plans/gomu-gomu-agents/PLAN.md (overblik)
2. <repo>/SESSION_STATE.md (hvor vi stoppede)

Vi er midt i Fase S<N>. Konkret stoppede vi her:
<2-3 sætninger om hvad der lige nu er på vej>

Næste konkrete skridt:
- <step 1>
- <step 2>

Forsæt arbejdet uden at stille bekræftelses-spørgsmål — alle store beslutninger er taget i PLAN.md.
```

---

## 7. Lab integration-punkter

### Nye API endpoints i lab

```
GET  /api/world/state
  Returns: { workspaces: [...], agents: [...], tickets: [...], jobs: [...] }
  Auth:    none (read-only, samme som /api/tickets)

GET  /api/world/events  (SSE stream)
  Stream of: { type: "agent.status", agentName, status, ticketId, ... }
             { type: "ticket.created" | "ticket.approved" | "ticket.done" | ... }
             { type: "comment.added", ticketId, author, body }
  Auth:    none

POST /api/world/agent-mapping  (admin)
  Body: { agentName, character: "luffy" | "zoro" | ..., displayName? }
  Auth: API key
```

### Ny side i lab

```
src/app/world/page.tsx
  - Imports <GomuGomuWorld /> fra gomu-gomu-agents
  - Henter mapping fra config (file eller DB)
  - Forsyner pakken med assets-prefix (`/onepiece/`) og initial state
```

### Sidebar-entry

`Sidebar.tsx` får en ny gruppe-section "Verden" med link til `/world`.

---

## 8. Risici & open questions

| Risk | Sandsynlighed | Impact | Mitigation |
|---|---|---|---|
| Toei DMCA mod assets | Lav (privat repo) | Medium | Privat repo, ingen monetisering, fjern hvis henvendelse |
| PixiJS bundle-size for stor i lab | Medium | Lav | Lazy-load `/world` rute med dynamic import |
| Real-time skalerer dårligt | Lav (få samtidige users) | Medium | SSE i stedet for WS hvis simpler; Redis pub/sub hvis nødvendigt |
| Mobil-performance | Medium | Medium | Test tidligt på rigtig telefon, reducer animation-kompleksitet |
| Animation-arbejde tager længe | Høj | Medium | Start med 1-2 polerede karakterer; resten generic crew-look |

### Open questions (afgøres i kommende sessions)

1. Skal `gomu-gomu-agents` deploye sin egen demo-side til GitHub Pages? (lav prioritet)
2. Skal historiske events lagres et sted (replay)? — afgjort: nej, kun activity feed
3. Hvad sker der med arkiverede projekter i strawhats? — vis dem som "wrecks" på havets bund? (lav prioritet)

---

## 9. Næste konkrete skridt (efter denne fil)

1. ~~Research assets~~ ✓
2. ~~Skriv PLAN.md~~ ← du er her
3. **Bootstrap repo `gomu-gomu-agents`** ← næste
4. Bygge isometrisk verden med 4 statiske zoner
5. Real-time integration

---

## Appendix A: Asset research summary (2026-05-10)

**Free One Piece pixel-sprites på itch.io** (licens ikke specificeret af creators):
- One Piece — Monkey D. Luffy, 32x32 animeret (SpriteLunchbox)
- Animated Shanks (DAZ)
- Animated Monkey D. Dragon (DAZ)

**Paid:**
- 4 Mountains Island Backgrounds — $4.99 (CaptainSkolot)
- Luffy Gum Gum Gatling — $5 (Xhelixhi)

**Ikke brugbare**:
- Spriters Resource (rips fra Bandai/Toei spil — copyright)
- OpenGameArt (ingen One Piece — afviser ophavsretsbeskyttet)
- GitHub repos (joverlee521/One-Piece-RPG, wlwanpan/Luffy) — ingen LICENSE-filer

**Juridisk vurdering**:
- Toei Animation + Shueisha ejer rettighederne
- Fan-art tolereres for personlig/non-kommerciel brug
- Privat repo + intern brug = lav risiko
- Public npm publish + monetisering = højere risiko

**Konklusion**: Hybrid — free itch.io sprites med attribution + code-drawn for manglende karakterer. Begge repos forbliver private.

---

## Appendix B: Inspiration kilder

- [JamsusMaximus/codemap](https://github.com/jamsusmaximus/codemap) — primær arkitektur-reference (CodeMap-stil pixel-art med Canvas)
- [pablodelucca/pixel-agents](https://github.com/pablodelucca/pixel-agents) — Habbo-stil agent-visualisering
- [androoagi TikTok](https://www.tiktok.com/@androoagi) — space-station dashboard inspiration (#openclaw)
- [nathanhodgson_ TikTok — Pixel Agents](https://www.tiktok.com/@nathanhodgson_/video/7633311434918628630) — original CodeMap demo
