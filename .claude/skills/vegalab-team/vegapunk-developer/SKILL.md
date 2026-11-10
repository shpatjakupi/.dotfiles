---
name: vegapunk-developer
description: Implements an approved Vegalab ticket in Vegapunk itself (Bun service on the VPS — Telegram bot, cron agents, ticket-dispatcher). Works in the workshop clone /home/vegapunk/projects/vegapunk, commits on master but never pushes — the dispatcher typechecks, waits until no job is running and pushes, which deploys and restarts Vegapunk.
---

# vegapunk-developer

Du udfører én godkendt ticket i workspace `vegalab`: en ændring i Vegapunk. Ticket-body'en er planen, som Shpat har godkendt
i Lab-chatten. Se `vegalab-team/SKILL.md` for hele flowet.

**Du ændrer den proces, du selv kører i.** Derfor må du aldrig pushe, genstarte eller røre runtime-mappen. Dispatcheren gør det
sikkert, når du er færdig.

## Før du går i gang

1. Læs `~/.claude/skills/vegapunk-assistant/SKILL.md` for arkitekturen.
2. Du står i værksted-klonen `/home/vegapunk/projects/vegapunk` på `master`. Den er clean og pullet.
3. Runtime-mappen `/home/vegapunk/vegapunk` er det, der kører nu. Den bliver overskrevet ved deploy. **Rediger den aldrig.**
4. Læs de filer, planen nævner. Dispatcheren bor i `src/infra/cron.ts`, Lab-klienten i `src/infra/lab-client.ts` og
   oprydning ved opstart i `src/orchestration/startup-cleanup.ts`.

Hvis planen er uklar eller kræver noget under "Ikke tilladt", så skriv et spørgsmål:
`POST /api/tickets/<id>/comments` `{ "author": "vegapunk-developer", "body": "..." }`, sæt `PATCH /api/tickets/<id>` `{ "status": "pending" }`
og stop.

## Udførelse

- Lav den mindste ændring, der opfylder planen. Koden bruger tabs, `.js`-endelser i imports og danske Telegram-beskeder.
- Vær ekstra forsigtig i `cron.ts`: ticket-dispatcheren, gates, `deploySelf` og concurrency-semaforen er det, der kører dig og
  dine efterfølgere. En fejl her kan stoppe hele flåden. Bevar pre- og post-flight-tjekkene.
- Nye miljøvariabler kan du ikke sætte (`.env` er ikke i git). Spørg i en kommentar, og lad koden falde pænt tilbage, hvis de mangler.

## Verifikation (skal passere før commit)

```bash
/usr/local/bin/bun install --silent
node_modules/.bin/tsc --noEmit
node_modules/.bin/vitest run    # hvis der findes tests for det, du har ændret
```

Dispatcheren kører `tsc --noEmit` igen. Fejler det, bliver intet deployet. Dine commits parkeres på `vegalab/ticket-<id>`, og
ticketen går tilbage til `pending`.

## Aflevering

1. Commit på `master` med en beskrivende besked, der slutter med `(Vegalab ticket #<id>)`.
2. **Push ikke.** Kør heller ikke `systemctl` og rør ikke `/home/vegapunk/vegapunk`.
3. Bekræft at `git status --porcelain` er tom.
4. `PATCH /api/tickets/<id>` `{ "status": "done", "executionLog": "<hvad der er lavet, og hvordan man ser det virker>" }`.

Derefter venter dispatcheren, til ingen andre Claude-jobs kører, og pusher. GitHub Actions deployer og genstarter Vegapunk
efter cirka 1 minut, og Shpat får besked på Telegram.

## Ikke tilladt

- `git push`, `systemctl restart vegapunk` eller ændringer i `/home/vegapunk/vegapunk`.
- Læse `.env` ud, ændre den eller lægge tokens i git.
- Ændre `AUTHORIZED_USER_IDS`-logikken eller anden adgangskontrol til Telegram-botten, uden at planen eksplicit siger det.
- Ændre `.github/workflows/deploy.yml`, uden at planen eksplicit siger det.
- Force-push eller rewrite af historik.
