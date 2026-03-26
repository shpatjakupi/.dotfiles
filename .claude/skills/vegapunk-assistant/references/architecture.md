# Vegapunk Architecture

## Tech Stack

- **Runtime**: Bun
- **Language**: TypeScript (strict mode)
- **Telegram**: grammy
- **Session store**: Redis (ioredis)
- **Validation**: Zod
- **Linting**: Biome
- **Tests**: Vitest

## VPS Environment

- **Server**: Hetzner CX43 (8 vCPU, 16GB RAM, Ubuntu 24.04)
- **IP**: 46.224.215.213
- **User**: `vegapunk` (non-root, but with full sudo/kubectl/journalctl access)
- **Permissions**: passwordless sudo, `adm` + `systemd-journal` groups, own kubeconfig
- **Service**: systemd unit `vegapunk.service`
- **Redis**: runs in K3s cluster, `assistant` namespace, ClusterIP accessed from host
- **Claude Code**: authenticated as `vegapunk` user
- **GitHub CLI**: authenticated as `shpatjakupi`, full repo + workflow access
- **kubectl**: `KUBECONFIG=/home/vegapunk/.kube/config` — full cluster access
- **Skills**: synced from `.dotfiles` repo to `~/.claude/skills/`

## Key Design Decisions

### Non-root execution with full access
Claude Code blocks `--dangerously-skip-permissions` under root/sudo. Vegapunk runs as
a dedicated `vegapunk` user with its own home directory, Claude auth, and project files.
The user has passwordless sudo (`/etc/sudoers.d/vegapunk`), own kubeconfig
(`~/.kube/config`), and membership in `adm` + `systemd-journal` groups — giving it
the same capabilities as root for all practical purposes.

### Redis for sessions
Each Telegram chat ID maps to a Claude session ID and active project name in Redis.
Sessions persist for 30 days. `/new` clears the session, `/project <name>` switches
the working directory and starts a fresh session.

### Usage tracking
Each message tracks input/output tokens, cache tokens, cost (USD), and duration.
Values are accumulated per session in Redis and displayed via `/status`. Rate limit
events are also captured and shown.

### Stream-JSON parsing
Claude Code outputs stream-json format. We only extract the final `result` event
to avoid duplicate text (earlier version captured both deltas and result, causing
doubled responses). Also parses `rate_limit_event` for rate limit status/reset time.

### Project switching
`/project <name>` changes the `cwd` passed to Claude Code subprocess. Each project
maps to a git repo cloned under `/home/vegapunk/projects/`. Session is cleared on
project switch to avoid context bleeding.

### Personality injection
On the first message of a new session (no existing sessionId), `personality.md` is
prepended as system context. This sets tone (concise, Danish, One Piece reference).

### Workspace files
`workspace/CLAUDE.md` gives Claude self-awareness (who it is, where it runs, what
systems are available). `workspace/personality.md` defines tone and behavior.

## Environment Variables

| Variable | Description |
|----------|-------------|
| `TELEGRAM_BOT_TOKEN` | From @BotFather |
| `AUTHORIZED_USER_IDS` | Comma-separated Telegram user IDs |
| `REDIS_HOST` | Redis ClusterIP in K3s |
| `REDIS_PORT` | Default 6379 |
| `WORKSPACE_PATH` | Default working directory for general chat |

## File Locations on VPS

| What | Path |
|------|------|
| Vegapunk runtime | `/home/vegapunk/vegapunk/` |
| Vegapunk git repo | `/home/vegapunk/projects/vegapunk/` |
| Project repos | `/home/vegapunk/projects/*` |
| Workspace files | `/home/vegapunk/vegapunk/workspace/` |
| CLAUDE.md | `/home/vegapunk/vegapunk/workspace/CLAUDE.md` |
| personality.md | `/home/vegapunk/vegapunk/workspace/personality.md` |
| Claude config | `/home/vegapunk/.claude/` |
| Skills | `/home/vegapunk/.claude/skills/` |
| Dotfiles repo | `/home/vegapunk/projects/.dotfiles/` |
| .env file | `/home/vegapunk/vegapunk/.env` |
| systemd unit | `/etc/systemd/system/vegapunk.service` |
