---
name: kidsapp-infra
description: KidsApp infrastructure knowledge — Unity 6.3 WebGL build pipeline, GameCI, GHCR, ArgoCD, k3s deployment for kidsapp.gomuos.com. Use when building, debugging, or modifying the KidsApp build/deploy chain. For general k3s/Traefik/cert-manager see vps-cluster, for GomuOS-shared platform see gomuos-infra.
---

# KidsApp Infrastructure

Deploy and operations knowledge for the KidsApp Unity project. The flow is: source → GameCI WebGL build → Docker image with nginx → GHCR → ArgoCD → k3s pod serving `kidsapp.gomuos.com`.

## Repos

| Repo | Local | VPS | Purpose |
|------|-------|-----|---------|
| kids-app | `C:\Users\shpat\Desktop\projects\kids-app` | `/home/vegapunk/projects/kids-app` | Unity 6.3 project + WebGL build pipeline |
| infra-gitops | `C:\Users\shpat\Desktop\projects\infra-gitops` | `/home/vegapunk/projects/infra-gitops` | k3s manifests for kidsapp under `apps/kidsapp/` |

## Project Structure

```
kids-app/
├── Assets/                          # Unity project content
├── Packages/                        # Unity packages (incl. embedded mcp-unity)
├── ProjectSettings/
├── docker/
│   ├── Dockerfile                   # nginx + WebGL output
│   └── nginx.conf                   # Cross-Origin headers + WASM/gzip/br MIME
├── .github/workflows/build.yml      # GameCI build pipeline
└── README.md
```

## Build Pipeline (`.github/workflows/build.yml`)

Triggers: push to `main`, manual `workflow_dispatch`.

```
Job 1 — build (Ubuntu, ~10 min)
   actions/checkout
   Free disk space (Unity images are large)
   actions/cache@v4 → Library/ keyed on Assets+Packages+ProjectSettings hash
   game-ci/unity-builder@v4 (targetPlatform=WebGL, unityVersion=6000.3.0f1)
     env: UNITY_LICENSE, UNITY_EMAIL, UNITY_PASSWORD
   actions/upload-artifact → webgl-build

Job 2 — docker (depends on build)
   actions/download-artifact → build/WebGL/
   docker/login-action → GHCR
   docker/build-push-action → ghcr.io/shpatjakupi/kids-app:latest + :sha
```

**Required secrets** (repo `shpatjakupi/kids-app`):
- `UNITY_LICENSE` — content of `C:\ProgramData\Unity\Unity_lic.ulf`
- `UNITY_EMAIL` — Unity account email
- `UNITY_PASSWORD` — Unity account password

License is generated locally by activating Personal license via Unity Hub → Preferences → Licenses → "Get a free personal license". Do NOT use the deprecated `unity-request-activation-file` action.

## Docker / nginx

`docker/Dockerfile`:
```dockerfile
FROM nginx:alpine
COPY docker/nginx.conf /etc/nginx/conf.d/default.conf
COPY build/WebGL/kids-app /usr/share/nginx/html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

`docker/nginx.conf` highlights:
- `Cross-Origin-Opener-Policy: same-origin` + `Cross-Origin-Embedder-Policy: require-corp` (required by Unity 6 WebGL for SharedArrayBuffer)
- Content-Encoding routing for `.br` and `.gz` Unity build artifacts
- `application/wasm` MIME for `.wasm` files
- Aggressive cache for static assets (1y immutable)

## k3s Deployment (`infra-gitops/apps/kidsapp/`)

```
deployment.yaml   # 1 replica, image ghcr.io/shpatjakupi/kids-app:latest, imagePullPolicy: Always
service.yaml      # ClusterIP on port 80
ingress.yaml      # Traefik websecure + cert-manager letsencrypt-prod, host kidsapp.gomuos.com
```

ArgoCD app at `infra-gitops/argocd/kidsapp.yaml` watches `apps/kidsapp/`. Auto-sync with prune + selfHeal.

**Critical:** keep `replicas: 1` in deployment.yaml. The deployment was once accidentally set to `0`, which made ArgoCD scale the live pod down repeatedly.

## Common Operations

### Trigger a fresh build manually

```bash
cd /home/vegapunk/projects/kids-app
gh workflow run build.yml
gh run watch --exit-status
```

### Force k3s to pull latest image

ArgoCD's auto-sync detects manifest changes, but image tag `:latest` doesn't change. To force a pull:
```bash
ssh root@46.224.215.213 "kubectl rollout restart deployment/kidsapp -n gomuos"
ssh root@46.224.215.213 "kubectl rollout status deployment/kidsapp -n gomuos"
```

### Check live status

```bash
ssh root@46.224.215.213 "kubectl get pods -n gomuos -l app=kidsapp; \
  kubectl get ingress kidsapp-ingress -n gomuos; \
  kubectl get certificate kidsapp-tls -n gomuos"

curl -sI https://kidsapp.gomuos.com | head -5
```

### Read pod logs

```bash
ssh root@46.224.215.213 "kubectl logs -l app=kidsapp -n gomuos --tail=80"
```

## Troubleshooting

**Build fails: "License activation strategy could not be determined"**
- All three secrets must be present: `UNITY_LICENSE`, `UNITY_EMAIL`, `UNITY_PASSWORD`
- License must be Personal (free), not Pro — Pro uses `UNITY_SERIAL` instead
- License file is at `C:\ProgramData\Unity\Unity_lic.ulf` (Windows host)

**Build fails: "Cannot build untitled scene"**
- `ProjectSettings/EditorBuildSettings.asset` `m_Scenes` is empty or all entries are `enabled: 0`
- At least one scene must be present and enabled

**Pod is in `Completed` state instead of `Running`**
- nginx exited — usually means the WebGL build files aren't where the Dockerfile expects them
- Check `gh run view <id> --log` to confirm `build/WebGL/kids-app/` exists in the artifact

**ArgoCD shows Synced but `replicas: 0`**
- Some prior commit set `replicas: 0` in `apps/kidsapp/deployment.yaml`. Edit the file, push, ArgoCD will sync.

**`https://kidsapp.gomuos.com` returns 404 after a working deploy**
- Ingress and pod are alive but nginx is serving the wrong path. Check the Dockerfile's `COPY build/WebGL/kids-app /usr/share/nginx/html` matches the actual artifact path. The `kids-app` segment comes from `buildName: kids-app` in the GameCI step.

## Related Skills

- `vps-cluster` — general k3s, Traefik, cert-manager, ArgoCD on this VPS
- `gomuos-infra` — shared GomuOS platform deployment knowledge

## Platform

| | |
|--|--|
| VPS | `46.224.215.213` |
| Live URL | `https://kidsapp.gomuos.com` |
| GitHub repo | `https://github.com/shpatjakupi/kids-app` |
| GHCR image | `ghcr.io/shpatjakupi/kids-app:latest` |
| ArgoCD app | `kidsapp` (namespace `argocd`, watches `apps/kidsapp/`) |
| k3s namespace | `gomuos` |
| Unity version | `6000.3.0f1` (Unity 6.3 LTS) |
