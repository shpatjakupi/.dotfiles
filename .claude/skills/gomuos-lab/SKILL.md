---
name: gomuos-lab
description: GomuOS Lab — the human approval dashboard at lab.gomuos.com. Architecture, Prisma schema, API routes, comment system, ticket status flow, UI components, deployment, and known gotchas. Use when building or debugging the Lab itself (not when working with agent skills or the order app).
---

# GomuOS Lab

Human approval dashboard for the GomuOS agent team. Agents create tickets; the human reviews, comments, and approves or rejects. Lives at `lab.gomuos.com`.

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

## Prisma Schema

```prisma
model Job {
  id          Int       @id @default(autoincrement())
  name        String    @unique
  description String
  schedule    String
  enabled     Boolean   @default(true)
  lastRunAt   DateTime?
  lastStatus  String?   // idle | running | success | error
  lastOutput  String?   @db.LongText
  createdAt   DateTime  @default(now())
  updatedAt   DateTime  @updatedAt
  tickets     Ticket[]
  @@map("lab_jobs")
}

model Ticket {
  id            Int         @id @default(autoincrement())
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
  @@map("lab_tickets")
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

model Comment {
  id        Int      @id @default(autoincrement())
  ticketId  Int
  author    String   // "human" or agent name e.g. "gomuos-frontend-developer"
  body      String   @db.LongText
  createdAt DateTime @default(now())
  @@map("lab_comments")
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

## API Routes

All write endpoints require `Authorization: Bearer $LAB_API_KEY`.

| Method | Path | Notes |
|--------|------|-------|
| `GET` | `/api/tickets` | `?status=` filter. No auth required (web UI uses it) |
| `POST` | `/api/tickets` | Create ticket. Body: `{ title, description, severity, jobName?, assignedAgent?, delegatedBy? }` |
| `GET` | `/api/tickets/:id` | Includes `job`, `executions`, `comments` |
| `PATCH` | `/api/tickets/:id` | Update `status`, `assignedAgent`. Also accepts `executionLog`+`executionStatus` to create an Execution row |
| `GET` | `/api/tickets/:id/poll` | Returns `{ id, status }` — used by Vegapunk to wait for approval |
| `GET` | `/api/tickets/:id/comments` | Fetch comments ordered by `createdAt asc` |
| `POST` | `/api/tickets/:id/comments` | Create comment. Body: `{ author, body, askAgent? }`. If `askAgent: true`, sets ticket status to `needs_response` |
| `GET` | `/api/jobs` | List all jobs |
| `POST` | `/api/jobs` | Upsert job. Create: `{ name, description, schedule }`. Update: `{ name, lastStatus, lastOutput? }` |

## Comment System ("Spørg agent")

The human can post questions on any ticket and ask the assigned agent to respond before approving.

**Flow:**
1. Human writes a question, clicks **"Spørg agent"**
2. `POST /api/tickets/:id/comments { author: "human", body: "...", askAgent: true }` → status becomes `needs_response`
3. Vegapunk dispatcher (every 3 min) polls `needs_response` tickets
4. Dispatcher spawns agent in respond-only mode: agent reads all comments, posts a reply via `POST /api/tickets/:id/comments { author: "<agentName>", body: "..." }`, then sets status back to `pending`
5. Human sees the reply in the thread and decides to approve, reject, or ask again

**UI:** `src/components/CommentThread.tsx` — human comments left-aligned (grey), agent comments right-aligned (accent). Always shown on ticket detail page.

## UI Structure

```
src/
├── app/
│   ├── login/           page.tsx
│   ├── dashboard/       page.tsx
│   ├── tickets/
│   │   ├── page.tsx     list with tabs (Alle / Afventer / Spørgsmål / Godkendt / I gang / Udført / Afvist)
│   │   └── [id]/page.tsx  detail with executions + CommentThread
│   └── api/             (see routes above)
├── components/
│   ├── Sidebar.tsx
│   ├── StatusBadge.tsx  SeverityBadge | StatusBadge | JobStatusBadge
│   ├── TicketActions.tsx  Approve/Reject buttons (only shown on pending)
│   └── CommentThread.tsx  comment Q&A (client component)
└── lib/
    ├── prisma.ts
    └── auth.ts
```

## Tailwind Theme Tokens

Defined in `tailwind.config.ts`:

| Token | Usage |
|-------|-------|
| `lab-bg` | Page background (darkest) |
| `lab-surface` | Cards and panels |
| `lab-border` | Borders |
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

The project uses `prisma db push` (no migration files). **The Prisma CLI cannot run inside the Docker container** (WASM binary copy issue). Schema changes must be applied manually:

```bash
# Get DATABASE_URL from secret
kubectl get secret gomuos-lab-secret -n gomuos -o jsonpath='{.data.DATABASE_URL}' | base64 -d

# Apply SQL directly
kubectl exec mysql-0 -n gomuos -- mysql -u admin '-p<password>' gomuos_lab -e "
  ALTER TABLE lab_tickets ADD COLUMN new_col VARCHAR(191);
  -- or CREATE TABLE, etc.
"
```

After applying the SQL, rebuild the image (which regenerates the Prisma client with the new schema) and restart the pod.

## Known Gotchas

- **Prisma CLI WASM crash**: Copying `node_modules/.bin/prisma` to the Docker runner stage fails because the binary references `prisma_schema_build_bg.wasm` by relative path. Don't try to run `prisma db push` at container startup — apply migrations manually via SQL.
- **`dynamic = "force-dynamic"`**: All pages that read from DB have this export to prevent Next.js from caching at build time.
- **`searchParams` is a Promise in Next.js 14**: Always `await searchParams` before accessing properties.
- **Comment author field**: `"human"` for the human, exact agent name (e.g. `"gomuos-frontend-developer"`) for agents. The UI uses `author === "human"` to determine bubble alignment.
