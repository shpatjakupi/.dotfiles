---
name: kidsapp-infra
description: KidsApp infrastructure knowledge — Unity 6.3 build pipelines for the WebGL dev preview at kidsapp.gomuos.com (k3s/ArgoCD) AND mobile production releases (Android Play, iOS App Store via fastlane match). Use when building, debugging, or modifying any of the three build/release chains. For general Unity project conventions see kidsapp-team. For unrelated GomuOS infra see gomuos-infra.
---

# KidsApp Infrastructure

Build and release knowledge for the KidsApp Unity project. There are **three
parallel pipelines** off the same Unity source:

1. **WebGL dev preview** (every push, ~13 min): GameCI WebGL → docker → GHCR →
   ArgoCD → k3s pod at `https://kidsapp.gomuos.com`. This is the daily test
   loop — open it on a phone to touch-test new mini-games.
2. **Android production**: GameCI Android → signed `.aab` → `fastlane supply`
   → Google Play Console (Internal → Production).
3. **iOS production**: GameCI iOS → `fastlane match` + `gym` → signed `.ipa`
   → `pilot` → App Store Connect / TestFlight → App Store review.

The WebGL pipeline runs automatically on every push to `main`. The two mobile
pipelines run on `workflow_dispatch` (manual) so we don't burn CI minutes
and store version codes on every commit.

## Repos

| Repo | Local | VPS | Purpose |
|------|-------|-----|---------|
| kids-app | `C:\Users\shpat\Desktop\projects\kids-app` | `/home/vegapunk/projects/kids-app` | Unity 6.3 project + 3 build pipelines |
| infra-gitops | `C:\Users\shpat\Desktop\projects\infra-gitops` | `/home/vegapunk/projects/infra-gitops` | k3s manifests for the WebGL preview pod under `apps/kidsapp/` |

## Project Structure (infra-relevant parts only)

```
kids-app/
├── Assets/                                 # Unity project content
├── Packages/                               # Unity packages (incl. embedded mcp-unity)
├── ProjectSettings/                        # incl. ProjectSettings.asset (bundleVersion, bundleIdentifier)
├── docker/
│   ├── Dockerfile                          # nginx + WebGL build artifact (preview only)
│   └── nginx.conf                          # COOP/COEP for SharedArrayBuffer + br/gz/wasm MIME
├── fastlane/
│   ├── Fastfile                            # ios + android lanes (match, gym, pilot, supply)
│   ├── Appfile                             # bundle id, Apple ID, team id, package name
│   └── Matchfile                           # match storage repo + branch
└── .github/workflows/
    ├── webgl.yml                           # GameCI WebGL → GHCR → ArgoCD (auto on push)
    ├── android.yml                         # GameCI Android build → Play upload (manual)
    └── ios.yml                             # GameCI iOS build → TestFlight (manual)
```

## Build Pipelines

`webgl.yml` triggers on push to `main`. `android.yml` and `ios.yml` are
manual (`workflow_dispatch` only).

### `webgl.yml` — dev preview (~13 min total)

```
Job 1 — build (Ubuntu, ~10 min)
   actions/checkout
   Free disk space (Unity images are large)
   actions/cache@v4 → Library/ keyed on Assets+Packages+ProjectSettings hash
   game-ci/unity-builder@v4 (targetPlatform=WebGL, unityVersion=6000.3.0f1)
     env: UNITY_LICENSE, UNITY_EMAIL, UNITY_PASSWORD
   actions/upload-artifact → webgl-build

Job 2 — docker (depends on build, ~3 min)
   actions/download-artifact → build/WebGL/
   docker/login-action → GHCR
   docker/build-push-action → ghcr.io/shpatjakupi/kids-app:latest + :sha
```

Then ArgoCD picks up the new image (image tag `:latest` doesn't change, so
the trigger is `kubectl rollout restart` — see Common Operations).

### `android.yml` — production release (~12 min, manual)

```
actions/checkout
Free disk space (Unity images are large)
actions/cache@v4 → Library/ keyed on Assets+Packages+ProjectSettings hash
game-ci/unity-builder@v4
  targetPlatform: Android
  unityVersion:   6000.3.0f1
  androidAppBundle: true
  androidKeystoreName: kids-app.keystore
  androidKeystoreBase64: ${{ secrets.ANDROID_KEYSTORE_BASE64 }}
  androidKeystorePass:   ${{ secrets.ANDROID_KEYSTORE_PASS }}
  androidKeyaliasName:   ${{ secrets.ANDROID_KEYALIAS_NAME }}
  androidKeyaliasPass:   ${{ secrets.ANDROID_KEYALIAS_PASS }}
  env: UNITY_LICENSE, UNITY_EMAIL, UNITY_PASSWORD
→ build/Android/kids-app.aab (signed)

fastlane supply
  package_name: com.shpatjakupi.kidsapp
  aab: build/Android/kids-app.aab
  track: internal
  json_key_data: ${{ secrets.PLAY_STORE_JSON_KEY }}
→ uploaded to Google Play Console "Internal testing" track
```

### `ios.yml` — production release (~18 min on `macos-14`, manual)

```
actions/checkout
actions/cache@v4 → Library/
game-ci/unity-builder@v4
  targetPlatform: iOS
  unityVersion:   6000.3.0f1
  env: UNITY_LICENSE, UNITY_EMAIL, UNITY_PASSWORD
→ build/iOS/ (Xcode project)

fastlane ios beta
  match (readonly)            # pulls signing certs + provisioning profiles
    type: appstore
    git_url: ${{ secrets.MATCH_GIT_URL }}
    password: ${{ secrets.MATCH_PASSWORD }}
  gym                         # xcodebuild + sign
    workspace: build/iOS/Unity-iPhone.xcworkspace
    scheme:    Unity-iPhone
    export_method: app-store
    output_directory: build/ipa
  pilot upload                # to App Store Connect / TestFlight
    api_key_path: <decoded from ASC_API_KEY>
→ build available in TestFlight (~10 min Apple processing)
```

## WebGL preview — Docker + k3s

The WebGL preview is hosted as a single nginx container on the same Hetzner
k3s cluster as the rest of the GomuOS apps.

### `docker/Dockerfile`

```dockerfile
FROM nginx:alpine
COPY docker/nginx.conf /etc/nginx/conf.d/default.conf
COPY build/WebGL/kids-app /usr/share/nginx/html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

### `docker/nginx.conf` highlights

- `Cross-Origin-Opener-Policy: same-origin` + `Cross-Origin-Embedder-Policy:
  require-corp` (required by Unity 6 WebGL for SharedArrayBuffer)
- Content-Encoding routing for `.br` and `.gz` Unity build artifacts
- `application/wasm` MIME for `.wasm` files
- Aggressive cache for static assets (1y immutable)

### k3s deployment (`infra-gitops/apps/kidsapp/`)

```
deployment.yaml   # 1 replica, image ghcr.io/shpatjakupi/kids-app:latest, imagePullPolicy: Always
service.yaml      # ClusterIP on port 80
ingress.yaml      # Traefik websecure + cert-manager letsencrypt-prod, host kidsapp.gomuos.com
```

ArgoCD app at `infra-gitops/argocd/kidsapp.yaml` watches `apps/kidsapp/`.
Auto-sync with prune + selfHeal.

**Critical:** keep `replicas: 1` in `deployment.yaml`. The deployment was
once accidentally set to `0`, which made ArgoCD scale the live pod down
repeatedly.

Because the image tag is `:latest` and never changes, ArgoCD won't notice
a new push by itself. The pipeline relies on `kubectl rollout restart`
(see Common Operations) to force a new image pull.

## Required GitHub Secrets

Stored on `shpatjakupi/kids-app`:

**Unity (both pipelines):**
- `UNITY_LICENSE` — content of `C:\ProgramData\Unity\Unity_lic.ulf`
- `UNITY_EMAIL`
- `UNITY_PASSWORD`

License is Personal (free); generated locally via Unity Hub → Preferences →
Licenses → "Get a free personal license". Do NOT use deprecated
`unity-request-activation-file`.

**Android signing + Play upload:**
- `ANDROID_KEYSTORE_BASE64` — `base64 -w0 kids-app.keystore`
- `ANDROID_KEYSTORE_PASS`
- `ANDROID_KEYALIAS_NAME`
- `ANDROID_KEYALIAS_PASS`
- `PLAY_STORE_JSON_KEY` — service account JSON from Google Play Console with
  "Release manager" permission on the app

**iOS signing + TestFlight upload:**
- `MATCH_GIT_URL` — private repo URL holding encrypted certs/profiles
- `MATCH_PASSWORD` — decrypts the match repo
- `ASC_API_KEY` — App Store Connect API key (JSON: `key_id`, `issuer_id`,
  `key` PEM contents). Used by `pilot` for TestFlight upload — replaces the
  old Apple-ID + app-specific-password flow.

## Versioning

`ProjectSettings/ProjectSettings.asset`:
- `bundleVersion` (string, e.g. `1.0.3`) — the public version
- `AndroidBundleVersionCode` (int, monotonically increasing) — Play
- `iPhone.buildNumber` (string but treated as int) — App Store

Both stores **reject duplicate version codes / build numbers**. CI bumps the
build number automatically using the GitHub Actions run number; manual local
builds must remember to bump it before pushing.

## Common Operations

### Trigger a fresh build manually

```bash
cd /home/vegapunk/projects/kids-app
gh workflow run webgl.yml      # dev preview — auto on push, but force a rerun
gh workflow run android.yml    # production — manual only
gh workflow run ios.yml        # production — manual only
gh run watch --exit-status
```

### Force the WebGL preview pod to pull `:latest`

ArgoCD auto-syncs manifest changes, but image tag `:latest` doesn't change.
After `webgl.yml` finishes:

```bash
ssh root@46.224.215.213 "kubectl rollout restart deployment/kidsapp -n gomuos"
ssh root@46.224.215.213 "kubectl rollout status deployment/kidsapp -n gomuos"
curl -sI https://kidsapp.gomuos.com | head -5
```

### Check WebGL preview live status

```bash
ssh root@46.224.215.213 "kubectl get pods -n gomuos -l app=kidsapp; \
  kubectl get ingress kidsapp-ingress -n gomuos; \
  kubectl get certificate kidsapp-tls -n gomuos"
```

### Read WebGL preview pod logs

```bash
ssh root@46.224.215.213 "kubectl logs -l app=kidsapp -n gomuos --tail=80"
```

### Promote a build between Play tracks

```bash
# Internal → Closed/Open testing → Production happens in the Play Console UI.
# Programmatic alternative via fastlane:
bundle exec fastlane supply \
  --package_name com.shpatjakupi.kidsapp \
  --track internal \
  --track_promote_to production \
  --rollout 0.1   # 10% staged rollout
```

### Promote a TestFlight build to App Store review

Open App Store Connect → My Apps → KidsApp → Distribution → "+ Version" →
pick the latest TestFlight build → submit. Review takes 1–3 days.

### Check what's live on each store

```bash
# Latest CI runs
gh run list --workflow=android.yml --limit 3
gh run list --workflow=ios.yml --limit 3

# What's in Play Internal Testing
bundle exec fastlane supply \
  --package_name com.shpatjakupi.kidsapp \
  --track internal \
  --json_key_path play.json \
  --skip_upload_aab true \
  --skip_upload_apk true
```

App Store status is easiest to check in App Store Connect or via
`bundle exec fastlane pilot builds`.

## Troubleshooting

**Build fails: "License activation strategy could not be determined"**
- All three Unity secrets must be present: `UNITY_LICENSE`, `UNITY_EMAIL`, `UNITY_PASSWORD`
- License must be Personal (free), not Pro — Pro uses `UNITY_SERIAL` instead
- License file path on Windows host: `C:\ProgramData\Unity\Unity_lic.ulf`

**Build fails: "Cannot build untitled scene"**
- `ProjectSettings/EditorBuildSettings.asset` `m_Scenes` is empty or all entries are `enabled: 0`
- At least one scene must be present and enabled

**WebGL preview pod is in `Completed` state instead of `Running`**
- nginx exited — usually means the WebGL build files aren't where the
  Dockerfile expects them. Check `gh run view <id> --log` to confirm
  `build/WebGL/kids-app/` exists in the artifact.

**WebGL preview: ArgoCD shows Synced but `replicas: 0`**
- Some prior commit set `replicas: 0` in `apps/kidsapp/deployment.yaml`.
  Edit the file, push, ArgoCD will sync.

**WebGL preview: `https://kidsapp.gomuos.com` returns 404 after a working deploy**
- Ingress and pod are alive but nginx is serving the wrong path. Check the
  Dockerfile's `COPY build/WebGL/kids-app /usr/share/nginx/html` matches the
  actual artifact path. The `kids-app` segment comes from `buildName: kids-app`
  in the GameCI step.

**WebGL preview: new push didn't update the live site**
- The `:latest` tag is cached on the node. Run
  `kubectl rollout restart deployment/kidsapp -n gomuos` (see Common Operations).

**Android: "Failed to read key from keystore" or "Keystore was tampered with"**
- Keystore secret was re-encoded with a different password, or the keystore
  itself was regenerated. Restore from the **original** keystore — Play
  Store will permanently reject any APK signed with a new key on the
  same package name unless Play App Signing was enabled and you can
  request a key reset.

**Android: Play upload fails "Version code XX has already been used"**
- `AndroidBundleVersionCode` is a duplicate. Bump it. CI usually does this
  via the GitHub run number; if a manual push went through with a stale
  number, bump in `ProjectSettings.asset` and re-push.

**iOS: `match` fails with "Could not find matching profile"**
- Either the match repo is empty for this app id, or the profile expired.
  Run `bundle exec fastlane match appstore --readonly false` locally on a
  Mac to regenerate, then push the encrypted updates to the match repo.
- Apple distribution certs expire yearly; provisioning profiles expire
  yearly too. Set a calendar reminder.

**iOS: `pilot upload` fails with "Invalid Bundle"**
- Most often the bundle id in `ProjectSettings.asset` doesn't match the App
  Store Connect app record, or `ITSAppUsesNonExemptEncryption` is missing
  from `Info.plist`. Add it as `false` for non-crypto apps to skip export
  compliance prompts on every TestFlight upload.

**iOS: TestFlight build stuck in "Processing"**
- Apple-side. Usually 5–30 min, sometimes hours. If > 24h, contact App
  Store Connect support — almost always a missing required `Info.plist`
  key (e.g. `NSCameraUsageDescription` even when camera isn't used).

**IL2CPP build fails on iOS with "PlatformNotSupportedException"**
- A runtime path called `Expression.Compile`, `DynamicMethod`, or another
  IL-emitting reflection API. These don't work under IL2CPP. Find the
  offending call in the stripped logs and replace with a non-emit
  alternative.

## Related Skills

- `kidsapp-team` — full Unity project conventions, mini-game flow, agent team
- `vps-cluster` — general k3s, Traefik, cert-manager, ArgoCD on this VPS (used by the WebGL preview)
- `gomuos-infra` — shared GomuOS platform deployment knowledge

## Platform

| | |
|--|--|
| VPS | `46.224.215.213` (hosts the WebGL preview pod only) |
| GitHub repo | `https://github.com/shpatjakupi/kids-app` |
| Unity version | `6000.3.0f1` (Unity 6.3 LTS) |
| Dev preview URL | `https://kidsapp.gomuos.com` (WebGL, every push) |
| Dev preview image | `ghcr.io/shpatjakupi/kids-app:latest` |
| Dev preview ArgoCD app | `kidsapp` (namespace `argocd`, watches `apps/kidsapp/`) |
| Dev preview k3s namespace | `gomuos` |
| Android package id | `com.shpatjakupi.kidsapp` |
| iOS bundle id | `com.shpatjakupi.kidsapp` |
| Distribution (Android) | Google Play Console — Internal → Closed → Open → Production |
| Distribution (iOS) | App Store Connect — TestFlight → App Store review |
| Code signing (Android) | Local keystore stored as GH secret (base64) |
| Code signing (iOS) | fastlane match — encrypted private git repo |
