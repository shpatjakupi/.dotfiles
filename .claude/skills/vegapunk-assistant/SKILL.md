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
│   │   ├── claude-subprocess.ts   # spawn Claude Code CLI, parse stream-json + usage
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
| `/new` | Clear session, start fresh |
| `/project <name>` | Switch Claude's working directory |
| `/status` | Show session, usage, tokens, cost, rate limit, uptime |
| `/help` | Show available commands |
| (anything else) | Chat with Claude Code |

## How Changes Are Deployed

1. Edit code locally or via `/project vegapunk` in Telegram
2. Commit and push to GitHub
3. On VPS: pull and restart
```bash
ssh root@46.224.215.213
su - vegapunk -c "cd ~/projects/vegapunk && git pull"
cp -r /home/vegapunk/projects/vegapunk/src /home/vegapunk/projects/vegapunk/workspace /home/vegapunk/vegapunk/
chown -R vegapunk:vegapunk /home/vegapunk/vegapunk/src /home/vegapunk/vegapunk/workspace
systemctl restart vegapunk
```

## Available Projects (via /project)

| Command | Path on VPS | Repo |
|---------|------------|------|
| `/project backend` | `/home/vegapunk/projects/order-backend` | shpatjakupi/order-backend |
| `/project frontend` | `/home/vegapunk/projects/next-app-template` | shpatjakupi/next-app-template |
| `/project infra` | `/home/vegapunk/projects/infra-gitops` | shpatjakupi/infra-gitops |
| `/project dotfiles` | `/home/vegapunk/projects/.dotfiles` | shpatjakupi/.dotfiles |
| `/project vegapunk` | `/home/vegapunk/projects/vegapunk` | shpatjakupi/vegapunk |

## Inspiration: reclaw (Peter's bot)

Read `references/inspiration.md` for full details on reclaw's architecture and features.

Vegapunk is inspired by Peter's reclaw bot (`github.com/peterstorm/reclaw`) — a mature
Telegram-based AI assistant with scheduled skills, deep research, reminders, and more.

## When to Read References

- **Architecture, tech stack, VPS setup**: Read `references/architecture.md`
- **Reclaw features and inspiration for new features**: Read `references/inspiration.md`
- **Roadmap and planned features**: Read `references/roadmap.md`
