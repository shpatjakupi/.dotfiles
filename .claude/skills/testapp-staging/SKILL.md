---
name: testapp-staging
description: |
  Full context for the testapp.gomuos.com staging environment — a parallel test instance of the GomuOS food ordering platform running on the same k3s cluster as production. Use this skill when working on, debugging, redeploying, or extending the testapp staging environment. Triggers on "testapp", "staging", "testapp.gomuos.com", deploying or rebuilding the test environment, checking testapp pods/logs/TLS.
---

# testapp-staging

Parallel staging environment for the GomuOS food ordering platform. Runs on the same k3s cluster as production (`ordrupspizza.dk`), namespace `gomuos`, server `46.224.215.213`.

## Key facts

| | Value |
|---|---|
| URL | https://testapp.gomuos.com |
| Namespace | `gomuos` |
| Server | `46.224.215.213` (SSH: `root@46.224.215.213`) |
| ArgoCD app | `testapp` (auto-sync, prune, selfHeal) |
| GitOps repo | `infra-gitops` — path `restaurants/testapp/` |

## Architecture

```
testapp-frontend (port 3000)   <-- ghcr.io/shpatjakupi/next-app-template-testapp:latest
testapp-backend  (port 8080)   <-- ghcr.io/shpatjakupi/order-backend:latest
testapp-ingress                <-- Traefik websecure, cert-manager letsencrypt-prod
```

Ingress routes: `/ws`, `/images`, `/wolt` -> backend; `/` -> frontend.

## Frontend image

`NEXT_PUBLIC_*` vars are **baked at build time** — a separate image is required.

- Image: `ghcr.io/shpatjakupi/next-app-template-testapp:latest`
- Workflow: `next-app-template/.github/workflows/build-testapp.yml` — runs on **every push to `main`** (and `workflow_dispatch`), alongside the prod `build.yml`
- Key build args: `NEXT_PUBLIC_DOMAIN=testapp.gomuos.com`, `NEXT_PUBLIC_WS_URL=https://testapp.gomuos.com/ws`, token key from `TESTAPP_TOKEN_ENCRYPTION_KEY` secret

## Auto-deploy (staging always tracks main)

Since 2026-09-18 both CI workflows end with a restart step, so a push to `main` lands on testapp within minutes with no manual action:

```
push to main → GitHub Actions builds :latest
             → ssh testapp-deploy@46.224.215.213 frontend|backend   (secret TESTAPP_DEPLOY_KEY)
             → /usr/local/bin/testapp-restart                        (forced command, no shell)
             → kubectl rollout restart deployment/testapp-<x>         (kubeconfig bound to SA testapp-deployer)
```

Lock-down, in layers:
- `testapp-deploy` Linux user; its `authorized_keys` entry has `command="/usr/local/bin/testapp-restart"`, `no-pty`, no forwarding — the key can run nothing else
- the script accepts only `frontend` or `backend`
- `/home/testapp-deploy/.kube/config` uses the `testapp-deployer` ServiceAccount; its Role (`infra-gitops/restaurants/testapp/rbac.yaml`) allows get/patch/watch on **only** `testapp-frontend` and `testapp-backend` — `ordrupspizza-*` and secrets are denied
- production stays digest-pinned in infra-gitops and is never touched by CI

Manual trigger of the same path (private key: only in GitHub secrets — regenerate with `ssh-keygen -t ed25519` and update `authorized_keys` + both repos' `TESTAPP_DEPLOY_KEY` if lost):
```bash
ssh -i <deploy-key> testapp-deploy@46.224.215.213 frontend
```

## Backend config

Spring profiles: `SPRING_PROFILES_ACTIVE=prod,ordrupspizza`

Critical env vars (in `testapp-backend-secret`):

| Var | Value |
|---|---|
| `SPRING_DATASOURCE_URL` | `jdbc:mysql://46.224.215.213:30306/testapp?useSSL=false&allowPublicKeyRetrieval=true&serverTimezone=Europe/Copenhagen` |
| `APP_DOMAIN` | `testapp` |
| `APP_WEBSITE_URL` | `https://testapp.gomuos.com` |
| `TEST_MODE` | `true` |
| `BAMBORA_URL` | `https://api.v1.checkout.bambora.com/` (test credentials) |
| `WOLT_BASE_URL` | `https://daas-public-api.development.dev.woltapi.com` (dev) |

`APP_WEBSITE_URL` is injected into Spring's `app.website.url` property via relaxed binding — this is what adds `testapp.gomuos.com` to the CORS and WebSocket allowed origins in `CorsFilter.java` and `WebSocketConfig.java`.

## Infra manifest layout

```
infra-gitops/
  restaurants/testapp/
    backend/  deployment.yaml  secret.yaml  service.yaml
    frontend/ deployment.yaml  secret.yaml  service.yaml
    ingress/  ingress.yaml
  argocd/testapp.yaml
```

ArgoCD app was bootstrapped once with:
```bash
ssh root@46.224.215.213 "kubectl apply -n argocd -f -" <<'EOF'
# (contents of argocd/testapp.yaml)
EOF
```
After that, pushing to `infra-gitops/restaurants/testapp/` auto-syncs within ~3 min.

## Common operations

**Check status:**
```bash
ssh root@46.224.215.213 "kubectl get pods -n gomuos | grep testapp"
ssh root@46.224.215.213 "kubectl get certificate -n gomuos testapp-tls"
```

**View logs:**
```bash
ssh root@46.224.215.213 "kubectl logs -n gomuos deployment/testapp-backend --tail=50"
ssh root@46.224.215.213 "kubectl logs -n gomuos deployment/testapp-frontend --tail=50"
```

**Force redeploy (pull latest image):**
```bash
ssh root@46.224.215.213 "kubectl rollout restart deployment/testapp-backend deployment/testapp-frontend -n gomuos"
```

**Rebuild frontend image** (when build args need changing):
1. Edit `build-testapp.yml` in `next-app-template`
2. Push to `main` (or trigger `workflow_dispatch`) — the pod restarts automatically when the build finishes (~6-8 min)

## TLS

cert-manager `letsencrypt-prod`, HTTP-01 challenge. Secret: `testapp-tls`. If certificate is stuck `Ready: False`, check:
```bash
ssh root@46.224.215.213 "kubectl describe certificate -n gomuos testapp-tls"
ssh root@46.224.215.213 "kubectl get challenges -n gomuos"
```
DNS must resolve `testapp.gomuos.com` to `46.224.215.213` before HTTP-01 can complete.

## Test mode features

- `TEST_MODE=true` on backend — skips real payment processing
- Bambora test credentials (username `T3fn02OyumkhGF0I91GN@T267776101`)
- Wolt dev API (`daas-public-api.development.dev.woltapi.com`)
- `NEXT_TEST_MODE=TRUE` on frontend
- Separate `testapp` MySQL database (same host, port 30306)
