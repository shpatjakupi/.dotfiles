# Vegapunk Roadmap

Features planned or considered for Vegapunk, roughly ordered by priority.

## Phase 1: Foundation (DONE)
- [x] Telegram bot with grammy
- [x] Claude Code subprocess execution
- [x] Redis session persistence (30-day TTL)
- [x] Project switching (`/project <name>`)
- [x] Message splitting for long responses
- [x] Authorization (allowlisted Telegram user IDs)
- [x] systemd service with auto-restart
- [x] GitHub repos cloned on VPS
- [x] Skills synced from .dotfiles
- [x] `/status` command (usage, tokens, cost, rate limits, uptime)
- [x] Personality injection (`personality.md` on first session message)
- [x] Claude self-awareness (`workspace/CLAUDE.md`)
- [x] GitHub repo created (shpatjakupi/vegapunk)

## Phase 2: Core Features
- [ ] **BullMQ job queues** — proper async job handling with retry and dead-letter
- [ ] **Reminder system** — `/remind <duration> <message>` and `/remind every <interval> <message>`
- [ ] **Streaming responses** — live message editing as Claude types
- [ ] **Deploy command** — `/deploy` to pull latest code and restart vegapunk
- [ ] **Sync command** — `/sync` to pull dotfiles and update skills on VPS

## Phase 3: Scheduled Skills
- [ ] **YAML skill system** — define skills as YAML with cron schedule and prompt
- [ ] **Hot-reload** — chokidar watcher for skill files, no restart needed
- [ ] **Cron scheduler** — trigger skill jobs on schedule
- [ ] **ALL_CLEAR sentinel** — suppress notification when nothing to report
- [ ] **Morning briefing** — weather, calendar, priorities
- [ ] **Tech digest** — curated news from RSS feeds
- [ ] **VPS watchdog** — health checks every 6 hours

## Phase 4: Knowledge & Memory
- [ ] **Obsidian vault** — structured note storage on VPS
- [ ] **Vault review** — spaced repetition from notes
- [ ] **Journal system** — daily notes from Telegram
- [ ] **Cortex memory plugin** — persistent memory across sessions

## Phase 5: Advanced
- [ ] **Deep research pipeline** — NotebookLM integration
- [ ] **Podcast generation** — audio overviews from notes
- [ ] **Fitness tracking** — Garmin/Strava integration
- [ ] **Calendar integration** — Google Calendar or iCloud
- [ ] **Containerization** — Docker image, deploy via K3s + ArgoCD

## Ideas (not prioritized)
- Expense tracking from Telegram messages
- GitHub PR review alerts
- Learning plans with daily tasks
- Meeting prep briefings
- Code review from phone
- Multi-user support (family/team)
