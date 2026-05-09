---
name: vegapunk-assistant
description: >
  Full project context for Vegapunk — a personal AI assistant Telegram bot that bridges
  messages to Claude Code running on a Hetzner VPS. Use this skill when working on the
  vegapunk project: adding features, fixing bugs, deploying, or planning new capabilities.
  Triggers when user mentions: vegapunk, telegram bot, assistant bot, VPS bot, reclaw,
  scheduled skills, reminders, or wants to add new features to their personal AI assistant.
---

# Vegapunk

Personal AI assistant accessible via Telegram, running Claude Code on a Hetzner VPS.
Named after the scientist from One Piece who is 500 years ahead of his time.

## Quick Access

**Repos:**
- Local: `C:\Users\shpat\Desktop\projects\vegapunk`
- VPS: `/home/vegapunk/vegapunk` (runtime) + `/home/vegapunk/projects/vegapunk` (git repo)
- GitHub: `github.com/shpatjakupi/vegapunk` (private)

**VPS service:**
```bash
ssh root@46.224.215.213
systemctl status vegapunk        # check status
systemctl restart vegapunk       # restart after changes
journalctl -u vegapunk -f        # live logs
```

**Telegram bot:** @Vegapunk007_ai_bot

## Architecture

Read `references/architecture.md` for full technical details.

Quick overview:
```
Telegram (phone) --> grammy bot --> message router
                                        |
                    /new, /help, /status, /project <name>
                                        |
                                   chat handler
                                   (injects personality.md on first msg)
                                        |
                              Claude Code subprocess
                              (cwd = active project)
                              (reads workspace/CLAUDE.md)
                                        |
                                  Redis (sessions + usage tracking)
                                  (K3s, assistant ns)
```

## Project Structure

```
vegapunk/
├── src/
│   ├── main.ts                    # bootstrap, wiring, startup
│   ├── infra/
│   │   ├── config.ts              # env vars -> typed config (Zod)
│   │   ├── session-store.ts       # Redis: sessions, projects, usage, rate limits
│   │   ├── claude-subprocess.ts   # spawn Claude Code CLI, stream thinking/text/tool events
│   │   ├── markdown-to-telegram.ts # convert Claude markdown to Telegram HTML
│   │   └── telegram.ts           # grammy adapter, message splitting
│   └── orchestration/
│       ├── message-router.ts      # command routing + project paths
│       └── chat-handler.ts        # send prompt to Claude, personality injection
├── workspace/
│   ├── CLAUDE.md                  # self-awareness: who am I, what can I access
│   └── personality.md             # tone: concise, Danish, One Piece reference
├── package.json
├── tsconfig.json
├── biome.json
└── .env.example
```

## Telegram Commands

| Command | What it does |
|---------|-------------|
| `/new` | Clear session, show command overview |
| `/help` | Show command overview |
| `/status` | Show session, usage, tokens, cost, rate limit, uptime |
| `/project <name>` | Switch Claude's working directory |
| `/restart` | Self-restart via detached systemctl |
| `/refresh` | Sync runtime from git repo + restart (fix: runtime edited directly) |
| `/rollback` | `git reset --hard HEAD~1` + deploy + restart (fix: bad code in git) |
| (anything else) | Chat with Claude Code |

## How Changes Are Deployed

**Automatic** — `.github/workflows/deploy.yml` rsyncs `src/` + manifest files to
`/home/vegapunk/vegapunk/` on every push to `master` that touches source files,
runs `bun install`, restarts the systemd unit, and verifies it's active. End-to-end
in ~18 seconds.

```
git push origin master   # done — workflow handles the rest
```

Manual trigger if needed: `gh workflow run deploy.yml --repo shpatjakupi/vegapunk`.

**On failure** the workflow posts a Telegram message (same bot used at runtime)
with the short SHA, commit subject, and a link to the failed run.

**Secrets used** (set via `gh secret set`):
- `VPS_SSH_KEY` — dedicated ed25519 deploy key, separate from personal SSH (revocable).
  Public key lives in `/root/.ssh/authorized_keys` on the VPS.
- `VPS_HOST` — `46.224.215.213`
- `TELEGRAM_BOT_TOKEN` + `TELEGRAM_CHAT_ID` — for failure notification.

**`bun.lock` is not committed** — workflow runs plain `bun install` (not
`--frozen-lockfile`). With only 3 deps that's fine; if you ever want reproducible
builds, commit the lockfile and add `--frozen-lockfile` back.

## Available Projects (via /project)

| Command | Path on VPS | Repo |
|---------|------------|------|
| `/project backend` | `/home/vegapunk/projects/order-backend` | shpatjakupi/order-backend |
| `/project frontend` | `/home/vegapunk/projects/next-app-template` | shpatjakupi/next-app-template |
| `/project infra` | `/home/vegapunk/projects/infra-gitops` | shpatjakupi/infra-gitops |
| `/project dotfiles` | `/home/vegapunk/projects/.dotfiles` | shpatjakupi/.dotfiles |
| `/project vegapunk` | `/home/vegapunk/projects/vegapunk` | shpatjakupi/vegapunk |
| `/project indfoedsret` | `/home/vegapunk/projects/indfoedsret-app` | shpatjakupi/indfoedsret-app |

## Inspiration: reclaw (Peter's bot)

Read `references/inspiration.md` for full details on reclaw's architecture and features.

Vegapunk is inspired by Peter's reclaw bot (`github.com/peterstorm/reclaw`) — a mature
Telegram-based AI assistant with scheduled skills, deep research, reminders, and more.

## Agent Fleet Hosted Here

Vegapunk's `src/infra/cron.ts` is the runtime for **four independent agent teams**:
gomuos, strawhats, kidsapp, and revolutionaries. Each has its own skill folder and
its own workspace in the lab dashboard.

**For any question about which agents exist, when they run, or how tickets flow
between teams, read `agent-fleet` skill first** — it's the master index. Each
team's own skill (`gomuos-team`, `strawhat-team`, `kidsapp-team`, `revolutionary-team`)
has the deep dive.

Vegapunk-specific responsibilities for the fleet:
- `cron.ts` registers every job with its workspace via `workspaceForJob(name)` (prefix-based routing).
- Daily/weekly hunters/scouts align to their declared UTC time via `parseCron` + `msUntilNextFire`.
- The ticket-dispatcher polls every 3 minutes and spawns Claude Code subprocesses
  with `cwd` chosen by `getCwdForTicket` (different repo per agent prefix).
- All agent jobs call back to the lab via `lab-client.ts` (`updateJobStatus`, `createTicket`).

## When to Read References

- **Architecture, tech stack, VPS setup**: Read `references/architecture.md`
- **Reclaw features and inspiration for new features**: Read `references/inspiration.md`
- **Roadmap and planned features**: Read `references/roadmap.md`
- **Agent fleet (4 teams) overview**: Read the `agent-fleet` skill (separate skill in same dotfiles)
