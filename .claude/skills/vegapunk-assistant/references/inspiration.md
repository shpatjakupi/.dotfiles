# Inspiration: reclaw by peterstorm

Vegapunk is inspired by Peter's reclaw bot (`github.com/peterstorm/reclaw`), a mature
Telegram-based AI assistant running on NixOS. This document captures reclaw's features
as a reference for what Vegapunk could become.

## reclaw Architecture

Same core pattern as Vegapunk but with more layers:
- **grammy** for Telegram
- **BullMQ + Redis** for job queues (5 queues: chat, scheduled, reminder, research, podcast)
- **Claude Code subprocess** with streaming output and live message editing
- **Hot-reloadable YAML skill system** via chokidar file watcher
- **Cron scheduler** that reconciles skills to BullMQ repeatable jobs
- **Mutex** on chat jobs to prevent concurrent Claude sessions corrupting state

## Features Vegapunk Doesn't Have Yet

### 1. Scheduled Skills (cron jobs)
YAML files in `workspace/skills/` define automated tasks with cron schedules.
Skills run Claude Code on a schedule and send results to Telegram.

**reclaw's skills:**
| Skill | Schedule | What it does |
|-------|----------|--------------|
| Morning Briefing | Weekdays 07:20 | Weather, Garmin fitness, calendar, priorities |
| Tech Digest | Daily 08:00 | Curated articles from 11 RSS sources |
| Vault Review | 4x daily | Spaced repetition from Obsidian notes |
| Evening Report | Daily 21:00 | Day recap, tomorrow preview, journal prompt |
| Garmin Sync | Daily 22:00 | Fitness data -> Obsidian notes |
| Homelab Watchdog | Every 6h | 10 health checks, alert only on issues |
| Weekly Review | Sunday 10:00 | Week summary from journal + git |
| Insights Engine | Wednesday 11:00 | Behavioral pattern analysis |
| Monthly Review | 1st of month | Month retrospective |
| Self Improvement | Sunday 02:00 | Reclaw analyzes its own code for improvements |
| Cortex Prune | Daily midnight | Memory cleanup |

### 2. Reminder System
`/remind 2h take vitamins` — one-shot delayed messages
`/remind every weekday at 9am check inbox` — recurring via BullMQ

### 3. Deep Research Pipeline
`/research <topic>` triggers a 10-step state machine:
1. Create NotebookLM notebook
2. Search for sources
3. Add sources (URLs, YouTube)
4. Wait for processing
5. Generate research questions via Claude
6. Query NotebookLM per question
7. Resolve citations to wikilinks
8. Write structured Obsidian vault notes
9. Generate audio/podcast (optional)
10. Send summary to Telegram

Features crash recovery (resumes from last checkpoint) and quota tracking.

### 4. Podcast Generation
`/podcast <vault-note>` generates audio overview via NotebookLM.

### 5. Obsidian Vault Integration
Writes structured markdown notes to an Obsidian vault:
- Hub notes with metadata and quality grades
- Source notes with URLs and topics
- Q&A notes with resolved citations
- Daily journal entries
- Fitness tracking notes
- Weekly/monthly reviews

### 6. Cortex Memory Plugin
Persistent memory across Claude sessions:
- Auto-extracts insights from conversations
- `/recall` searches memories semantically
- `/remember` stores explicit memories
- Surfaces relevant context at session start

### 7. Streaming with Live Updates
Real-time message editing in Telegram as Claude thinks/types.
Edit throttling (1.5s) to avoid Telegram rate limits.

### 8. iCloud Calendar Integration
Add events via natural language: "sæt tandlæge tirsdag kl. 14"
Syncs bidirectionally via vdirsyncer.

### 9. Garmin Connect Integration
Fetches sleep, HR, steps, activities, VO2 Max, training readiness, HRV.
Custom script (`scripts/garmin-fetch.ts`) handles auth and data extraction.

### 10. Personality System
`workspace/personality.md` shapes Claude's tone across all interactions.

## Key Architectural Patterns from reclaw

### YAML Skill Format
```yaml
name: Morning Briefing
description: Personalized morning briefing for weekday commutes
schedule: "20 7 * * 1-5"
timeout: 300
permission_profile: scheduled
validity_window: 60
prompt: |
  Step 1: Fetch weather...
  Step 2: Check calendar...
```

### ALL_CLEAR Sentinel
Scheduled skills output "ALL_CLEAR" to suppress Telegram notifications.
Useful for watchdogs that should only alert on problems.

### Fire-and-forget Patterns
Cortex extraction and vault syncing run async after main response.
Never block the user's reply.

### Functional Core / Imperative Shell
Pure functions in `src/core/`, all I/O in `src/infra/`.
Makes testing easy — no mocks needed for business logic.
