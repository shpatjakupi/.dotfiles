---
name: order-app
description: >
  Full-stack context for the GomuOS food ordering platform (order-backend + next-app-template + infra-gitops).
  Use this skill when working on backend features, frontend UI, cross-project changes, debugging production,
  or navigating the deployed system. Triggers on: order-backend, next-app-template, food app, ordrupspizza,
  menu, orders, checkout, admin dashboard, Spring Boot backend, Next.js frontend, or any full-stack food app work.
  For deep infrastructure changes (K8s manifests, cert-manager, ArgoCD config), prefer the gomuos-infra skill.
---

# GomuOS Order App

Multi-tenant food ordering platform. One backend, one frontend, per-restaurant deployments on k3s.

## Projects

| Project | Local path | VPS path | GitHub | Tech |
|---------|-----------|----------|--------|------|
| **order-backend** | `C:\Users\shpat\Desktop\projects\order-backend` | `/home/vegapunk/projects/order-backend` | `shpatjakupi/order-backend` | Java 17, Spring Boot 3.1, Hibernate, MySQL |
| **next-app-template** | `C:\Users\shpat\Desktop\projects\next-app-template` | `/home/vegapunk/projects/next-app-template` | `shpatjakupi/next-app-template` | Next.js 14 (App Router), TypeScript, Mantine UI 7 |
| **infra-gitops** | `C:\Users\shpat\Desktop\projects\infra-gitops` | `/home/vegapunk/projects/infra-gitops` | `shpatjakupi/infra-gitops` | K8s manifests, ArgoCD apps |

## How the System Connects

```
Browser/Phone
    |
    v
Traefik Ingress (ordrupspizza.dk)
    |
    ├── /           --> Frontend (Next.js, port 3000)
    ├── /ws         --> Backend  (Spring Boot, port 8080) — WebSocket/SockJS
    └── /images     --> Backend  (serves food images from PVC)

Frontend internally:
    /app/api/*  --> proxy routes --> Backend ClusterIP (http://ordrupspizza-backend:8080)
    /services/* --> call /app/api/* routes

Backend internally:
    Spring Boot --> MySQL (mysql.gomuos.svc.cluster.local:3306)
```

**Key insight**: Frontend never calls backend directly from the browser. All API calls go through Next.js API routes (`/app/api/`) which proxy server-side to the backend ClusterIP. This avoids CORS issues and keeps the backend internal.

## Non-Obvious Gotchas

- **`APP_DOMAIN`** env var in backend must be `ordrupspizza` (no `.dk`). CorsFilter and WebSocketConfig append `.dk` themselves. Setting `ordrupspizza.dk` causes double `.dk`.
- **`NEXT_PUBLIC_*`** vars are baked at Docker **build time**, not runtime. Change them in `.github/workflows/build.yml` build-args, not in K8s secrets.
- **Customer auth** is an in-memory Spring Security user created at boot from `CUSTOMER_USERNAME`/`CUSTOMER_PASSWORD` env vars. Not in the database.
- **`order` table** is a SQL reserved word — requires backticks in raw queries.
- **Firefox 400 errors** — Firefox sends `DNT` and `TE` headers. These must be in `StrictHttpFirewall` allowed list in `Config.java`.
- **Blocked IPs** — backend auto-blocks suspicious IPs. Check `blocked_ips` table if customers report 403s.
- **Spring profile**: `prod,ordrupspizza` — restaurant-specific config in `application-ordrupspizza.properties`.

## Cross-Project Feature Workflow

Adding a new feature end-to-end:

1. **Backend**: Entity → DTO → DAO → Service → Controller endpoint
2. **Frontend**: TypeScript interface → Service function → API route (`/app/api/`) → Component
3. **Infra** (if new env vars needed): Update secret in `infra-gitops/restaurants/ordrupspizza/backend/secret.yaml`

## Quick Ops (SSH + kubectl)

```bash
# SSH into VPS
ssh root@46.224.215.213

# Check pods
kubectl get pods -n gomuos

# Backend logs
kubectl logs -l app=ordrupspizza-backend -n gomuos --tail=100

# Frontend logs
kubectl logs -l app=ordrupspizza-frontend -n gomuos --tail=100

# Restart pods (pull latest image)
kubectl rollout restart deployment/ordrupspizza-backend -n gomuos
kubectl rollout restart deployment/ordrupspizza-frontend -n gomuos

# MySQL shell
kubectl exec -it mysql-0 -n gomuos -- mysql -u admin -p ordrupspizza

# Recent orders
kubectl exec mysql-0 -n gomuos -- mysql -u admin -p -e "SELECT Id, Ordered_date, Is_payment_successful FROM \`order\` ORDER BY Id DESC LIMIT 10;" ordrupspizza
```

For deep infra work (K8s manifests, cert-manager, ArgoCD, DNS), use the **gomuos-infra** skill.

## When to Read References

- **Backend architecture, entities, package structure**: Read `references/backend.md`
- **Frontend architecture, components, services**: Read `references/frontend.md`
