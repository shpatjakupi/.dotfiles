---
name: gomuos-lab
description: GomuOS Lab — Shpat's dashboard at lab.gomuos.com (he calls it "vegalab"). Three workspaces (gomuos for the food platform, kidsapp shown as "Bumble", vegalab for changes to the Lab and Vegapunk), 2FA login, image attachments, a built-in Claude chat (/chat) with ⚡ Udfør execution plans, the agent-fleet page (/fleet), Prisma schema, API routes, ticket status flow, UI components, deployment, migrations and gotchas. Use when building or debugging the Lab itself (not when working with agent skills or the order app).
---

# GomuOS Lab

Shpat's control center for the agent fleet. Agents create tickets; Shpat reviews, comments
(with images), approves or rejects, and spars with Claude in `/chat`. Lives at `lab.gomuos.com`.

## Tech Stack

| | |
|--|--|
| Framework | Next.js 14 (App Router), TypeScript |
| Database | MySQL via Prisma ORM |
| Styling | Tailwind CSS with custom `lab-*` theme tokens |
| Auth | Password + TOTP (2FA) → HMAC-signed session cookie; agents use `LAB_API_KEY` |
| Chat | Claude Code CLI (`claude -p`) inside the Lab pod |
| Hosting | K8s, namespace `gomuos`, server `46.224.215.213` |

**Local path:** `C:\Users\shpat\Desktop\projects\gomuos-lab`
**GitHub:** `shpatjakupi/gomuos-lab`
**K8s manifest:** `infra-gitops/apps/lab/`

## Workspaces

Every Ticket and Job belongs to exactly one workspace. Straw Hats and Revolutionaries were
retired and deleted on 2026-11-10 (DB rows, pages, APIs and the Project model are gone).

| Slug | Shown as | Purpose | UI |
|------|----------|---------|-----|
| `gomuos` | GomuOS | Maintains the food-ordering platform | `/tickets`, `/hunters`, `/jobs` |
| `kidsapp` | Bumble | Bumble App (children's app with the bumblebee) | `/kidsapp`, `/kidsapp/agents` |
| `vegalab` | Vegalab | Changes to the Lab and Vegapunk, ordered from `/chat` (⚡ Udfør) and executed by Vegapunk | `/vegalab` |

The slug `kidsapp` stays so URLs, tickets and the `kidsapp-*` agent prefix keep working.

## Auth

- **Pages**: `middleware.ts` redirects to `/login` unless the `lab_session` cookie verifies.
- **`/api/*`**: middleware requires the session cookie OR `Authorization: Bearer $LAB_API_KEY`
  on every route except `/api/auth/login` and `/api/auth/logout`. (Before 2026-11-10 `/api/*`
  was open and ticket PATCH, comments and skill commits had no check.) Routes may add stricter
  checks — the chat routes accept the session only.
- **Login**: `LAB_PASSWORD` + 6-digit TOTP code (`TOTP_SECRET`, RFC 6238, SHA-1/30 s,
  `src/lib/totp.ts`). 5 failures within 15 min lock login. Without `TOTP_SECRET` the code
  isn't required.
- **Session**: `"<expires>.<hmac(expires)>"` signed with `SESSION_SECRET` (`src/lib/session.ts`,
  Web Crypto so it runs in edge middleware). 30-day lifetime. Rotating `SESSION_SECRET` logs
  everyone out.
- **QR for the authenticator app**: Shpat runs `ssh root@46.224.215.213 /root/lab-2fa-qr.sh`
  in his own terminal. Never run it inside a Claude session — the secret would land in the transcript.

## Prisma Schema (abridged)

```prisma
model Workspace  { id, slug @unique, name, description?, pausedUntil?, tickets, jobs }      // lab_workspaces
model Job        { id, workspaceId, name @unique, description, schedule, enabled,
                   model?, reasoningEffort?, notifyOnFailure, dailyCostCapUsd?, pausedUntil?,
                   runNowRequestedAt?, lastRunAt?, lastStatus?, lastOutput?, tickets, runs } // lab_jobs
model JobRun     { id, jobId (cascade), status, output?, costUsd?, model?, startedAt, finishedAt? } // lab_job_runs
model Setting    { key @id, value }   // concurrency_limit | vacation_mode_until | vegapunk_restart_requested_at
model Ticket     { id, workspaceId, jobId?, title, body, severity, status, assignedAgent?,
                   delegatedBy?, createdAt, updatedAt, resolvedAt?, executions, comments, attachments } // lab_tickets
model Comment    { id, ticketId?, author, body, createdAt }                                  // lab_comments
model Execution  { id, ticketId, status, log?, startedAt, finishedAt? }                      // lab_executions
model Attachment { id, ticketId? (cascade), filename, mimeType, sizeBytes, data LongBlob, uploadedBy, createdAt } // lab_attachments
model ChatThread { id, title, createdAt, updatedAt, messages }                               // lab_chat_threads
model ChatMessage{ id, threadId (cascade), role user|assistant, content, status pending|done|error, createdAt,
                   planTitle?, planBody?, planAgent?, planStatus? pending|approved|rejected, ticketId? } // lab_chat_messages
```

Full definitions: `prisma/schema.prisma`.

## Ticket Status Flow

```
pending ──→ approved ──→ in_progress ──→ done
   │                                      ↑
   │         ↓ (human clicks "Spørg agent")
   │      needs_response ──→ in_progress ──→ pending (agent answered)
   │
   └──→ rejected
```

| Status | Meaning |
|--------|---------|
| `pending` | Waiting for human review |
| `approved` | Human approved — dispatcher will spawn the agent |
| `in_progress` | Dispatcher has picked it up, agent is running |
| `needs_response` | Human posted a question via "Spørg agent" |
| `done` | Agent completed the task |
| `rejected` | Human rejected it |

`resolvedAt` is set when status becomes `done` or `rejected`.

## API Routes

Every route needs the session or the API key (see Auth).

| Method | Path | Notes |
|--------|------|-------|
| `GET` | `/api/tickets` | `?status=` and `?workspace=` filters |
| `POST` | `/api/tickets` | API key only. Body: `{ title, description, severity, jobName?, assignedAgent?, delegatedBy?, workspace? }`. `workspace` defaults to `"gomuos"`. |
| `GET` | `/api/tickets/:id` | Includes `job`, `executions`, `comments` |
| `PATCH` | `/api/tickets/:id` | Update `status`, `assignedAgent`. Also accepts `executionLog`+`executionStatus` to create an Execution row |
| `GET` | `/api/tickets/:id/poll` | Returns `{ id, status }` — used by Vegapunk to wait for approval |
| `GET` / `POST` | `/api/tickets/:id/comments` | Body: `{ author, body, askAgent? }`. `askAgent: true` → `needs_response` |
| `POST` | `/api/attachments` | Multipart: `file` (PNG/JPEG/WebP/GIF, ≤ 5 MB, type checked by magic bytes — no SVG), `ticketId?`, `uploadedBy?`. Returns `{ id, url, markdown }` — paste `markdown` into a ticket body or comment |
| `GET` | `/api/attachments/:id` | Serves the image (session or API key) |
| `GET` / `POST` | `/api/chat/threads` | Session only. POST `{ message }` starts a thread |
| `GET` / `DELETE` | `/api/chat/threads/:id` | Thread with messages |
| `POST` | `/api/chat/threads/:id/messages` | `{ message }` — 409 while a reply is pending |
| `POST` | `/api/chat/threads/:id/plan` | ⚡ Udfør — Claude writes an execution plan (session only) |
| `POST` | `/api/chat/plans/:id` | `{ action: approve\|reject }` — approve creates an approved Vegalab ticket |
| `GET` / `POST` | `/api/jobs` | List (`?workspace=`) / upsert. Terminal `lastStatus` appends a `JobRun` |
| `PATCH` | `/api/jobs/:name` | Controls: `schedule`, `enabled`, `model`, `reasoningEffort`, `notifyOnFailure`, `dailyCostCapUsd`, `pausedUntil`, `runNowRequestedAt` (null only) |
| `POST` | `/api/jobs/:name/run-now` | Vegapunk's poller fires + clears within 30 s |
| `GET` | `/api/jobs/:name/runs?limit=10` | Last N JobRun rows |
| `GET` | `/api/jobs/cost-summary` | `[{name, dailyUsd}]` — source of truth for the daily cost cap |
| `GET` / `PATCH` | `/api/workspaces/:slug` | `{ pausedUntil: ISO\|"forever"\|null }` pauses the whole team |
| `GET` / `PATCH` | `/api/settings` | Global k/v settings |
| `PUT` | `/api/skills/file` | Commits a skill `.md` to `.dotfiles` via `GITHUB_TOKEN` |
| `POST` | `/api/skills/feedback`, `/api/agents/feedback` | Create skill/agent feedback tickets |

Agents attaching images: `curl -H "Authorization: Bearer $LAB_API_KEY" -F file=@x.png -F ticketId=42 -F uploadedBy=<agent> https://lab.gomuos.com/api/attachments`, then put the returned `markdown` in a comment.

## Comments and images

- Ticket comments (`CommentThread.tsx`): "Spørg agent" flips the ticket to `needs_response`; the
  dispatcher runs the agent in respond-only mode and it flips back to `pending`.
- Images: 📎 button or Ctrl+V in the comment box uploads and inserts `![alt](/api/attachments/N)`.
  `MarkdownBody` renders images inline (click → full size).
- `author === "human"` → human bubble; anything else is an agent.

## Chat (`/chat`)

Lab's own Claude engine — independent of Vegapunk. `src/lib/chat.ts`:
- Each turn runs `claude -p --tools Read,Grep,Glob` in `/data/chat` inside the pod. It can read
  and propose, never change anything itself.
- Context, refreshed at most every 5 min: `./dotfiles` (tarball of the `.dotfiles` repo),
  `./code/gomuos-lab` and `./code/vegapunk` (tarballs via `GITHUB_TOKEN`, best effort) and
  `./lab` (workspaces.md, jobs.md, tickets/<slug>.md with comments). Images referenced in the
  thread are written to `./images` so Claude can Read them.
- The last 30 messages go into the prompt (no `--resume`). The reply runs in the background; the
  UI polls every 2.5 s. A reply pending > 10 min is marked as lost (pod restart).
- Model from `CLAUDE_MODEL` (Opus 5.5); auth from `CLAUDE_CODE_OAUTH_TOKEN` (shared with Vegapunk
  and pizzacrew-inbox, expires 2027-08-18).

### ⚡ Udfør (execution plans → Vegalab)

- `POST /api/chat/threads/:id/plan` adds the user message "⚡ Udfør — …" and a pending assistant
  turn that runs `runClaudeJson` with `{reply, agent: lab-developer|vegapunk-developer|none, title, plan}`.
  The plan is stored on the message (`planTitle`, `planBody`, `planAgent`, `planStatus`, migration 011).
- `POST /api/chat/plans/:id {action: approve|reject}` (session only). **Approve is the one approval**:
  it creates a ticket in workspace `vegalab` with `status: "approved"`, `assignedAgent` = the agent and
  `delegatedBy: "lab-chat"`, stores `ticketId` and supersedes older pending plans in the thread.
- Vegapunk's dispatcher then runs the agent and deploys. The flow and the agents are described in
  the `vegalab-team` skill.

## Skill editor (✨ Forbedr med AI)

Every `SkillCard` (on `/fleet` and the team agent pages) has a chat per skill file — modelled on
pizzacrew-inbox's Playbook studio. `src/lib/skill-chat.ts`:
- Claude gets the current file (fresh from GitHub via `readSkillFile`), the last 20 messages for that
  path, and Read/Grep/Glob over the dotfiles export; it answers with JSON `{reply, summary, edits[]}`.
- Edits are search/replace (`applyEdits` in `src/lib/diff.ts`, one retry if a search text misses). The
  full new content is stored as the proposal with the blob sha it was made against.
- **Anvend** re-reads the file; if the sha moved on it refuses, otherwise `commitSkillFile` commits to
  dotfiles and other pending proposals for that file are rejected. Table `lab_skill_chat_messages`
  (migration 010). Routes: `GET/POST /api/skills/chat?path=`, `POST /api/skills/chat/:id {action}` — session only.

## UI Structure

```
src/
├── app/
│   ├── login/                 password + TOTP code
│   ├── tickets/, hunters/, jobs/        (gomuos)
│   ├── kidsapp/, kidsapp/agents/        (Bumble)
│   ├── fleet/                 Agent-flåden — mirrors projects/agent-fleet.code-workspace (ENTRIES)
│   ├── chat/                  Chat med Claude
│   ├── settings/              vacation mode, concurrency, restart-vegapunk
│   └── api/                   (see routes above)
├── components/
│   ├── Sidebar.tsx            groups: GomuOS / Bumble / Globalt
│   ├── ChatPanel.tsx          thread list + messages + image upload + polling
│   ├── CommentThread.tsx, MarkdownBody.tsx
│   ├── TeamSkills.tsx         skill cards for a team, or any folder via `folder` (used by /fleet)
│   ├── TeamAgents.tsx, TeamJobs.tsx, JobRow.tsx, AgentControls.tsx, TeamPauseControl.tsx
│   ├── StatusBadge.tsx, TicketActions.tsx, ScheduleEditor.tsx, SettingsForm.tsx, Tooltip.tsx
│   └── SkillCard.tsx, SkillFeedbackForm.tsx, AgentCard.tsx, AgentFeedbackForm.tsx
└── lib/
    ├── session.ts             HMAC session, API key check, password check (edge-safe)
    ├── auth.ts                isAuthenticated() via next/headers + re-exports
    ├── totp.ts, chat.ts, claude.ts, attachments.ts, skills.ts, workspace.ts, prisma.ts
```

When the fleet VS Code workspace gains or loses a folder, update `ENTRIES` in `src/app/fleet/page.tsx` too.

## Agent Controls

Every `/<team>/agents` page renders `TeamJobs` → `JobRow` rows; a row expands into
`AgentControls` (sleep/wake, run-now, model, effort, daily cost cap, notify, last 5 runs).
`TeamPauseControl` toggles `Workspace.pausedUntil`. Global vacation mode and concurrency live on
`/settings`. Semantics of the hot-reload live in vegapunk (see `vegapunk-assistant`).

## Workspace helper

Resolve workspace IDs via `getWorkspaceIdBySlug()` / `isWorkspaceSlug()` from `src/lib/workspace.ts`
(`WORKSPACE_SLUGS = ["gomuos", "kidsapp", "vegalab"]`). Adding a workspace: append the slug there, insert a
`lab_workspaces` row, add a Sidebar group and pages, and a `<team>-team/` skill folder.

## Tailwind Theme Tokens

`lab-bg` (page), `lab-surface` (cards), `lab-border`, `lab-muted`, `lab-text`, `lab-dim`,
`lab-accent` / `lab-accent-hover` — defined in `tailwind.config.ts`.

## Deployment

- **Image**: `node:22-bookworm-slim` with `@anthropic-ai/claude-code` installed globally.
  Node 22 is required — on Node 20 npm silently installs Claude Code 2.1.197, which can't run
  current models. `HOME=/data/home`, `DATA_DIR=/data`.
- **CI**: every push to `main` → `ghcr.io/shpatjakupi/gomuos-lab:latest`.
- **Pod**: `imagePullPolicy: Always`, limits 1536Mi / 1 CPU (the chat runs Claude in the pod).
- **Secrets**: `gomuos-lab-secret` (DATABASE_URL, LAB_PASSWORD, LAB_API_KEY, GITHUB_TOKEN — in
  infra-gitops) and `gomuos-lab-auth` (SESSION_SECRET, TOTP_SECRET, CLAUDE_CODE_OAUTH_TOKEN —
  **not in git**, created with kubectl). When the Vegapunk token is renewed, update `gomuos-lab-auth` too.

```bash
# After CI is green:
ssh root@46.224.215.213 "kubectl rollout restart deployment/gomuos-lab -n gomuos && kubectl rollout status deployment/gomuos-lab -n gomuos --timeout=240s"
```

## Schema Migrations

The Prisma CLI does not run in the container. Keep a SQL file in `prisma/migrations/NNN_name.sql`
and apply it by hand. The DB password contains URL-encoded characters, so parse DATABASE_URL properly:

```bash
scp prisma/migrations/0NN_name.sql root@46.224.215.213:/tmp/m.sql
ssh root@46.224.215.213 'URL=$(kubectl get secret gomuos-lab-secret -n gomuos -o jsonpath="{.data.DATABASE_URL}" | base64 -d)
eval $(python3 -c "import sys,shlex,urllib.parse as u;p=u.urlparse(sys.argv[1]);print(\"U=%s P=%s DB=%s\" % (p.username, shlex.quote(u.unquote(p.password)), p.path.lstrip(\"/\")))" "$URL")
kubectl exec -i mysql-0 -n gomuos -- env MYSQL_PWD="$P" mysql -u "$U" "$DB" < /tmp/m.sql'
```

Order:
- **Adding** tables/columns: apply SQL first, then deploy the code that uses them.
- **Dropping** columns/tables: deploy code that no longer uses them first, then apply SQL
  (the old client selects every column and breaks if one disappears).
- New NOT NULL column on an existing table: add as NULL → backfill → MODIFY NOT NULL → FK last.

Take a backup before destructive changes: `mysqldump --single-transaction --no-tablespaces` into
`/root/backups/` (see 2026-11-10 `gomuos_lab-before-fleet-cleanup-*.sql.gz`).

## Known Gotchas

- **`dynamic = "force-dynamic"`** on every page that reads the DB.
- **`searchParams` / `params` are Promises** in this Next.js setup — always `await` them.
- **Stale `.next/types`** after deleting pages makes `tsc` fail — `rm -rf .next` and rebuild.
- **Workspace defaulting**: write endpoints default `workspace` to `"gomuos"`; Bumble agents must send `"workspace": "kidsapp"`.
- **Rollout window**: right after a rollout the old pod can answer 504 for a few seconds while terminating.
