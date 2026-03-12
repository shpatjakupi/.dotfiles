---
name: gomuos-infra
description: Infrastructure knowledge for the Gomu Order System (GomuOS) - a multi-tenant food ordering platform. Use this skill when the user wants to work on server infrastructure, Kubernetes deployments, ArgoCD GitOps, database management, DNS/SSL setup, or CI/CD pipelines for their self-hosted k3s setup on Hetzner. Triggers when user mentions: server, k3s, ArgoCD, Hetzner, infra-gitops, gomuos, ordrupspizza deployment, MySQL on the server, pods, deployments, or wants to SSH into the server.
---

# GomuOS Infrastructure

Full infrastructure context for the Gomu Order System. Read `references/setup.md` for complete server/architecture details and `references/commands.md` for common operations.

## Quick Access

**SSH into server:**
```bash
ssh root@46.224.215.213
```

**Key repos:**
- Infrastructure (GitOps): `C:\Users\shpat\Desktop\projects\infra-gitops` → github.com/shpatjakupi/infra-gitops
- Backend: `C:\Users\shpat\Desktop\projects\order-backend` → github.com/shpatjakupi/order-backend
- Frontend: `C:\Users\shpat\Desktop\projects\next-app-template` → github.com/shpatjakupi/next-app-template

## How Changes Are Deployed

1. Edit K8s manifests in `infra-gitops` → push → ArgoCD auto-syncs
2. Edit backend/frontend code → push → GitHub Actions builds Docker image → delete pod in ArgoCD UI (or `kubectl rollout restart`) to pull new `:latest` image

## When to Read References

- **Server details, architecture, credentials**: Read `references/setup.md`
- **kubectl/git commands for daily ops**: Read `references/commands.md`
