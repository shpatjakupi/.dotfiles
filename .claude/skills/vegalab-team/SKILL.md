---
name: vegalab-team
description: Vegalab — the workspace where Shpat orders changes to the Lab (lab.gomuos.com) and to Vegapunk itself from the Lab chat. Covers the flow (chat → ⚡ Udfør → one approval → Vegapunk executes and deploys), the two agents lab-developer and vegapunk-developer, what the dispatcher enforces, and the safety rules. Use when working on or with the Vegalab flow.
---

# Vegalab-teamet

Vegalab er workspacet (slug `vegalab`) til ændringer i **Lab'en selv** og i **Vegapunk**. Shpat bestiller dem i Lab-chatten
(`/chat`), og Vegapunk udfører og deployer dem. Bygget 2026-11-10.

## Flowet

1. Shpat sparrer med Claude i `/chat`. Chatten har koden til begge repos (`./code/gomuos-lab`, `./code/vegapunk`), dotfiles og et
   øjebliksbillede af Lab'en, men kan ikke ændre noget selv.
2. Shpat trykker **⚡ Udfør**. Claude skriver en plan (mål, ændringer, database, acceptkriterier, risici) og vælger agent.
3. **Én godkendelse:** "Godkend og udfør" opretter en ticket i workspace `vegalab` med status `approved` og
   `assignedAgent` = agenten (`delegatedBy: "lab-chat"`). Der er ingen anden godkendelse.
4. Vegapunks `ticket-dispatcher` (hvert 3. minut) tager ticketen og kører agenten med 40 min timeout.
5. Dispatcheren deployer, når agenten er færdig (se nedenfor), og sender Telegram-besked.

Planen er hele ticket-body'en. Agenten ser ikke chatten.

## Agenterne

| Agent | Repo (cwd) | Branch | Deploy |
|---|---|---|---|
| `lab-developer` | `/home/vegapunk/projects/gomuos-lab` | main | Agenten pusher. Dispatcheren venter på CI-buildet af commit'en, kører `kubectl rollout restart deploy/gomuos-lab -n gomuos` og smoke-tester `/login`. |
| `vegapunk-developer` | `/home/vegapunk/projects/vegapunk` (værksted-klonen) | master | Agenten committer **uden at pushe**. Dispatcheren typechecker, venter til ingen andre Claude-jobs kører og pusher. GitHub Actions rsync'er til `/home/vegapunk/vegapunk` og genstarter. |

Skills: `vegalab-team/lab-developer/SKILL.md` og `vegalab-team/vegapunk-developer/SKILL.md`.

## Det dispatcheren håndhæver (src/infra/cron.ts i vegapunk)

- **Pre-flight:** repoet skal være clean og ikke foran origin. Ellers skippes ticketen, sættes til `pending`, og Shpat får besked.
  Derefter `git pull --ff-only`.
- **lab-developer post-flight:** intet uncommitted, intet upushet. Ellers `pending`. Udrulningen sker kun, hvis HEAD flyttede sig.
  Hvis CI eller rollout fejler, går ticketen til `pending` med en kommentar.
- **vegapunk-developer post-flight:** intet uncommitted, og `tsc --noEmit` skal passere. Ellers parkeres commits på branch
  `vegalab/ticket-<id>`, master nulstilles til origin, og ticketen går til `pending` med en kommentar.
- **Ingen selv-genstart midt i et job:** under en selv-deploy starter ingen nye Claude-kald, og pushet sker først, når de
  kørende er færdige (højst 45 min). Efter genstart sætter startup-cleanup `in_progress`-tickets tilbage til `approved`.
- **Workspace-pause:** dispatcheren kører uanset pause, men tickets i et pauset workspace venter. GomuOS og Bumble er pauset,
  Vegalab er aktivt.

## Regler for begge agenter

- Gør præcis det, planen siger, og ikke mere. Er planen uklar eller forkert, så skriv et spørgsmål som kommentar, sæt ticketen
  til `pending` og stop.
- Rør aldrig secrets: `gomuos-lab-auth`, `gomuos-lab-secret`, Vegapunks `.env`. Læg aldrig nøgler i git.
- Svæk aldrig login, 2FA eller API-auth i Lab'en uden at planen eksplicit beder om det.
- Ingen force-push, ingen `git reset` på delte branches, ingen ændringer i infra-gitops. Hvis det kræves, så spørg i en kommentar.
