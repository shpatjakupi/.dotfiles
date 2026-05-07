---
name: strawhat-team
description: Complete overview of the Straw Hats agent team — innovation crew that builds full-stack apps from human wishes. Structure, roles, project lifecycle from intake to deployed, Lab API for /strawhats, and how to add or modify agents. Read this first when working with or modifying the Straw Hats system.
---

# Straw Hats Agent Team

A team of specialized Claude Code agents that turn human wishes into deployed full-stack apps. The human writes a wish in `lab.gomuos.com/strawhats` and the team takes it from idea → spec → architecture → code → tested → deployed. Every step requires human approval via lab tickets.

This is **separate from the GomuOS team** — GomuOS maintains the existing food-ordering platform; Straw Hats builds new apps from scratch.

## Team Structure

```
strawhat-team/
├── captain/      ← orchestrator, routes work between phases
│   └── strawhat-manager
├── intake/       ← reads the wish, asks clarifying questions, writes the spec
│   └── strawhat-product-intake
├── design/       ← chooses tech stack, designs system, writes ARCHITECTURE.md
│   └── strawhat-architect
├── devs/         ← implementers
│   ├── strawhat-backend-developer    (Spring Boot 3.1, Java 17, MySQL)
│   └── strawhat-frontend-developer   (Next.js 14, TypeScript, Mantine UI 7)
├── ops/          ← repo creation, Docker, k3s, ArgoCD
│   └── strawhat-devops
└── qa/           ← Playwright E2E + code review
    └── strawhat-qa-tester
```

## Project Lifecycle

```
[Human writes wish in /strawhats UI]
   ↓
status='intake' — Project created, auto-ticket → strawhat-product-intake
   ↓ (intake reads wish, posts clarifying questions as project-comments)
   ↓ (human answers via /strawhats/[id])
   ↓ (intake refines until it has enough → writes spec, PATCH project)
status='spec'
   ↓ (manager creates ticket → strawhat-architect)
   ↓ (architect designs system, picks tech, writes architecture, PATCH project)
status='design'
   ↓ (manager creates ticket → strawhat-devops)
   ↓ (devops creates folder + GitHub repo + Dockerfile + k8s manifest +
   ↓  ArgoCD app + commits SPEC.md/ARCHITECTURE.md, PATCH repoUrl/repoPath)
status='building'
   ↓ (manager creates parallel tickets → backend-dev + frontend-dev)
   ↓ (devs implement, push to GitHub, deploy to staging via ArgoCD)
status='testing'
   ↓ (manager creates ticket → strawhat-qa-tester)
   ↓ (qa runs Playwright + code review, creates fix-tickets if needed)
status='deployed'
```

Status only advances when the **previous phase's ticket is marked done**. The manager runs after each completion to decide the next phase.

## What Each Agent Does

### strawhat-manager (captain)
- Polls `/api/strawhats/projects` and routes work based on `status`
- Creates the next-phase ticket when previous phase finishes
- Reads `~/.claude/skills/strawhat-team/GOALS.md` for active priorities
- Never writes code — orchestrates only

### strawhat-product-intake
- Reads the wish on a project (status='intake')
- Posts clarifying questions to `POST /api/strawhats/projects/{id}/comments` with `askHuman: true`
- When it has enough context, writes a SPEC and PATCHes the project: `{ spec: "...", status: "spec" }`
- The spec covers: user stories, key features, success criteria, non-goals, scope of v1

### strawhat-architect
- Reads project.spec
- Picks tech stack within the GomuOS family (default: Spring Boot + Next.js + MySQL + k3s)
- Writes architecture: data model, API surface, key components, deployment shape
- PATCHes project: `{ architecture: "...", techStack: "Spring Boot + Next.js", status: "design" }`

### strawhat-devops
- Creates `/home/vegapunk/projects/<project.slug>/` with the right structure
- Creates GitHub repo `shpatjakupi/<project.slug>`
- Writes Dockerfile, GitHub Actions workflow, k8s manifest in `infra-gitops/apps/<slug>/`, ArgoCD Application
- Commits SPEC.md + ARCHITECTURE.md to the new repo
- PATCHes project: `{ repoUrl, repoPath, stagingUrl, status: "building" }`

### strawhat-backend-developer
- Reads project.spec + project.architecture
- Implements backend in `<repoPath>/backend/` (Spring Boot)
- Commits + pushes; ArgoCD deploys to staging
- Reports back via ticket done

### strawhat-frontend-developer
- Reads project.spec + project.architecture
- Implements frontend in `<repoPath>/frontend/` (Next.js + Mantine)
- Commits + pushes
- Reports back via ticket done

### strawhat-qa-tester
- Once both devs have deployed to stagingUrl, writes Playwright E2E tests covering the spec's success criteria
- Reviews backend + frontend code for security and convention violations
- Creates fix-tickets back to devs if issues found
- When clean: PATCHes project `{ status: "deployed" }`

## Lab API — /strawhats endpoints

- **URL**: `https://lab.gomuos.com/api`
- **Auth**: `Authorization: Bearer $LAB_API_KEY`

| Endpoint | Use |
|----------|-----|
| `GET /api/strawhats/projects` | List all projects (`?status=` filter) |
| `GET /api/strawhats/projects/{id}` | Project detail with tickets and comments |
| `POST /api/strawhats/projects` | Create new project from wish (auto-spawns intake ticket) |
| `PATCH /api/strawhats/projects/{id}` | Update spec, architecture, status, repoUrl, deployUrl, etc. |
| `GET /api/strawhats/projects/{id}/comments` | Project-level comments (intake Q&A) |
| `POST /api/strawhats/projects/{id}/comments` | Post comment. `askHuman: true` flags it as waiting on human |
| `GET /api/tickets?workspace=strawhats&projectId={id}` | List tickets for a project |
| `POST /api/tickets` | Create ticket. Always include `workspace: "strawhats"` and `projectId` |

## Project status values

| Status | Set by | Meaning |
|--------|--------|---------|
| `intake` | initial create OR `askHuman` comment | waiting on human input |
| `spec` | strawhat-product-intake | spec ready for architect |
| `design` | strawhat-architect | architecture ready for devops |
| `building` | strawhat-devops | repo ready, devs are coding |
| `testing` | dev (after deploy) | qa-tester is validating |
| `deployed` | strawhat-qa-tester | live |
| `archived` | human/manager | abandoned / replaced |

## Tech Stack (default)

All Straw Hats projects use the GomuOS family by default:

| Layer | Tech |
|-------|------|
| Backend | Spring Boot 3.1, Java 17, Hibernate, REST APIs |
| Database | MySQL on the existing `mysql-0` instance (new schema per project) |
| Frontend | Next.js 14 (App Router), TypeScript, Mantine UI 7 |
| Container | Docker, single image per service |
| Orchestration | k3s on `46.224.215.213` (gomuos namespace, separate Deployment) |
| GitOps | ArgoCD app under `infra-gitops/apps/<slug>/` |
| Domain | `<slug>.gomuos.com` via cert-manager (HTTP-01, `letsencrypt-prod` ClusterIssuer) |

**Wildcard DNS is already in place:** `*.gomuos.com → 46.224.215.213` is configured at simply.com, so any new subdomain works automatically without manual DNS work. Strawhat-devops only needs to create the IngressRoute and Certificate — no `askHuman` step for DNS.

The architect can deviate from the tech stack only with explicit human approval (post in project comment thread with `askHuman: true`).

## Project File Layout

When `strawhat-devops` creates a project, the folder must look like:

```
~/Desktop/projects/<slug>/         (mirrored to /home/vegapunk/projects/<slug>/)
├── README.md
├── SPEC.md                         (synced from project.spec on create + when changed)
├── ARCHITECTURE.md                 (synced from project.architecture)
├── backend/                        (Spring Boot)
│   ├── pom.xml
│   ├── Dockerfile
│   └── src/
├── frontend/                       (Next.js)
│   ├── package.json
│   ├── Dockerfile
│   └── src/
└── .github/workflows/
    └── build.yml                   (builds both images on push)
```

Plus in `infra-gitops/`:
```
apps/<slug>/
├── deployment.yaml                 (backend + frontend Deployments)
├── service.yaml
├── ingress.yaml                    (Traefik route to <slug>.gomuos.com)
└── argocd-app.yaml
```

## Cron Integration

`strawhat-manager` runs as a cron job in `vegapunk/src/infra/cron.ts` (every 30 min) and routes pending work. Dev/intake/architect/devops/qa agents are triggered reactively by the existing `ticket-dispatcher` when a ticket is approved.

The dispatcher chooses `cwd` based on agent name:
- `strawhat-product-intake`, `strawhat-architect`, `strawhat-manager` → `/home/vegapunk/vegapunk` (no specific repo)
- `strawhat-devops`, `strawhat-backend-developer`, `strawhat-frontend-developer`, `strawhat-qa-tester` → `/home/vegapunk/projects/<project.slug>` (looked up via the ticket's `projectId`)

## Goals File

Active priorities live in `~/.claude/skills/strawhat-team/GOALS.md`. The manager reads this when deciding what to work on next.

## Skills Location

```
dotfiles: .dotfiles/.claude/skills/strawhat-team/
local:    ~/.claude/skills/  (installed flat by sync-skills)
```

## How to Add a New Agent

1. Create `dotfiles/.claude/skills/strawhat-team/<group>/<agent-name>/SKILL.md`
2. Update this file's "Team Structure" and "What Each Agent Does"
3. Update `strawhat-manager/SKILL.md` routing table
4. If reactive only — the dispatcher needs `cwd` mapping in `vegapunk/src/infra/cron.ts`
5. If runs on schedule — add a cron job in the same file
6. Commit + push, then run sync-skills on VPS

## How to Modify an Existing Agent

1. Edit the skill file in `dotfiles/.claude/skills/strawhat-team/<group>/<agent-name>/SKILL.md`
2. If routing or workflow changed → update `strawhat-manager/SKILL.md`
3. Commit + push → sync-skills on VPS
