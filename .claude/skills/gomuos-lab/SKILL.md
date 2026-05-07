---
name: gomuos-lab
description: GomuOS Lab — the human approval dashboard at lab.gomuos.com. Hosts two workspaces (gomuos for the food platform, strawhats for the innovation team that builds new apps from wishes). Architecture, Prisma schema, API routes, comment system, ticket status flow, project lifecycle, UI components, deployment, and known gotchas. Use when building or debugging the Lab itself (not when working with agent skills or the order app).
---

# GomuOS Lab

Human approval dashboard for the GomuOS agent system. Agents create tickets; the human reviews, comments, and approves or rejects. Lives at `lab.gomuos.com`.

## Tech Stack

| | |
|--|--|
| Framework | Next.js 14 (App Router), TypeScript |
| Database | MySQL via Prisma ORM |
| Styling | Tailwind CSS with custom `lab-*` theme tokens |
| Auth | Simple password + API key (no OAuth) |
| Hosting | K8s, namespace `gomuos`, server `46.224.215.213` |

**Local path:** `C:\Users\shpat\Desktop\projects\gomuos-lab`
**GitHub:** `shpatjakupi/gomuos-lab`
**K8s manifest:** `infra-gitops/apps/lab/`

## Workspaces — Two Teams Live Here

The Lab hosts two independent agent teams as **workspaces**. Every Ticket and Job belongs to exactly one workspace; the UI, sidebar nav, and most APIs filter by it.

| Slug | Team | Purpose | UI |
|------|------|---------|-----|
| `gomuos` | GomuOS team (hunters, devs, validators) | Maintains the food-ordering platform | `/tickets`, `/hunters`, `/jobs` |
| `strawhats` | Straw Hats team (intake, architect, devops, devs, qa) | Builds new full-stack apps from human wishes | `/strawhats`, `/strawhats/[id]` |

Strawhats additionally has a `Project` table (one per app/wish) that owns multiple tickets and project-level comments.

## Prisma Schema

```prisma
model Workspace {
  id          Int       @id @default(autoincrement())
  slug        String    @unique // 'gomuos' | 'strawhats'
  name        String
  description String?
  createdAt   DateTime  @default(now())
  updatedAt   DateTime  @updatedAt
  tickets     Ticket[]
  jobs        Job[]
  projects    Project[]
  @@map("lab_workspaces")
}

model Job {
  id          Int        @id @default(autoincrement())
  workspaceId Int        // → Workspace
  workspace   Workspace
  name        String     @unique
  description String
  schedule    String
  enabled     Boolean    @default(true)
  lastRunAt   DateTime?
  lastStatus  String?    // idle | running | success | error
  lastOutput  String?    @db.LongText
  createdAt   DateTime   @default(now())
  updatedAt   DateTime   @updatedAt
  tickets     Ticket[]
  @@index([workspaceId])
  @@map("lab_jobs")
}

model Ticket {
  id            Int         @id @default(autoincrement())
  workspaceId   Int         // → Workspace (always required)
  workspace     Workspace
  projectId     Int?        // → Project (strawhats only)
  project       Project?
  jobId         Int?
  job           Job?
  title         String
  body          String      @db.LongText
  severity      String      @default("info")    // info | warning | error | critical
  status        String      @default("pending") // see status flow below
  assignedAgent String?
  delegatedBy   String?
  createdAt     DateTime    @default(now())
  updatedAt     DateTime    @updatedAt
  resolvedAt    DateTime?
  executions    Execution[]
  comments      Comment[]
  @@index([workspaceId])
  @@index([projectId])
  @@map("lab_tickets")
}

model Comment {
  id        Int      @id @default(autoincrement())
  ticketId  Int?     // either ticketId OR projectId must be set
  ticket    Ticket?
  projectId Int?     // strawhats only — comments directly on a Project (intake Q&A)
  project   Project?
  author    String   // "human" or agent name e.g. "gomuos-frontend-developer"
  body      String   @db.LongText
  createdAt DateTime @default(now())
  @@index([ticketId])
  @@index([projectId])
  @@map("lab_comments")
}

model Execution {
  id         Int       @id @default(autoincrement())
  ticketId   Int
  status     String    @default("running") // running | success | failed
  log        String?   @db.LongText
  startedAt  DateTime  @default(now())
  finishedAt DateTime?
  @@map("lab_executions")
}

model Project {
  id            Int       @id @default(autoincrement())
  workspaceId   Int       // always 'strawhats' for now
  workspace     Workspace
  slug          String    @unique // url-safe, also used as folder + repo name
  name          String
  wish          String    @db.LongText // raw human wish
  spec          String?   @db.LongText // refined by strawhat-product-intake
  architecture  String?   @db.LongText // designed by strawhat-architect
  techStack     String?
  status        String    @default("intake") // intake|spec|design|building|testing|deployed|archived
  repoUrl       String?
  repoPath      String?   // local cwd e.g. /home/vegapunk/projects/<slug>
  deployUrl     String?
  stagingUrl    String?
  createdAt     DateTime  @default(now())
  updatedAt     DateTime  @updatedAt
  archivedAt    DateTime?
  tickets       Ticket[]
  comments      Comment[]
  @@index([workspaceId])
  @@map("lab_projects")
}
```

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

## Project Status Flow (strawhats only)

```
intake ──→ spec ──→ design ──→ building ──→ testing ──→ deployed
                                                            ↓
                                                        archived
```

| Status | Set by | Meaning |
|--------|--------|---------|
| `intake` | initial create OR `askHuman` comment | waiting on human input or intake agent |
| `spec` | strawhat-product-intake | spec ready for architect |
| `design` | strawhat-architect | architecture ready for devops |
| `building` | strawhat-devops | repo bootstrapped, devs implementing |
| `testing` | dev (after deploy) | qa-tester validating |
| `deployed` | strawhat-qa-tester | live |
| `archived` | human/manager | abandoned / replaced |

## API Routes

All write endpoints require `Authorization: Bearer $LAB_API_KEY`.

### Shared (workspace-scoped via `?workspace=` param)

| Method | Path | Notes |
|--------|------|-------|
| `GET` | `/api/tickets` | `?status=` and `?workspace=` and `?projectId=` filters. No auth required (web UI uses it) |
| `POST` | `/api/tickets` | Create ticket. Body: `{ title, description, severity, jobName?, assignedAgent?, delegatedBy?, workspace?, projectId? }`. `workspace` defaults to `"gomuos"` if omitted. |
| `GET` | `/api/tickets/:id` | Includes `job`, `executions`, `comments` |
| `PATCH` | `/api/tickets/:id` | Update `status`, `assignedAgent`. Also accepts `executionLog`+`executionStatus` to create an Execution row |
| `GET` | `/api/tickets/:id/poll` | Returns `{ id, status }` — used by Vegapunk to wait for approval |
| `GET` | `/api/tickets/:id/comments` | Fetch comments ordered by `createdAt asc` |
| `POST` | `/api/tickets/:id/comments` | Create comment. Body: `{ author, body, askAgent? }`. If `askAgent: true`, sets ticket status to `needs_response` |
| `GET` | `/api/jobs` | List all jobs. `?workspace=` filter. |
| `POST` | `/api/jobs` | Upsert job. Create: `{ name, description, schedule, workspace? }`. Update: `{ name, lastStatus, lastOutput? }`. `workspace` defaults to `"gomuos"`. |

### Strawhats-only

| Method | Path | Notes |
|--------|------|-------|
| `GET` | `/api/strawhats/projects` | List all strawhat projects. `?status=` filter. Includes `_count` of tickets and comments. |
| `POST` | `/api/strawhats/projects` | Create project from a wish. Body: `{ name, wish, source? }`. `source: "api"` requires API key (default web form is open). Auto-creates the first ticket assigned to `strawhat-product-intake`. |
| `GET` | `/api/strawhats/projects/:id` | Project detail with all tickets and comments. |
| `PATCH` | `/api/strawhats/projects/:id` | Update `name`, `spec`, `architecture`, `techStack`, `status`, `repoUrl`, `repoPath`, `deployUrl`, `stagingUrl`. Setting `status: "archived"` sets `archivedAt`. |
| `GET` | `/api/strawhats/projects/:id/comments` | Project-level comments (intake Q&A) ordered by `createdAt asc`. |
| `POST` | `/api/strawhats/projects/:id/comments` | Create comment. Body: `{ author, body, askHuman? }`. If `askHuman: true`, sets project status to `intake`. |

## Comment System

There are **two** comment threads, depending on context:

**Ticket-level** (`POST /api/tickets/:id/comments`):
- Used for "Spørg agent" — human asks a clarifying question on a specific ticket before approving
- `askAgent: true` flips ticket to `needs_response`; dispatcher spawns agent in respond-only mode; agent posts a reply and flips back to `pending`
- UI: `src/components/CommentThread.tsx`

**Project-level** (`POST /api/strawhats/projects/:id/comments`):
- Used during intake Q&A (before tickets exist for the next phase)
- `strawhat-product-intake` posts clarifying questions with `askHuman: true` → project status flips to `intake` so the wishes UI shows it as "waiting on you"
- Human answers via the project detail page
- UI: `src/components/ProjectCommentThread.tsx`

In both threads, `author === "human"` → grey bubble (left-aligned); anything else → blue accent bubble (right-aligned).

## UI Structure

```
src/
├── app/
│   ├── login/                 page.tsx
│   ├── dashboard/             page.tsx
│   ├── tickets/
│   │   ├── page.tsx           list with status tabs (gomuos workspace)
│   │   └── [id]/page.tsx      detail with executions + CommentThread
│   ├── hunters/               page.tsx (gomuos)
│   ├── jobs/                  page.tsx (gomuos)
│   ├── strawhats/
│   │   ├── page.tsx           project list with status tabs + "Nyt ønske" button
│   │   └── [id]/page.tsx      detail: wish, spec, architecture, intake comments, tickets
│   └── api/                   (see routes above)
├── components/
│   ├── Sidebar.tsx            grouped nav (GomuOS / Straw Hats)
│   ├── StatusBadge.tsx        SeverityBadge | StatusBadge | JobStatusBadge
│   ├── ProjectStatusBadge.tsx ProjectStatusBadge | ProjectProgress (5-step bar)
│   ├── TicketActions.tsx      Approve/Reject buttons (only on pending)
│   ├── CommentThread.tsx      ticket comment Q&A
│   ├── ProjectCommentThread.tsx  project intake Q&A
│   ├── NewWishForm.tsx        client form on /strawhats
│   └── HunterCard.tsx, HunterFeedbackForm.tsx
└── lib/
    ├── prisma.ts
    ├── auth.ts
    ├── workspace.ts           getWorkspaceIdBySlug() with cache
    └── slugify.ts             slug-safe (handles æøå)
```

## Workspace helper

When writing API routes that touch tickets/jobs, always resolve the workspace ID via the helper — never look it up by slug inline:

```typescript
import { getWorkspaceIdBySlug, isWorkspaceSlug } from "@/lib/workspace";

const slug = isWorkspaceSlug(req.searchParams.get("workspace")) ? ... : "gomuos";
const workspaceId = await getWorkspaceIdBySlug(slug);
```

The helper caches the slug→id mapping in memory (workspaces are seeded once and never deleted).

## Adding a New Workspace

To onboard a third (or fourth) team — e.g. `kidsapp`, `noteapp`, etc. — touch four places:

1. **Code**: append the slug to `WORKSPACE_SLUGS` in `src/lib/workspace.ts`.
2. **DB**: insert a row in `lab_workspaces`. From the VPS:
   ```bash
   PWD=$(kubectl get secret gomuos-lab-secret -n gomuos -o jsonpath='{.data.DATABASE_URL}' | base64 -d | sed 's/.*://;s/@.*//')
   kubectl exec mysql-0 -n gomuos -- mysql -u admin "-p$PWD" gomuos_lab -e "
     INSERT INTO lab_workspaces (slug, name, description) VALUES ('<slug>', '<Display Name>', '<one-line>');
   "
   ```
3. **UI nav**: add an entry in `src/components/Sidebar.tsx` under a new `group:` so it appears in its own section. Reuse `/tickets?workspace=<slug>` if the workspace doesn't need a dedicated detail UI; otherwise build `/<slug>/page.tsx`.
4. **Agents**: usually a new workspace gets its own team — drop a `<slug>-team/` skill folder in `.dotfiles/.claude/skills/` and wire a `<slug>-manager` cron job in `vegapunk/src/infra/cron.ts`.

A workspace can exist without a dedicated UI page — `gomuos` and `kidsapp` reuse `/tickets` filtered by workspace, while `strawhats` has its own `/strawhats` UI because it owns Projects.

## Tailwind Theme Tokens

Defined in `tailwind.config.ts`:

| Token | Usage |
|-------|-------|
| `lab-bg` | Page background (darkest) |
| `lab-surface` | Cards and panels |
| `lab-border` | Borders |
| `lab-muted` | Muted background (sidebar hover, progress bar empty state) |
| `lab-text` | Primary text |
| `lab-dim` | Secondary/muted text |
| `lab-accent` | Brand accent (blue) |

## Deployment

**GitHub Actions** builds Docker image on every push to `main` → `ghcr.io/shpatjakupi/gomuos-lab:latest`.

**K8s** deployment in `infra-gitops/apps/lab/deployment.yaml` — `imagePullPolicy: Always` so pod restart always pulls fresh image.

**To deploy a change:**
```bash
# 1. Push to GitHub → Actions builds image (~2 min)
# 2. Restart pod to pull new image:
ssh root@46.224.215.213
kubectl rollout restart deployment/gomuos-lab -n gomuos
```

## Schema Migrations

The project uses `prisma db push` (no Prisma migration files, but we keep manual SQL files in `prisma/migrations/` for record). **The Prisma CLI cannot run inside the Docker container** (WASM binary copy issue). Schema changes must be applied manually:

```bash
# Get DATABASE_URL from secret
kubectl get secret gomuos-lab-secret -n gomuos -o jsonpath='{.data.DATABASE_URL}' | base64 -d

# For multi-statement migrations, copy SQL up and pipe in:
scp prisma/migrations/00X_name.sql root@46.224.215.213:/tmp/
ssh root@46.224.215.213
PWD=$(kubectl get secret gomuos-lab-secret -n gomuos -o jsonpath='{.data.DATABASE_URL}' | base64 -d | sed 's/.*://;s/@.*//')
kubectl exec -i mysql-0 -n gomuos -- mysql -u admin -p"$PWD" gomuos_lab < /tmp/00X_name.sql

# For single-statement changes:
kubectl exec mysql-0 -n gomuos -- mysql -u admin '-p<password>' gomuos_lab -e "ALTER TABLE ... ;"
```

After applying the SQL, **rebuild the image** (which regenerates the Prisma client with the new schema) and **restart the pod**. Both steps are required — the running pod has the old client.

**Recommended order** (proven during the strawhats migration):

1. Push schema + code changes to GitHub (kicks off CI build, ~2 min)
2. Wait for CI green (`gh run list --limit 1`)
3. Apply migration SQL via the scp + kubectl pipe above
4. Verify with a SELECT or SHOW TABLES on the new schema
5. `kubectl rollout restart deployment/gomuos-lab -n gomuos && kubectl rollout status deployment/gomuos-lab -n gomuos --timeout=180s`
6. Smoke-test a new endpoint with `curl https://lab.gomuos.com/api/...`

**For NOT NULL columns on existing tables**, follow the safe-add pattern (see `prisma/migrations/001_workspaces_and_projects.sql` for a full example):
```sql
ALTER TABLE x ADD COLUMN newcol INT NULL;        -- 1. add as NULL
UPDATE x SET newcol = ... WHERE newcol IS NULL;  -- 2. backfill
ALTER TABLE x MODIFY COLUMN newcol INT NOT NULL; -- 3. promote
ALTER TABLE x ADD CONSTRAINT ... FOREIGN KEY ... ;  -- 4. FK last
```
This minimizes the risk-window where the OLD pod could insert a row and fail the new constraint.

When introducing a new required column on an existing table:
1. Add the column as `NULL` first
2. Backfill existing rows
3. `MODIFY COLUMN ... NOT NULL`
4. Add the FK constraint last

This pattern is what `prisma/migrations/001_workspaces_and_projects.sql` follows for `workspaceId`.

## Known Gotchas

- **Prisma CLI WASM crash**: Copying `node_modules/.bin/prisma` to the Docker runner stage fails because the binary references `prisma_schema_build_bg.wasm` by relative path. Don't try to run `prisma db push` at container startup — apply migrations manually via SQL.
- **`dynamic = "force-dynamic"`**: All pages that read from DB have this export to prevent Next.js from caching at build time.
- **`searchParams` is a Promise in Next.js 14**: Always `await searchParams` before accessing properties. Same for `params` in dynamic routes.
- **Comment author field**: `"human"` for the human, exact agent name (e.g. `"gomuos-frontend-developer"` or `"strawhat-product-intake"`) for agents. The UI uses `author === "human"` to determine bubble alignment.
- **Workspace defaulting**: All write endpoints default `workspace` to `"gomuos"` for backward compatibility. Strawhat agents must explicitly include `"workspace": "strawhats"` (and `projectId`) in every ticket they create.
- **Comment.ticketId is now nullable**: Either `ticketId` or `projectId` must be set, but not enforced at the DB level — application code must guarantee this. The two POST routes do.
- **Project slug uniqueness**: Auto-generated from name via `slugify()`; collisions get `-2`, `-3`, etc. appended. The slug also becomes the GitHub repo name and the local folder name on VPS, so it must be DNS- and shell-safe.
