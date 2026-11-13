---
name: kidsapp-team
description: Full context for Bumble App (formerly KidsApp) — a web-tech game app for children 1-4 år with a bumblebee mascot, shipped to App Store and Google Play via Capacitor. TypeScript + Three.js/PixiJS, 3D assets made headless in Blender on the VPS by Claude. Sound-only, no language/text. Covers the project, the asset pipeline, the phase plan and the (mostly paused) agent team. Use for anything Bumble App / KidsApp-related.
---

# Bumble App (workspace `kidsapp`)

A mobile app with small games for children aged 1-4. Goal: Bimi Boo quality or better,
on the **App Store** and **Google Play**. Sound-only, no language, no readable text.

Unity was dropped on 2026-10-22. The old Unity repo `shpatjakupi/kids-app` is obsolete —
do not build on it.

## At a Glance

| | |
|--|--|
| App repo | `https://github.com/shpatjakupi/bumble-app` (private) |
| Plan | `PLAN.md` in the app repo — phases, constants, store requirements |
| Goals | `~/.claude/skills/kidsapp-team/GOALS.md` |
| Lab | `https://lab.gomuos.com/kidsapp` — workspace slug stays `kidsapp` (id 3) |
| VPS | Hetzner `46.224.215.213`, user `vegapunk`. Ubuntu 24.04, 8 cores, 15 GB RAM, **no GPU** |
| Blender | 5.2.3 LTS at `/opt/blender` → `/usr/local/bin/blender` |

## Constants

- **Target**: 1-4 år
- **No text, no speech** — sounds, icons, and the mascot's wordless expressions only
- **Mascot**: a bumblebee. Its buzz is its voice (happy, curious, confused, cheering)
- **Universe**: flower meadow, sun, honey, beehive
- **No punishment** — wrong attempts are funny, never "errors"

## Stack

| Role | Choice |
|---|---|
| Game engine | TypeScript + Vite. Three.js (3D) / PixiJS (2D) |
| Workshop | Blender headless on the VPS, driven by Claude via Python → `.glb` |
| 2D art | **OpenAI API** (GPT Image) — characters, poses, parts with transparent background |
| 3D art | **Blender only, built by Claude** — no image or 3D generators |
| Native apps | Capacitor → App Store + Google Play |
| iOS build | macOS runner in GitHub Actions — cannot be built on the VPS |
| Testing | Browser link on iPhone ≈ the real app (Capacitor uses the same WebKit as Safari) |

## Blender pipeline

Claude writes `bpy` scripts and runs them headless — no UI, no MCP needed:

```bash
sudo -u vegapunk nice -n 15 blender -b -P script.py
```

- Render previews with Cycles on CPU (low samples, ~5 s at 512px) and **look at the PNG**
  before showing anything to the human.
- Export `GLB` for Three.js. Keep meshes low-poly and textures small — cheap Android phones
  are the target's reality.
- Always `nice` heavy jobs: the VPS also runs ordrupspizza production.
- Everything 3D is modelled procedurally by script — shape-sorter box, shapes, meadow, the
  bumblebee. **No image→3D generators** (Shpat's rule, 2026-10-22).

## 2D pipeline (OpenAI API)

- Model: newest GPT Image via the API (not ChatGPT/Codex). Key in
  `/home/vegapunk/.config/bumble/openai.env` (chmod 600, never in git or chat).
- Budget: 30 $ prepaid credit with auto-recharge off — the hard cap. Make sure
  `OPENAI_API_KEY` is only loaded by the image script.
- Request transparent PNGs. Characters come as **parts** (head, body, wings, antennae, legs,
  eyes/brows/mouths per expression) for cutout animation in code, plus whole **key poses**.
  Always pass the approved character as reference image so it stays consistent.
- Continuous motion (wing flap, hover, blink, squash/stretch) is code, not frames.

## Assets in git

- Code and optimized runtime assets (`.glb`, compressed audio) — normal git.
- Source/work files (`.blend`, raw AI output, `.wav`) — Git LFS under `assets-src/`.
  Blender scripts are the real source; keep them in git so models can be regenerated.

## Phase plan

| Phase | Content |
|---|---|
| 0 | The right first game: shape sorter (triangle, square, circle) with the bumblebee — built in **both 2D and 3D** on one shared core, then pick one |
| 1 | Shared game framework + feel system + hub + reward |
| 2 | 5-8 games on the framework |
| 3 | Store-ready: parental gate, subscription, privacy, Capacitor builds, TestFlight/Play Internal |
| 4 | Launch + Google "Teacher Approved" |

Store rules (Apple Kids Category / Google Families Policy) apply from day one: no third-party
analytics or ads, parental gate in front of purchases and external links.

## Agent team

The workspace is **paused** (`pausedUntil` 9999-12-31) — the dispatcher ignores its tickets.
Work happens in interactive sessions; Lab tickets are the tracking board.

**There are no agents.** The manager, developer, reviewer and infra agents went with Unity
(2026-10-22), and the three idea scouts were removed 2026-11-13. New agents get designed once
phase 0 has produced a repeatable recipe.

In `vegapunk/src/infra/cron.ts`, `kidsapp-*` agent names still map to the `kidsapp` workspace
and to `/home/vegapunk/projects/bumble-app` as cwd, ready for future agents.

## Lab API

- URL `https://lab.gomuos.com/api`, header `Authorization: Bearer $LAB_API_KEY`
  (key in `/home/vegapunk/vegapunk/.env`)
- Every ticket body must include `"workspace": "kidsapp"`
- `POST /api/tickets` · `PATCH /api/tickets/{id}` · `POST /api/tickets/{id}/comments`
