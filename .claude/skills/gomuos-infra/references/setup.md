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
│       │   └── service.yaml
│       └── ingress/
│           └── ingress.yaml    # TLS via cert-manager, routes to frontend
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
- **Status**: Pending - waiting for DNS to point to server

## DNS (Pending Switch)
- **Domain**: ordrupspizza.dk
- **Registrar/DNS**: Simply.com
- **Current A record**: 76.76.21.21 (Vercel - still live)
- **Target A record**: 46.224.215.213 (Hetzner)
- **www**: CNAME → ordrupspizza.dk
- **Action needed**: Change A @ and CNAME www at Simply.com when restaurant closes
- **Email records**: Keep all MX, DKIM, DMARC, SPF untouched (Simply.com email)

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

## Multi-tenant Architecture
- Shared MySQL (one DB per restaurant)
- Each restaurant gets: own backend deployment, frontend deployment, ingress
- Adding a new restaurant: create `restaurants/<name>/` folder with backend/frontend/ingress

---

## AWS (Old Infrastructure - Still Running)

**Still live on AWS (don't delete until migration confirmed stable):**

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
