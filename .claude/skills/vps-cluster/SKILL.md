---
name: vps-cluster
description: General infrastructure knowledge for the Hetzner VPS and k3s cluster at 46.224.215.213. Use when working on server-level concerns: adding new apps to the cluster, configuring Traefik/cert-manager/ArgoCD, firewall rules, MySQL host, CoreDNS, or any Kubernetes operation not specific to GomuOS. Triggers on: k3s, Traefik, ArgoCD, cert-manager, Hetzner, firewall, NodePort, cluster, namespace, new app deployment.
---

# VPS / k3s Cluster

General infrastructure for the self-hosted k3s cluster on Hetzner.

## Server

| | |
|--|--|
| Provider | Hetzner Cloud |
| Plan | CX43 (8 vCPU, 16GB RAM, x86) — Falkenstein, Germany |
| IP | `46.224.215.213` |
| OS | Ubuntu 24.04 |
| SSH | `ssh root@46.224.215.213` (key auth) |
| Cost | ~€11/month |

## Kubernetes (k3s)

| Component | Details |
|-----------|---------|
| Distribution | k3s (lightweight) |
| Ingress | Traefik (built-in) |
| Namespaces | `gomuos` (apps), `argocd`, `cert-manager` |
| GitOps | ArgoCD — auto-sync, prune, selfHeal |

## ArgoCD

- **UI**: port 30080 (NodePort, whitelisted in Hetzner firewall for home IP)
- **Apps** live in `infra-gitops/argocd/` — each `.yaml` is an ArgoCD Application
- **Auto-sync**: push to the watched path → ArgoCD applies within ~3 min
- **Force sync**: ArgoCD UI → app → Sync, or `kubectl delete pod` to pull fresh image

**Adding a new ArgoCD app:**
```yaml
# infra-gitops/argocd/myapp.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/shpatjakupi/infra-gitops
    targetRevision: HEAD
    path: apps/myapp
  destination:
    server: https://kubernetes.default.svc
    namespace: gomuos
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```
Bootstrap once: `ssh root@46.224.215.213 "kubectl apply -n argocd -f -" < argocd/myapp.yaml`

## cert-manager

- **Version**: v1.14.4
- **ClusterIssuer**: `letsencrypt-prod` (HTTP-01 challenge via Traefik)
- **Usage**: add `cert-manager.io/cluster-issuer: letsencrypt-prod` annotation to Ingress + `tls:` block
- **DNS must resolve** to `46.224.215.213` before HTTP-01 can complete
- **Check status**:
  ```bash
  ssh root@46.224.215.213 "kubectl get certificate -n gomuos && kubectl get challenges -n gomuos"
  ```

## Traefik (Ingress)

- Built into k3s — no separate install
- Listens on 80 (HTTP) and 443 (HTTPS)
- Use `kubernetes.io/ingress.class: traefik` annotation (or omit — it's the default)
- WebSocket: Traefik proxies WS/WSS transparently — no special annotation needed

## MySQL

- **StatefulSet**: `mysql-0` in `gomuos` namespace
- **Internal**: `mysql.gomuos.svc.cluster.local:3306`
- **External NodePort**: `46.224.215.213:30306` (whitelisted in Hetzner firewall for home IP)
- **Credentials**: stored in `infra-gitops/base/mysql/secret.yaml`
- **Connect from local**: MySQL Workbench → Host `46.224.215.213`, Port `30306`, User `admin`
- **Shell access**:
  ```bash
  ssh root@46.224.215.213 "kubectl exec -it mysql-0 -n gomuos -- mysql -u admin -p"
  ```

## GitHub Container Registry (ghcr.io)

- **Secret**: `ghcr-secret` in `gomuos` namespace (`infra-gitops/base/ghcr-secret.yaml`)
- **Images**: `ghcr.io/shpatjakupi/<repo>:latest`
- **Pull policy**: `Always` on all deployments — ensures fresh image on pod restart
- **Workflow**: push code → GitHub Actions builds + pushes image → `kubectl rollout restart` pulls it

## Firewall

| Layer | Rules |
|-------|-------|
| UFW (host) | Open: 22, 80, 443 for all |
| Hetzner Cloud Firewall | Port 30080 (ArgoCD): home IP only. Port 30306 (MySQL): home IP only |
| Important | UFW cannot block NodePort services — k3s bypasses it via iptables. Use Hetzner firewall for NodePorts |

## CoreDNS

Patched to use `8.8.8.8` upstream (instead of the local resolver which caused internal DNS issues):
```bash
ssh root@46.224.215.213 "kubectl edit configmap coredns -n kube-system"
# forward . 8.8.8.8 1.1.1.1
```

## Apply Manifests Manually (bypass ArgoCD wait)

```bash
scp local-file.yaml root@46.224.215.213:/root/
ssh root@46.224.215.213 "kubectl apply -f /root/local-file.yaml"
```

## Adding a New App — Checklist

1. Create `infra-gitops/apps/<appname>/` with `deployment.yaml`, `service.yaml`, `secret.yaml`, `ingress.yaml`
2. Add TLS to ingress (cert-manager annotation + `tls:` block)
3. Create `infra-gitops/argocd/<appname>.yaml` ArgoCD Application
4. Bootstrap ArgoCD app once via `kubectl apply`
5. Add DNS A record pointing to `46.224.215.213`
6. Wait for cert-manager HTTP-01 challenge to complete (~2-5 min after DNS propagates)
