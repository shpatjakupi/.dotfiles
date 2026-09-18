---
name: gomuos-infra
description: GomuOS-specific infrastructure knowledge — ordrupspizza.dk production deployment, infra-gitops repo structure, ArgoCD apps, multi-tenant restaurant setup, Spring backend config gotchas, Bambora payment flow, WebSocket/CORS config, and GomuOS troubleshooting. Use when working on GomuOS deployments, secrets, ingress routing, or debugging production issues. For general cluster/VPS knowledge use the vps-cluster skill.
---

# GomuOS Infrastructure

GomuOS-specific deployment and operations knowledge. For general k3s/Traefik/cert-manager/ArgoCD/firewall see the `vps-cluster` skill.

## Repos

| Repo | Local | VPS | Purpose |
|------|-------|-----|---------|
| infra-gitops | `C:\Users\shpat\Desktop\projects\infra-gitops` | `/home/vegapunk/projects/infra-gitops` | K8s manifests, ArgoCD apps |
| order-backend | `C:\Users\shpat\Desktop\projects\order-backend` | `/home/vegapunk/projects/order-backend` | Spring Boot backend |
| next-app-template | `C:\Users\shpat\Desktop\projects\next-app-template` | `/home/vegapunk/projects/next-app-template` | Next.js frontend |

## infra-gitops Structure

```
infra-gitops/
├── base/                          # Shared cluster resources (ArgoCD app: gomuos-base)
│   ├── namespace.yaml
│   ├── ghcr-secret.yaml
│   ├── images-pvc.yaml            # 5Gi PVC for food images (mounted at /data/images)
│   └── mysql/                     # MySQL StatefulSet, services, secret
├── restaurants/
│   └── ordrupspizza/              # Production (ArgoCD app: ordrupspizza)
│       ├── backend/               # deployment, service, secret
│       ├── frontend/              # deployment, service, secret
│       └── ingress/               # ingress.yaml (ordrupspizza.dk TLS)
├── apps/
│   ├── lab/                       # lab.gomuos.com (ArgoCD app: lab)
│   └── testapp/                   # testapp.gomuos.com (ArgoCD app: testapp)
└── argocd/
    ├── base.yaml                  # gomuos-base app
    ├── ordrupspizza.yaml
    ├── lab.yaml
    └── testapp.yaml
```

**Deploy changes**: `git add -A && git commit -m "..." && git push` → ArgoCD picks up within ~3 min.

## ArgoCD Apps

| App | Watches | Namespace |
|-----|---------|-----------|
| `gomuos-base` | `base/` | `gomuos` |
| `ordrupspizza` | `restaurants/ordrupspizza/` | `gomuos` |
| `lab` | `apps/lab/` | `gomuos` |
| `testapp` | `apps/testapp/` | `gomuos` |

## Deploying New Images

```bash
# After GitHub Actions builds + pushes :latest image:
ssh root@46.224.215.213 "kubectl rollout restart deployment/ordrupspizza-backend -n gomuos"
ssh root@46.224.215.213 "kubectl rollout restart deployment/ordrupspizza-frontend -n gomuos"

# Watch status
ssh root@46.224.215.213 "kubectl rollout status deployment/ordrupspizza-backend -n gomuos"

# Check logs
ssh root@46.224.215.213 "kubectl logs -l app=ordrupspizza-backend -n gomuos --tail=50"
ssh root@46.224.215.213 "kubectl logs -l app=ordrupspizza-frontend -n gomuos --tail=50"
```

## Spring Backend Config Gotchas

- **`APP_DOMAIN`** must be `ordrupspizza` — NOT `ordrupspizza.dk`. `CorsFilter` and `WebSocketConfig` append `.dk` themselves. Setting `.dk` causes double-suffix in allowed origins.
- **Spring profiles**: `prod,ordrupspizza` — restaurant config in `application-ordrupspizza.properties`
- **Customer auth**: in-memory Spring Security user (not in DB), created at boot from `CUSTOMER_USERNAME`/`CUSTOMER_PASSWORD` env vars
- **Firefox 400 errors**: Firefox sends `DNT` and `TE` headers — must be in `StrictHttpFirewall` allowed list in `Config.java`
- **Blocked IPs**: backend auto-blocks suspicious IPs in `blocked_ips` table. Check if a customer reports 403s.

## Frontend Config Gotchas

- **`NEXT_PUBLIC_*` vars are baked at Docker build time** — changing them requires a new Docker build (update `build-args` in `.github/workflows/build.yml`), not just restarting the pod
- **`NEXT_PUBLIC_BACKEND_API`**: `http://ordrupspizza-backend:8080` — ClusterIP, server-side only. Never expose to browser.
- **`pages/payment/callback.tsx`** must stay in Pages Router. Moving it to App Router breaks Bambora redirects.

## Multi-Tenant Architecture

- Shared MySQL (one database per restaurant)
- Each restaurant gets its own: backend deployment, frontend deployment, ingress
- Adding a new restaurant: create `restaurants/<name>/` in infra-gitops mirroring `ordrupspizza/`
- New restaurant needs its own `APP_DOMAIN`, database, and DNS record

## Payment Flow (Bambora)

1. Backend creates payment session → returns Bambora hosted checkout URL
2. Customer pays on Bambora page
3. Bambora POSTs to `https://www.ordrupspizza.dk/payment/callback`
4. Frontend Pages Router (`pages/payment/callback.tsx`) receives it server-side
5. Frontend proxies to backend `BamboraRestController.handlePaymentCallback`
6. Backend marks order `Is_payment_successful = true`

**If orders exist but show as unpaid**: Bambora callback failed. Check `Is_payment_successful` column and frontend/backend logs around callback time.

## WebSocket (Admin Dashboard)

- SockJS + STOMP over `/ws` (ingress routes `/ws` → backend port 8080)
- CSP `connect-src` must include `wss://ordrupspizza.dk` in `next.config.mjs`
- "Connection closed" on first SockJS attempt is normal (falls back to XHR streaming)
- `STOMP CONNECTED` in console = working

## MySQL

| Database | Used by |
|----------|---------|
| `ordrupspizza` | Production |
| `testapp` | Staging |

```bash
# Shell
ssh root@46.224.215.213 "kubectl exec -it mysql-0 -n gomuos -- mysql -u admin -p"

# Recent orders (backtick workaround for reserved word `order`)
ssh -T root@46.224.215.213 'printf "SELECT Id, Ordered_date, Is_payment_successful FROM \x60order\x60 ORDER BY Id DESC LIMIT 10;\n" > /tmp/q.sql && kubectl cp /tmp/q.sql gomuos/mysql-0:/tmp/q.sql && kubectl exec -n gomuos statefulset/mysql -- bash -c "mysql -u admin -p'\''<password>'\'' ordrupspizza < /tmp/q.sql"'
```

**Note**: The `order` table is a SQL reserved word — always use backticks in raw queries.

## Troubleshooting

**Orders not in admin dashboard**
1. Check `Is_payment_successful = 0` → Bambora callback failed
2. Check frontend logs for callback errors
3. Check backend logs: `grep callback`
4. Fix: `UPDATE \`order\` SET Is_payment_successful = 1 WHERE Id = <id>;`

**CORS / 403 errors**
- Check `APP_DOMAIN` env var — must be `ordrupspizza` (no `.dk`)
- Check `blocked_ips` table for auto-blocked client IPs

**WebSocket "Connection closed"**
- First SockJS transport always closes — normal behaviour
- Check CSP `connect-src` includes `wss://ordrupspizza.dk`
- Check ingress has `/ws` path routing to backend port 8080

**Frontend env vars empty in production**
- `NEXT_PUBLIC_*` baked at build time — update `build-args` in workflow, rebuild image, restart pod
