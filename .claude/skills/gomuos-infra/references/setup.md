# GomuOS Infrastructure Setup

## Hetzner VPS
- **Provider**: Hetzner Cloud
- **Server**: CX43 (8 vCPU, 16GB RAM, x86) - Falkenstein, Germany
- **IP**: 46.224.215.213
- **OS**: Ubuntu 24.04
- **Monthly cost**: ~€11/month
- **SSH**: `ssh root@46.224.215.213` (key auth)

## Kubernetes (k3s)
- **Distribution**: k3s (lightweight Kubernetes)
- **Ingress controller**: Traefik (built-in)
- **Namespace**: `gomuos` (Gomu Order System)
- **ArgoCD namespace**: `argocd`
- **cert-manager namespace**: `cert-manager`

## ArgoCD (GitOps)
- **Access**: ArgoCD UI via SSH tunnel or port 30080 (whitelisted in Hetzner firewall)
- **NodePort**: 30080
- **Apps**:
  - `gomuos-base` → watches `base/` folder (MySQL, namespace, ghcr-secret, cert-manager)
  - `ordrupspizza` → watches `restaurants/ordrupspizza/` folder
- **Sync**: Automated with prune + selfHeal

## infra-gitops Repo Structure
```
infra-gitops/
├── base/
│   ├── namespace.yaml          # gomuos namespace
│   ├── ghcr-secret.yaml        # GitHub Container Registry auth
│   ├── mysql/
│   │   ├── statefulset.yaml
│   │   ├── service.yaml        # headless ClusterIP
│   │   ├── service-external.yaml  # NodePort 30306 for Workbench
│   │   ├── pvc.yaml
│   │   └── secret.yaml
│   └── cert-manager/
│       └── cluster-issuer.yaml # Let's Encrypt letsencrypt-prod
├── restaurants/
│   └── ordrupspizza/
│       ├── backend/
│       │   ├── deployment.yaml
│       │   ├── service.yaml
│       │   └── secret.yaml     # all env vars + SPRING_PROFILES_ACTIVE=prod,ordrupspizza
│       ├── frontend/
│       │   ├── deployment.yaml
│       │   ├── service.yaml
│       │   └── secret.yaml     # TOKEN_ENCRYPTION_KEY for frontend
│       └── ingress/
│           └── ingress.yaml    # TLS via cert-manager, routes / to frontend and /ws to backend
└── argocd/
    ├── base.yaml               # gomuos-base ArgoCD app
    └── ordrupspizza.yaml       # ordrupspizza ArgoCD app
```

## MySQL
- **Internal service**: `mysql.gomuos.svc.cluster.local:3306` (headless)
- **External NodePort**: `46.224.215.213:30306` (whitelisted in Hetzner firewall)
- **Root password**: stored in `base/mysql/secret.yaml` in infra-gitops
- **App user**: `admin` / (same password as root, stored in secret.yaml)
- **Databases**: `ordrupspizza`, `testapp`
- **Workbench connection**: Host `46.224.215.213`, Port `30306`, User `admin`, Password from secret.yaml

## Docker Images (GitHub Container Registry)
- **Registry**: ghcr.io
- **Backend**: `ghcr.io/shpatjakupi/order-backend:latest`
- **Frontend**: `ghcr.io/shpatjakupi/next-app-template:latest`
- **imagePullPolicy**: Always (set on both deployments)
- **GitHub PAT**: stored in `base/ghcr-secret.yaml` in infra-gitops

## SSL / TLS
- **cert-manager** v1.14.4 installed
- **ClusterIssuer**: `letsencrypt-prod` (HTTP-01 challenge via Traefik)
- **Certificate**: `ordrupspizza-tls` in `gomuos` namespace
- **Status**: Active and working (DNS switched 2026-03-12)

## DNS (Completed 2026-03-12)
- **Domain**: ordrupspizza.dk
- **Registrar/DNS**: Simply.com
- **A record**: 46.224.215.213 (Hetzner) — switched from Vercel 2026-03-12
- **www**: CNAME → ordrupspizza.dk
- **Email records**: MX, DKIM, DMARC, SPF untouched (Simply.com email)

## Firewall
- **Host firewall**: UFW (open: 22, 80, 443 for all)
- **External firewall**: Hetzner Cloud Firewall
  - Port 30080 (ArgoCD): whitelisted for user's home IP
  - Port 30306 (MySQL): whitelisted for user's home IP
- **Note**: UFW cannot block NodePort services (k3s bypasses it via iptables). Use Hetzner firewall instead.

## Spring Backend Config
- **Profile**: `prod,ordrupspizza`
- **Properties file**: `application-ordrupspizza.properties` (bundled in jar)
- **Email**: Simply.com SMTP (`smtp.simply.com:587`, `ordrupspizza@gomuos.com`)
- **Secret name**: `ordrupspizza-backend-secret`
- **Customer service account**: In-memory Spring Security user (NOT in database), created at boot from `CUSTOMER_USERNAME` / `CUSTOMER_PASSWORD` env vars with role `CUSTOMER`
- **`APP_DOMAIN` env var**: Must be `ordrupspizza` (without `.dk`). CorsFilter and WebSocketConfig append `.dk` themselves. Setting it to `ordrupspizza.dk` causes double `.dk` in allowed origins.

## Ingress Routing
- `/ws` → backend (port 8080) — WebSocket/SockJS endpoint
- `/` → frontend (port 3000) — everything else
- Both `ordrupspizza.dk` and `www.ordrupspizza.dk` have the same rules

## Frontend Environment Variables
- **`NEXT_PUBLIC_*` vars are baked at build time** (Next.js inlines them). They must be passed as Docker build args in `.github/workflows/build.yml`, NOT just as runtime env vars in the K8s deployment.
- **`NEXT_PUBLIC_DOMAIN`**: Set to `ordrupspizza.dk` (with `.dk`) in the build workflow. The `next.config.mjs` uses this directly (does NOT append `.dk`).
- **`NEXT_PUBLIC_BACKEND_API`**: Set to `http://ordrupspizza-backend:8080` in K8s deployment. Used by server-side API routes to proxy to backend. Works at runtime since API routes run server-side.
- **Frontend secret**: `ordrupspizza-frontend-secret` in `restaurants/ordrupspizza/frontend/secret.yaml` — holds `TOKEN_ENCRYPTION_KEY`

## Payment Flow (Bambora)
- Backend creates payment session with `BAMBORA_CALLBACK_URL` (from secret)
- Customer completes payment on Bambora hosted page
- Bambora calls `https://www.ordrupspizza.dk/payment/callback` with transaction params
- Frontend Pages Router handler (`pages/payment/callback.tsx`) receives it server-side
- Frontend proxies to backend at `${NEXT_PUBLIC_BACKEND_API}/bambora/callback`
- Backend (`BamboraRestController.handlePaymentCallback`) marks order as `isPaymentSuccessful = true`
- **Critical**: If the callback URL path is wrong, orders are created but never marked as paid, so they won't show in the admin dashboard

## WebSocket (Admin Dashboard)
- SockJS/STOMP protocol over `/ws` endpoint
- Ingress routes `/ws` to backend directly
- SockJS tries native WebSocket first (`wss://`), falls back to XHR streaming
- CSP `connect-src` must include `wss://${domain}` for native WebSocket transport
- Backend `WebSocketConfig` allowed origins use `app.domain` + `.dk` suffix

## Security (StrictHttpFirewall)
- `Config.java` in backend controls allowed headers via deny-list + allow-list
- Firefox sends `DNT` and `TE` headers — these must be in the allowed list or Firefox gets 400 errors
- K8s internal IPs (`10.*`, `172.*`, `192.168.*`, `127.0.0.1`) are whitelisted in `CorsFilter` to skip IP blocking and suspicious request checks

## MySQL Timezone
- MySQL on Hetzner runs in **UTC** (SYSTEM timezone)
- Backend stores `Ordered_date` in **Danish time** (Europe/Copenhagen)
- The `getOrdersFromToday()` query uses `LocalDate.now(ZoneId.of("Europe/Copenhagen"))` — this is correct
- The `order` table is a reserved word in SQL — requires backticks when querying directly

## Multi-tenant Architecture
- Shared MySQL (one DB per restaurant)
- Each restaurant gets: own backend deployment, frontend deployment, ingress
- Adding a new restaurant: create `restaurants/<name>/` folder with backend/frontend/ingress

---

## AWS (Old Infrastructure - Pending Decommission)

**DNS switched to Hetzner on 2026-03-12. AWS still running for safety — decommission after confirmed stable:**

| Service | Details | Monthly Cost |
|---------|---------|-------------|
| RDS MySQL | `database-orderapp.cy9tfut8dd6j.eu-north-1.rds.amazonaws.com:3306` | ~$14 |
| ECS Fargate | Backend + Frontend containers | ~$10 |
| Load Balancer | ALB | ~$18 |
| NAT Gateway | VPC networking | ~$15 |
| Other | Route53, ECR, etc. | ~$16 |
| **Total** | | **~$73/month** |

**AWS RDS credentials:**
- Host: `database-orderapp.cy9tfut8dd6j.eu-north-1.rds.amazonaws.com`
- Port: `3306`
- User: `admin`
- Password: same as Hetzner MySQL (stored in infra-gitops secrets)
- Databases: `ordrupspizza`, `testapp`

**AWS S3 (still in use for image storage):**
- Bucket: `ordrupspizza-images`
- Region: `eu-north-1`
- Access Key: stored in `restaurants/ordrupspizza/backend/secret.yaml` in infra-gitops

**Decommission plan (after 1 week stable on Hetzner):**
1. Delete ECS services
2. Delete RDS instance (after final DB backup)
3. Delete Load Balancer
4. Delete NAT Gateway / VPC
5. Keep S3 bucket (still used for images)
